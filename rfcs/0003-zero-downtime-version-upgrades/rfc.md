start_date: 2026-03-29
mlflow_issue: https://github.com/mlflow/mlflow/issues/21053
rfc_pr:

# Summary

Eliminate mandatory downtime for MLflow tracking server upgrades by treating database migrations the same way well-versioned APIs treat their request schemas: additive changes are backward-compatible by construction, breaking changes are split across releases. This RFC introduces a two-state migration classifier (SAFE / BREAKING) backed by AST analysis, a CI check that prevents PRs from shipping a migration alongside server-side code that contradicts its safety class, an operator CLI command that reports the upgrade path's classification, and an opt-in server flag that allows the tracking server to start against an older database when all pending migrations are SAFE.

The classifier is mechanical end to end: every Alembic operation MLflow uses today lands deterministically in either SAFE or BREAKING based on the operation and its arguments. The single exception is `op.execute()` raw SQL, which the AST cannot inspect; that operation defaults to BREAKING and may be promoted to SAFE only via an explicit, machine-validated annotation in the migration file. Operators never have to make a per-migration safety judgment, because the judgment is made at PR time by the tooling that has the most context.

# Basic example

**Operator checks an upgrade path before deploying:**

```bash
$ mlflow db check-upgrade "postgresql://mlflow:mlflow@db:5432/mlflow"
Current revision: 867495a8f9d4
Target revision:  76601a5f987d
Pending migrations: 3

  [+] a1b2c3d4e5f6 (SAFE)
      create_table: table=scorers
  [+] f6e5d4c3b2a1 (SAFE)
      create_index: index=ix_scorers_name
  [+] 76601a5f987d (SAFE)
      add_column: table=experiments column=archived nullable=True

All 3 pending migrations are SAFE. Zero-downtime upgrade is possible.
```

**Rolling deployment with safe migrations (zero downtime):**

1. `mlflow db check-upgrade <db-url>` returns exit code 0
2. Deploy new MLflow server instances via rolling update; the new servers are started with `--allow-zero-downtime-upgrades`
3. New servers start successfully against the old schema (Tier 3 auto-compat)
4. Run `mlflow db upgrade <db-url>` as a post-deploy job
5. No downtime occurred

**Operator sees a breaking migration:**

```bash
$ mlflow db check-upgrade "postgresql://mlflow:mlflow@db:5432/mlflow"
Current revision: 867495a8f9d4
Target revision:  c8d9e0f1a2b3
Pending migrations: 2

  [+] a1b2c3d4e5f6 (SAFE)
      create_table: table=scorers
  [!] c8d9e0f1a2b3 (BREAKING)
      add_column: table=experiments column=workspace nullable=False
      reason: non-nullable column add; new code writes the column on every INSERT

1 of 2 pending migrations is BREAKING. Use the traditional stop-the-world workflow.
```

**Emergency bypass (production incident, operator overrides the classifier):**

```bash
export MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS=true
mlflow server --backend-store-uri "postgresql://..."
# Server starts. Logs ERROR at startup listing every skipped migration,
# and ERROR on every incoming request, until the env var is unset.
```

## Motivation

MLflow's `_verify_schema()` function performs a strict equality check between the database's current Alembic revision and the revision the running code expects. If there is any mismatch, the server raises `MlflowException` and refuses to start. This design forces a stop-the-world deployment workflow on every release that includes a schema migration:

1. Scale down all tracking server instances
2. Run `mlflow db upgrade`
3. Scale back up

For teams running MLflow as a shared tracking server in production (e.g., on Kubernetes), this means scheduled downtime on every release. The problem is that the vast majority of MLflow migrations are already structurally additive: creating new tables, adding nullable columns, or building indexes. Of the migrations currently shipped under `mlflow/store/db_migrations/versions/`, more than 90% are pure `add_*` / `create_*` / `increase_*` operations. These are fully compatible with the previous release's server code by construction — and yet the strict schema check refuses to start against them, forcing downtime that the migrations themselves don't require.

