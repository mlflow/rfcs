start_date: 2026-03-29
mlflow_issue: https://github.com/mlflow/mlflow/issues/21053
rfc_pr:

# Summary

Add tiered schema verification and an AST-based migration classifier so that MLflow tracking servers can start against an older database schema when all pending migrations are purely additive. This eliminates mandatory downtime for the majority of MLflow releases while preserving the safety guarantees of the existing strict check for destructive migrations.

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
      create_table: table=issues

All pending migrations are SAFE. Zero-downtime upgrade is possible.
```

**Rolling deployment with safe migrations (zero downtime):**

1. `mlflow db check-upgrade <db-url>` returns exit code 0
2. Deploy new MLflow server instances via rolling update
3. New servers start successfully against the old schema (Tier 3 auto-compat)
4. Run `mlflow db upgrade <db-url>` as a post-deploy job
5. No downtime occurred

**Manual override for cautious migrations:**

```bash
export MLFLOW_ALLOW_SCHEMA_MISMATCH=true
mlflow server --backend-store-uri "postgresql://..."
# Server starts with a warning instead of crashing
```

## Motivation

MLflow's `_verify_schema()` function performs a strict equality check between the database's current Alembic revision and the revision the running code expects. If there is any mismatch, the server raises `MlflowException` and refuses to start. This design forces a stop-the-world deployment workflow on every release that includes a schema migration:

1. Scale down all tracking server instances
2. Run `mlflow db upgrade`
3. Scale back up

For teams running MLflow as a shared tracking server in production (e.g., on Kubernetes), this means scheduled downtime on every release. The problem is that the vast majority of MLflow migrations are purely additive: creating new tables, adding nullable columns, or building indexes. These operations are fully compatible with the old code still running. A server at revision N can safely serve traffic while the database is at revision N or N+K, as long as the pending migrations only add things the old code doesn't reference.

There is currently no mechanism to distinguish a safe additive migration from a breaking destructive one, and no escape hatch to bypass the check when an operator knows the upgrade is safe.

**Who this affects:**

- Any team running MLflow tracking server as a shared service in production
- Platform teams managing MLflow on Kubernetes or similar orchestrators
- Organizations with SLAs that prohibit scheduled downtime for routine upgrades

### Out of scope

- **Online (live) migration execution**: This proposal does not add a system to run migrations while the server is handling traffic. Migrations are still run explicitly via `mlflow db upgrade`.
- **Multi-version server clusters**: This proposal does not support running servers at two different code versions simultaneously for extended periods. The expectation is that rolling deployments complete within a reasonable window.
- **Backward migrations (downgrades)**: The classifier only analyzes the upgrade path. Rollback safety is not addressed.
- **Non-SQL backend stores**: This proposal only applies to the SQLAlchemy-backed tracking store.

## Detailed design

The implementation has four components: a migration safety classifier, a tiered schema verification function, a CLI command for operators, and a developer tool for CI.

### 1. Migration safety classifier

A new module `mlflow/store/db_migrations/migration_classifier.py` provides AST-based static analysis of Alembic migration scripts. It parses the `upgrade()` function of each migration and classifies every Alembic operation call into one of three safety levels:

| Safety level | Meaning | Operations |
|---|---|---|
| **SAFE** | Purely additive; old code is unaffected | `create_table`, `create_index`, `add_column` (nullable), `alter_column` (set nullable=True), `alter_column` (server_default change) |
| **CAUTIOUS** | Likely safe but requires human review | `add_column` (non-nullable), `alter_column` (type change), `drop_constraint`, `create_foreign_key`, `drop_index`, `execute` (raw SQL) |
| **BREAKING** | Destructive; old code will fail | `drop_table`, `drop_column`, `rename_table`, `alter_column` (rename), ORM/data migrations (detected via `session.query` / `op.get_bind`) |

The overall safety of a migration is the worst classification among all its operations.

**Manual overrides**: A dictionary maps specific revision hashes to known safety levels for migrations the AST parser cannot accurately classify (e.g., VARCHAR widening, which the parser sees as a type change but is actually safe on all supported backends).

**Key design decisions:**

- **AST parsing over string matching**: Using Python's `ast` module provides structured analysis of the migration code rather than fragile regex matching. It correctly handles `op.X`, `batch_op.X`, and nested calls.
- **Conservative defaults**: Unknown operations default to CAUTIOUS. Missing `upgrade()` functions default to CAUTIOUS. Classifier failures fall back to the existing strict check (Tier 4).
- **ORM detection as BREAKING**: Migrations that use `session.query()` or `op.get_bind()` perform data transformations that depend on model definitions. These are inherently unsafe for online operation.

### 2. Tiered schema verification

The existing `_verify_schema(engine)` function in `mlflow/store/db/utils.py` is replaced with a 4-tier check:

```
Tier 1: Exact match       -> pass silently (unchanged behavior)
Tier 2: Env var override   -> warn and continue
Tier 3: Auto-compat        -> classify pending migrations; continue if all SAFE
Tier 4: Strict check       -> raise MlflowException (unchanged behavior)
```

**Tier 1 - Exact match**: If the database revision matches the code's expected revision, the server starts normally. This is identical to current behavior.

**Tier 2 - Environment variable override**: If `MLFLOW_ALLOW_SCHEMA_MISMATCH=true` is set, the server logs a warning and starts regardless of the mismatch. This is an escape hatch for operators who have manually verified compatibility. It works in both directions (DB ahead or behind).

**Tier 3 - Automatic compatibility**: If the database is behind the code (determined by walking the Alembic revision chain) AND all pending migrations are classified as SAFE, the server logs an informational message and starts. If any migration is CAUTIOUS, a warning is logged and the server falls through to Tier 4. If classification fails for any reason, the server falls through to Tier 4.

**Tier 4 - Strict check**: The existing behavior. The server raises `MlflowException` with the current error message instructing the operator to run `mlflow db upgrade`.

A new helper `_is_schema_behind(current_rev, head_revision)` walks the Alembic revision chain to determine directionality. This prevents auto-compat from activating when the database is ahead of the code (which would indicate a downgrade scenario).

### 3. `mlflow db check-upgrade` CLI command

A new subcommand under `mlflow db` that analyzes the upgrade path from the database's current revision to the latest head:

```bash
mlflow db check-upgrade <database-url> [--json]
```

**Human-readable output** lists each pending migration with a safety icon (`+`=safe, `~`=cautious, `!`=breaking), its operations, and any notes.

**JSON output** (`--json`) produces machine-parseable results for CI pipeline integration.

**Exit codes**: 0 = all safe, 1 = cautious migrations present, 2 = breaking migrations present.

This allows operators to pre-check upgrade safety in CI/CD pipelines before deploying.

### 4. `dev/schema_diff.py` operator tool

A standalone script that provides the same analysis without requiring a database connection:

```bash
# With database connection
python dev/schema_diff.py --db-url "postgresql://..."