The framing this RFC adopts is that **a database schema is an API contract between the migration (the producer) and the server code (the consumer)**, and the rules for evolving it without breaking clients are well-understood. They are the same rules every mature REST or gRPC service follows: new fields must be optional, removed fields go through a deprecation window, and renames are two operations across multiple releases. When these rules are followed, additive changes are backward-compatible *by construction* and no runtime safety check needs to second-guess them. When they are violated, the violation is caught in CI on the PR that introduces it — not at deploy time by an operator who lacks the context to make the call.

This RFC proposes the tooling to make that contract enforceable: a deterministic classifier for the schema side, a PR-diff check for the code side, an operator CLI for verification, and an opt-in server-side flag for the rolling-deploy path.

**Who this affects:**

- Any team running MLflow tracking server as a shared service in production
- Platform teams managing MLflow on Kubernetes or similar orchestrators
- Organizations with SLAs that prohibit scheduled downtime for routine upgrades

### Out of scope

- **Online (live) migration execution**: This proposal does not add a system to run migrations while the server is handling traffic. Migrations are still run explicitly via `mlflow db upgrade`.
- **Multi-version server clusters**: This proposal does not support running servers at two different code versions simultaneously for extended periods. The expectation is that rolling deployments complete within a reasonable window.
- **Backward migrations (downgrades)**: The classifier only analyzes the upgrade path. Rollback safety is not addressed.
- **Non-SQL backend stores**: This proposal only applies to the SQLAlchemy-backed tracking store.
- **Long-running migration impact** (lock contention, vacuum pressure, replication lag): The classifier reasons about *whether the new server can boot against the old schema*, not about how long the migration itself takes to apply or what operational pressure it creates while running. Online-DDL concerns deserve their own treatment in a separate RFC.
- **Per-feature migration granularity**: An operator-side ability to skip migrations belonging to features they don't use. See the Alternatives section for why this is out of scope.

## Detailed design

The implementation has five components: a migration safety classifier, a tiered schema verification function, a CI-side PR-diff check, a CLI command for operators, and a developer tool for local use.

### 1. Migration safety classifier

A new module `mlflow/store/db_migrations/migration_classifier.py` provides AST-based static analysis of Alembic migration scripts. It parses the `upgrade()` function of each migration and classifies every Alembic operation call into one of two safety levels. The overall safety of a migration is the worst classification among all its operations: if any operation is BREAKING, the migration is BREAKING.

| Operation | Classification | Notes |
|---|---|---|
| `create_table` | **SAFE** | New table; old code unaware, new code uses it freely. Bidirectional. |
| `create_index` | **SAFE** | Pure query-planner concern. |
| `drop_index` | **SAFE** | Pure query-planner concern; same family as `create_index`. |
| `drop_constraint` | **SAFE** | Strictly more permissive: anything legal before is still legal. |
| `add_column` (`nullable=True`, no `server_default` or default produces NULL) | **SAFE** | Genuinely additive in both directions. The code-side correctness is enforced separately by the PR-diff CI check (component 3). |
| `alter_column (nullable=True)` | **SAFE** | Relaxing a constraint. |
| `alter_column (server_default change)` | **SAFE** | Defaults are evaluated at write time and are invisible to old data. |
| `alter_column (in-family type widening)` | **SAFE** | Detected by reading `existing_type` and `type_` from the Alembic call. `VARCHAR(64) → VARCHAR(255)` and `INT → BIGINT` are typical examples. |
| `add_column (nullable=False)` | **BREAKING** | Required new field. By definition the new server expects a value the old DB cannot produce. This is the failure mode best illustrated by adding a `workspace` column with `server_default='default'` and immediately writing it from every INSERT. |
| `add_column` (`nullable=True` with non-null `server_default`) | **BREAKING** by default | Borderline shape: the non-null default is a smell that the new code expects a value. May be promoted to SAFE via annotation (component 1, "Annotations" below). |
| `alter_column` (type narrowing or cross-family) | **BREAKING** | Existing rows may not fit the new type. |
| `create_foreign_key` | **BREAKING** by default | Adding a constraint can reject inserts the old code path produced. May be promoted to SAFE via annotation if the author confirms application-side enforcement already exists. |
| `drop_table`, `drop_column`, `rename_table`, `rename_column` | **BREAKING** | Destructive. |
| `op.execute(...)` raw SQL | **BREAKING** by default | The AST cannot see inside the string. May be promoted to SAFE via annotation. |
| ORM/data migrations (`session.query`, `op.get_bind`) | **BREAKING** | Inherently unsafe for online operation. |

There is no third "cautious" or "unknown" tier. Every operation MLflow uses lands deterministically in SAFE or BREAKING based on its arguments. The classifier never returns "I don't know" — when it cannot prove SAFE, it returns BREAKING and the migration falls through to the strict check.

**Annotations.** Three operation shapes default to BREAKING but may be promoted to SAFE by an explicit annotation in the migration file: `op.execute(...)`, `add_column (nullable=True, non-null server_default)`, and `create_foreign_key`. The annotation format is a single comment placed on the line immediately preceding the operation call:

```python
def upgrade():
    # mlflow:safety=safe reason: creates a SQLite trigger that enforces immutability
    op.execute(_SQLITE_SECRETS_IMMUTABILITY_TRIGGER)

    # mlflow:safety=safe reason: experiments table already enforces this FK application-side
    op.create_foreign_key(...)
```

The classifier parses the comment from the AST's source-position information. The annotation must include both `safety=safe` and a `reason:` field with non-empty free-text content. Annotations without a reason, or annotations on operations that are not in the override-eligible set, are rejected by `dev/schema_diff.py` in CI (component 3). The vast majority of migration authors will never write an annotation because the vast majority of migrations don't use the override-eligible operations.

**Why no general "annotate every migration" requirement.** Manual safety annotations on every migration decay into copy-paste boilerplate within months — Rails's `safety_assured!` from `strong_migrations` is the canonical cautionary tale. The classifier deliberately scopes annotations to the small set of operations the AST genuinely cannot decide on its own, so the annotation surface stays small enough for reviewers to take seriously.

**Conservative defaults.** Unknown operations (anything not in the table above) default to BREAKING. Missing `upgrade()` functions default to BREAKING. Classifier exceptions default to BREAKING. The classifier fails closed in every direction.

**Key design decisions:**

- **AST parsing over string matching**: Using Python's `ast` module provides structured analysis of the migration code rather than fragile regex matching. It correctly handles `op.X`, `batch_op.X`, and nested calls.
- **Deterministic two-state output**: The classifier never asks the operator "is this safe?". The decision is binary and made at PR time. Operators only see SAFE/BREAKING in `mlflow db check-upgrade` output, and their workflow is determined by which one applies.
- **ORM detection as BREAKING**: Migrations that use `session.query()` or `op.get_bind()` perform data transformations that depend on model definitions. These are inherently unsafe for online operation.

### 2. Tiered schema verification

The existing `_verify_schema(engine)` function in `mlflow/store/db/utils.py` is replaced with a 4-tier check:

```
Tier 1: Exact match                  -> pass silently (unchanged behavior)
Tier 2: Emergency bypass             -> warn loudly and continue
Tier 3: Auto-compat (opt-in)         -> classify pending migrations; continue if all SAFE
Tier 4: Strict check                 -> raise MlflowException (unchanged behavior)
```

**Tier 1 — Exact match**: If the database revision matches the code's expected revision, the server starts normally. This is identical to current behavior.