# Without database connection (revision range only)
python dev/schema_diff.py --from-revision abc123 --to-revision def456

# JSON output for CI
python dev/schema_diff.py --from-revision abc123 --to-revision def456 --json
```

Same exit codes as `mlflow db check-upgrade`. This is intended for CI jobs that validate migration safety as part of the release process.

### 5. Environment variable

`MLFLOW_ALLOW_SCHEMA_MISMATCH` is a new boolean environment variable (default: `false`) registered in `mlflow/environment_variables.py`. When set to `true`, it activates Tier 2 behavior.

### Recommended deployment workflow

```
                     ┌──────────────────────┐
                     │ mlflow db check-upgrade│
                     └──────────┬───────────┘
                                │
                  ┌─────────────┼─────────────┐
                  │             │             │
              exit 0        exit 1        exit 2
            (all SAFE)    (CAUTIOUS)    (BREAKING)
                  │             │             │
                  v             v             v
           Rolling update   Review &     Scale down →
           (zero downtime)  decide       migrate →
                  │             │         scale up
                  v             │
           mlflow db upgrade   │
           (post-deploy job)   │
                               v
                    MLFLOW_ALLOW_SCHEMA_MISMATCH=true
                    + rolling update (if acceptable risk)
                    OR traditional workflow
```

## Drawbacks

- **Maintenance burden of the classifier**: The AST-based classifier must correctly handle all Alembic operation patterns used in MLflow migrations. New operation patterns may require updates to the classifier. The manual overrides dictionary must be maintained as new edge-case migrations are added.

- **False sense of safety**: The classifier makes static judgments about migration safety. It cannot account for all runtime behaviors. For example, a `create_index` on a very large table could cause lock contention in some databases, which is operationally unsafe even though it's structurally additive. The documentation should clearly state that "SAFE" means structurally additive, not guaranteed zero-impact.

- **Complexity in the startup path**: The tiered verification adds branching logic to server startup. If the classifier raises an unexpected exception, the fallback to Tier 4 ensures safety, but the added code paths increase the surface area for bugs.

- **Environment variable as escape hatch**: `MLFLOW_ALLOW_SCHEMA_MISMATCH=true` bypasses all safety checks. If operators set this permanently and forget about it, they could run into data corruption from genuinely breaking migrations. The documentation should emphasize this is a temporary override, not a permanent setting.

# Alternatives

### 1. Environment variable only (no classifier)

Add only `MLFLOW_ALLOW_SCHEMA_MISMATCH` and let operators decide. Simpler to implement but puts the full burden of migration analysis on the operator. Rejected because the classifier is the high-value component that makes this feature practical.

### 2. Schema version tolerance window

Allow the server to start if the database is within N revisions of the expected version, regardless of migration content. Simple but unsafe: a single breaking migration within the tolerance window would cause failures.

### 3. Maintain a manual safety manifest

Instead of AST parsing, maintain a YAML/JSON file that maps each migration revision to its safety level. More explicit but creates a maintenance burden on every migration and risks going stale. The AST parser automates this while the manual overrides dictionary handles edge cases.

### 4. Database-level compatibility views

Create SQL views that present the old schema shape to old code while the new schema evolves underneath. Powerful but extremely complex, database-specific, and far beyond the scope of MLflow's current architecture.

### 5. Do nothing

Operators continue using the stop-the-world workflow. This is the status quo and works, but it forces unnecessary downtime for the majority of releases that only include additive migrations.

# Adoption strategy

This is a fully backward-compatible change. Existing users experience zero difference unless they opt in:

- **No changes to default behavior**: Tier 1 (exact match) and Tier 4 (strict check) are identical to current behavior. The only new automatic behavior is Tier 3, which only activates when all pending migrations are classified as SAFE.
- **Opt-in escape hatch**: `MLFLOW_ALLOW_SCHEMA_MISMATCH` is `false` by default.
- **New CLI command**: `mlflow db check-upgrade` is additive and does not affect existing commands.
- **Documentation**: The self-hosting migration guide is updated with a "Zero-Downtime Upgrades" section documenting the new workflow, safety levels, and the environment variable.

For teams already running MLflow in production, adoption is straightforward:

1. Add `mlflow db check-upgrade` to CI/CD pipeline
2. If exit code is 0, switch to rolling deployment
3. If exit code is 1 or 2, use traditional workflow

# Open questions

1. **Should CAUTIOUS migrations auto-allow with the env var, or require a separate flag?** Currently, `MLFLOW_ALLOW_SCHEMA_MISMATCH=true` overrides everything (Tier 2). Should there be a more granular `MLFLOW_ALLOW_CAUTIOUS_MIGRATIONS=true` that only bypasses cautious migrations while still blocking breaking ones?

2. **Should the classifier run at import time or lazily?** Currently it runs on-demand during `_verify_schema()`. If classification is slow for repositories with many migrations, it could be pre-computed and cached.

3. **How should the manual overrides dictionary be maintained?** Currently it's hardcoded in the classifier module. Should it be extracted to a separate configuration file that's easier to review and update during migration development?

4. **Should there be a migration safety annotation for migration authors?** A decorator or comment convention (e.g., `# safety: safe`) that migration authors can add to explicitly declare safety, reducing reliance on AST inference.

5. **Index creation on large tables**: Should `create_index` be classified as CAUTIOUS instead of SAFE, given that it can cause lock contention on large tables in some databases? Or should this be database-engine-specific?