**Tier 2 — Emergency bypass**: If `MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS=true` is set, the server starts regardless of the schema state. At startup it logs an ERROR-level message listing every migration the database is behind on (or ahead of). For the duration of the bypass, every incoming request also logs an ERROR-level message tagged with the bypass state, so leaving the flag on creates an unmissable paper trail. This is the "production is on fire, get out of jail free" card. It is intentionally named to feel uncomfortable to type and intentionally loud to discourage use as a routine workflow.

**Tier 3 — Auto-compat**: Activated only when the operator passes `--allow-zero-downtime-upgrades` to `mlflow server` (or sets `MLFLOW_ALLOW_ZERO_DOWNTIME_UPGRADES=true`). Without the flag, Tier 3 does not run and the server falls through directly to Tier 4. With the flag, the server walks the Alembic revision chain and classifies every pending migration. If the database is behind the code AND every pending migration classifies as SAFE, the server logs an INFO-level message ("Schema mismatch detected: N pending migrations, all SAFE; starting normally per --allow-zero-downtime-upgrades") and continues. If any pending migration is BREAKING, or if the classifier raises any exception, the server falls through to Tier 4.

**Tier 4 — Strict check**: The existing behavior. The server raises `MlflowException` with the current error message instructing the operator to run `mlflow db upgrade`.

A new helper `_is_schema_behind(current_rev, head_revision)` walks the Alembic revision chain to determine directionality. This prevents auto-compat from activating when the database is ahead of the code (which would indicate a downgrade scenario).

**Why Tier 3 is opt-in even though SAFE is provably safe.** The classifier and the PR-diff CI check together close the failure modes that would otherwise require operator judgment, so "you have been warned" framing would be misleading. The reason Tier 3 is still opt-in is unrelated to safety: changing operator-visible behavior on upgrade without consent is bad regardless of whether the new behavior is safe. Some operators have wrapper scripts and health checks that depend on `mlflow server` failing fast on schema mismatch; flipping Tier 3 on by default would silently break them. Whether a server participates in zero-downtime upgrades should also be a grep-able fact in startup args, not an implicit behavior derived from a dependency upgrade. The flag preserves the existing default for everyone who didn't ask for the new behavior.

### 3. CI-side PR-diff check

A new check in `dev/schema_diff.py` runs on every pull request that touches `mlflow/store/db_migrations/versions/`. It exists to close the one failure mode the AST classifier cannot see on its own: a structurally additive migration paired with a backward-incompatible code change in the same PR.

The check operates in two passes:

1. **Migration parse**: For each new or modified migration file in the PR, identify every `add_column` operation and extract the `(table_name, column_name)` pair, the nullability, and the `server_default` (if any).
2. **Diff scan**: For each `(table_name, column_name)` pair from pass 1, scan the rest of the PR's diff for ORM model changes that introduce the same column to the corresponding `SqlAlchemy` model in `mlflow/store/tracking/dbmodels/`. If the model addition places the column in a position that makes it required at write time (e.g., it appears in a `Column(..., nullable=False)` declaration without a Python-side default, or it's referenced by a constructor that does not provide a default), the check fails the build.

The high-level rule the check enforces is: **a PR may add a migration, and a PR may add server-side writes to a new column, but not both in the same PR**. If both are present, the migration is structurally additive but the new server is no longer backward-compatible with the old DB — exactly the `workspace` failure mode. Splitting the change across two releases (one to add the column, a later one to start writing it) is the standard expand-contract pattern from API versioning, applied to schemas.

The check also validates annotation correctness: any `# mlflow:safety=safe` comment on an operation that is not in the override-eligible set (`op.execute`, `add_column nullable=True with non-null server_default`, `create_foreign_key`) fails the build, as does any annotation missing the `reason:` field or with an empty reason.

This component is the load-bearing piece of the RFC. The runtime classifier is defense in depth for properties that are already proven at merge time.

### 4. `mlflow db check-upgrade` CLI command

A new subcommand under `mlflow db` that analyzes the upgrade path from the database's current revision to the latest head:

```bash
mlflow db check-upgrade <database-url> [--json]
```

**Human-readable output** lists each pending migration with a safety icon (`+` = SAFE, `!` = BREAKING), its operations, and (for BREAKING migrations) the AST-detected reason it was classified as BREAKING.

**JSON output** (`--json`) produces machine-parseable results for CI pipeline integration.

**Exit codes**: `0` = all pending migrations are SAFE, `1` = at least one pending migration is BREAKING. There is no third exit code, because there is no third classification.

This allows operators to pre-check upgrade safety in CI/CD pipelines before deploying.

### 5. `dev/schema_diff.py` developer tool

The same script that powers the CI check (component 3) also exposes a standalone CLI for local use, providing the same analysis without requiring a database connection:

```bash
# With database connection
python dev/schema_diff.py --db-url "postgresql://..."

# Without database connection (revision range only)
python dev/schema_diff.py --from-revision abc123 --to-revision def456

# JSON output for CI
python dev/schema_diff.py --from-revision abc123 --to-revision def456 --json
```

Same exit codes as `mlflow db check-upgrade`. This is intended both for migration authors validating their work locally and for CI jobs that run the PR-diff check (component 3).

### 6. Environment variables and CLI flags

Two new environment variables are registered in `mlflow/environment_variables.py`:

- `MLFLOW_ALLOW_ZERO_DOWNTIME_UPGRADES` (boolean, default `false`). Equivalent to passing `--allow-zero-downtime-upgrades` to `mlflow server`. Activates Tier 3.
- `MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS` (boolean, default `false`). Activates Tier 2 emergency bypass. Logs ERROR at startup and on every incoming request.

The bypass variable is deliberately named to feel uncomfortable to type and obvious in a config-file review. It replaces the originally proposed `MLFLOW_ALLOW_SCHEMA_MISMATCH`, which was too neutral a name for a genuinely dangerous operation.

### Recommended deployment workflow

```
                     ┌────────────────────────┐
                     │ mlflow db check-upgrade │
                     └──────────┬─────────────┘
                                │
                  ┌─────────────┴─────────────┐
                  │                           │
              exit 0                       exit 1
            (all SAFE)                   (BREAKING)
                  │                           │
                  v                           v
        Rolling update with         Scale down →
        --allow-zero-downtime-      mlflow db upgrade →
        upgrades                    Scale up
        (zero downtime)             (traditional workflow)
                  │
                  v
        mlflow db upgrade
        (post-deploy job)
```

The emergency bypass (`MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS`) is not part of the recommended workflow and does not appear in the diagram. It exists for production incidents where the operator must roll forward against an old DB despite a BREAKING migration, and is loud enough that nobody will leave it engaged by accident.

## The migration contract

This RFC's classifier and CI check encode a project-level expectation about how MLflow database migrations should evolve. That expectation is worth stating explicitly, because it's what makes the SAFE category meaningful:

> **MLflow database migrations across minor versions are structurally additive and backward-compatible with the previous release's server code.** A change that requires a destructive operation — drop column, rename, NOT NULL tightening, type narrowing, data migration — is split across releases using the expand-contract pattern: phase 1 introduces the new shape additively, phase 2 backfills and switches code over, phase 3 removes the old shape. Each phase ships in a separate release. CI prevents a single PR from containing both a migration and a backward-incompatible code change against the prior schema.

This is not a new constraint invented by this RFC. It's a description of how more than 90% of MLflow's existing migrations already behave: the historical migrations under `mlflow/store/db_migrations/versions/` are overwhelmingly `add_*`, `create_*`, and `increase_*` operations. The genuinely destructive ones are countable on one hand. The contract codifies the de facto practice and gives the project the tooling to enforce it on PRs that would otherwise drift away from it.

The contract is presented here as **the project's recommended direction**, not as an enforced project-wide policy that this RFC alone commits the maintainers to. Adopting it strictly is a judgment for the maintainer team to make in a follow-up. The RFC's machinery (classifier, CI check, server flag, CLI command) works regardless of whether the maintainers formally adopt the contract — but the SAFE category is most useful when the contract is followed.

## Drawbacks

- **Maintenance burden of the classifier**: The AST-based classifier must correctly handle the Alembic operation patterns used in MLflow migrations. New operation patterns may require updates to the classifier. The override-eligible set (`op.execute`, the borderline `add_column`, `create_foreign_key`) and the annotation parser must be maintained alongside the operation table.

- **CI-side PR-diff check has false positives**: The diff scan in `dev/schema_diff.py` is a textual heuristic against the SQLAlchemy model files. It will occasionally flag a PR that adds a migration *and* an unrelated model change in the same diff, even when the two are not coupled. The mitigation is a documented escape: the PR author can split the diff into two PRs (which is what the contract recommends anyway), or add an explicit `# mlflow:safety-check skip` annotation on the model change with a justification, validated and recorded in CI.

- **Complexity in the startup path**: The tiered verification adds branching logic to server startup. If the classifier raises an unexpected exception, the fallback to Tier 4 ensures safety, but the added code paths increase the surface area for bugs.

- **Emergency bypass remains a footgun**: `MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS=true` bypasses all safety checks. The naming and the loud per-request logging are intended to make this hard to leave on accidentally, but a determined operator who suppresses the logs can still leave it engaged. The RFC accepts this trade-off because removing the escape hatch entirely would be hostile to operators in real incidents.

# Alternatives

### 1. Keep the original three-tier classification (SAFE / CAUTIOUS / BREAKING)

The original draft of this RFC included a CAUTIOUS tier for migrations whose safety depended on operator judgment (non-nullable column adds, type changes, foreign keys, raw SQL). This was rejected during review because every operation in that bucket can be deterministically classified into SAFE or BREAKING with a slightly richer AST classifier — the middle bucket has no inhabitants. CAUTIOUS as a runtime category also has the worst possible signal-to-noise property: every CAUTIOUS migration the operator sees and ignores trains them to ignore the next one, until one of them genuinely breaks production. Two states give operators an actionable decision every time and no opportunity to develop the "ah, it's probably fine" reflex.

### 2. General "annotate every migration" requirement

An alternative would be to require migration authors to declare a safety class on every migration, with the AST acting only as a suggestion. Rejected: manual safety annotations on every migration decay into copy-paste boilerplate within months, and the empirical evidence (the same review that surfaced this comment cited a migration that was written, reviewed, and merged by humans and still shipped a backward-compatibility bug) is that human review at this frequency is the *less* reliable signal, not the more reliable one. The current proposal scopes annotations narrowly to the three operation shapes the AST genuinely cannot decide, so most migration authors never see the annotation system.

### 3. Per-feature migration granularity

An alternative framing would let operators skip migrations for features their deployment doesn't use (e.g., skip AI Gateway migrations if the org doesn't use AI Gateway). This *appears* feasible because MLflow ships installable extras like `mlflow[gateway]` and `mlflow[genai]`, which look like opt-in feature toggles. In practice the extras only install optional runtime provider dependencies (boto3, tiktoken, slowapi, etc.); the feature code itself, including the database tables and the FastAPI routers, is unconditionally part of every MLflow tracking server. `mlflow/server/fastapi_app.py` mounts the gateway router unconditionally and the migrations live in the core package, so there is no MLflow installation that lacks the gateway tables. Genuine per-feature granularity would require a first-class feature registry, conditional router and ORM mounting, and graceful degradation on disabled features — a backward-incompatible architectural change to the server, not an addition. This approach is therefore out of scope for this RFC, but it composes cleanly with the migration contract and could be added in a future RFC if MLflow grows a feature-flag architecture.

### 4. Schema version tolerance window

Allow the server to start if the database is within N revisions of the expected version, regardless of migration content. Simple but unsafe: a single BREAKING migration within the tolerance window would cause failures. Rejected.

### 5. Maintain a manual safety manifest

Instead of AST parsing, maintain a YAML/JSON file that maps each migration revision to its safety level. More explicit but creates a maintenance burden on every migration and risks going stale. The AST classifier automates this for the vast majority of operations and the narrow annotation system handles the edge cases. Rejected.

### 6. Database-level compatibility views

Create SQL views that present the old schema shape to old code while the new schema evolves underneath. Powerful but extremely complex, database-specific, and far beyond the scope of MLflow's current architecture. Rejected.

### 7. Per-revision allowlist for selective bypass

The original draft of the RFC considered an `MLFLOW_SCHEMA_MISMATCH_ALLOWED_REVISIONS=rev1,rev2,rev3` mechanism that would let an operator selectively approve specific revisions whose safety they had personally verified. This was rejected because, after collapsing CAUTIOUS, there are no judgment-dependent revisions for an allowlist to operate on: every pending revision is either provably SAFE (auto-allowed under Tier 3) or provably BREAKING (only crossable via the emergency bypass). The single bypass mechanism (`MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS`) is sufficient for the genuinely exceptional case.

### 8. Do nothing

Operators continue using the stop-the-world workflow. This is the status quo and works, but it forces unnecessary downtime for the majority of releases that only include additive migrations.

# Adoption strategy

This is a fully backward-compatible change. Existing users experience zero difference unless they opt in:

- **No changes to default behavior**: Tier 1 (exact match) and Tier 4 (strict check) are identical to current behavior. Tier 3 only activates when the operator explicitly passes `--allow-zero-downtime-upgrades`. Tier 2 only activates when the operator explicitly sets `MLFLOW_BYPASS_SCHEMA_CHECK_DANGEROUS=true`.
- **New CLI command**: `mlflow db check-upgrade` is additive and does not affect existing commands.
- **CI check**: The new check in `dev/schema_diff.py` runs only on PRs that touch `mlflow/store/db_migrations/versions/`. It will need a one-time pass over the existing migrations to confirm they all pass (the historical destructive migrations are grandfathered and do not need to retroactively conform).
- **Documentation**: The self-hosting migration guide is updated with a "Zero-Downtime Upgrades" section documenting the new workflow, the contract, the safety levels, and the env vars.

For teams already running MLflow in production, adoption is straightforward:

1. Add `mlflow db check-upgrade` to CI/CD pipeline
2. If exit code is 0, deploy with `--allow-zero-downtime-upgrades` and switch to rolling deployment
3. If exit code is 1, use the traditional stop-the-world workflow

# Open questions

1. **Should the contract be elevated to a hard project policy?** This RFC presents the migration contract as the project's recommended direction and ships the tooling to enforce it on PRs. A follow-up decision the maintainer team can make is whether to adopt the contract as a strict policy that applies to every minor-version migration going forward. The RFC's machinery works either way, but the strongest version of the product story is "minor-version upgrades are zero-downtime by policy", which requires the maintainer commitment.

2. **Annotation syntax stability.** This RFC specifies `# mlflow:safety=safe reason: <text>` as the annotation format. Should this be locked in as part of the RFC, or treated as a placeholder that the implementation PR can refine? The format is parser-friendly and human-readable as written, but reviewers may have preferences.

3. **Should the CI check enforce the contract retroactively on existing migrations?** The current proposal grandfathers them in. An alternative is a one-time audit that classifies every existing migration and flags any that would not pass the check. Useful documentation, but it's a separate effort.

4. **Index creation on large tables**: `create_index` is classified as SAFE, but operationally it can cause lock contention on multi-million-row tables in some databases. This RFC treats lock contention as a separate concern (out of scope; a long-running-migration RFC). Should `mlflow db check-upgrade` at least surface a hint about this for `create_index` operations on tables known to be large? This would require the tool to have some notion of table size, which it currently doesn't.
