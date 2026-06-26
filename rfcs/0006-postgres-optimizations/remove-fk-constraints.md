start_date: 2026-06-26
mlflow_issue: # TBD
rfc_pr:       # TBD

# Summary

Remove ~10 foreign key (FK) constraints from high-volume tracking tables where the application layer already validates parent existence before every write and SQLAlchemy's ORM cascade already handles child deletion. This reduces per-row insert overhead on the hottest write paths (metrics, params, tags).

# Motivation

Every time MLflow logs a metric, param, or tag, the database performs an FK lookup to verify the parent `run_uuid` or `experiment_id` exists. This check is redundant because:

1. The store layer (`SqlAlchemyStore`) already calls `_get_run()` or `_get_experiment()` before writing, which raises `RESOURCE_DOES_NOT_EXIST` if the parent is missing.
2. SQLAlchemy's `relationship(..., cascade="all")` already handles child deletion when a parent is deleted through the ORM — it emits explicit DELETE statements for children without needing the database's `ON DELETE CASCADE`.

These FK checks add measurable overhead on high-throughput workloads (batch metric logging, parallel run creation). Removing them lets the database skip the index lookup on the parent table for every child row insert (depending on database engine; need to be vetted).

## Detailed design

### FK constraints to remove

**Run-scoped child tables → `runs.run_uuid`:**

| Table | Column |
|---|---|
| `tags` | `SqlTag.run_uuid` |
| `metrics` | `SqlMetric.run_uuid` |
| `latest_metrics` | `SqlLatestMetric.run_uuid` |
| `params` | `SqlParam.run_uuid` |

**Denormalized `experiment_id` references → `experiments.experiment_id`:**

| Table | Column |
|---|---|
| `logged_model_metrics` | `SqlLoggedModelMetric.experiment_id` |
| `logged_model_params` | `SqlLoggedModelParam.experiment_id` |
| `logged_model_tags` | `SqlLoggedModelTag.experiment_id` |
| `spans` | `SqlSpan.experiment_id` |
| `online_scoring_configs` | `SqlOnlineScoringConfig.experiment_id` |

**Run → experiment reference:**

| Table | Column |
|---|---|
| `runs` | `SqlRun.experiment_id` |

### Why this is safe

1. **Writes are already guarded.** Every write path in `SqlAlchemyStore` validates the parent exists before inserting. This has been the pattern since the beginning of the project (todo: double check again with Mlflow code).

2. **Deletes are already handled by the ORM.** The `relationship(..., cascade="all")` on the parent model tells SQLAlchemy to emit explicit `DELETE` statements for child rows. This is independent of the database FK — removing the FK does not break this behavior.

3. **Precedent exists in this codebase.** `SqlInput.source_id`/`destination_id` and `SqlReviewQueueLabelSchema.schema_id` already skip FK constraints and rely on app-level validation.

### Migration approach

A single Alembic migration that drops the FK constraints:

```python
def upgrade():
    # Run-scoped tables
    op.drop_constraint("fk_tags_run_uuid", "tags", type_="foreignkey")
    op.drop_constraint("fk_metrics_run_uuid", "metrics", type_="foreignkey")
    op.drop_constraint("fk_latest_metrics_run_uuid", "latest_metrics", type_="foreignkey")
    op.drop_constraint("fk_params_run_uuid", "params", type_="foreignkey")
    
    # Denormalized experiment_id columns
    op.drop_constraint("fk_logged_model_metrics_experiment_id", "logged_model_metrics", type_="foreignkey")
    op.drop_constraint("fk_logged_model_params_experiment_id", "logged_model_params", type_="foreignkey")
    op.drop_constraint("fk_logged_model_tags_experiment_id", "logged_model_tags", type_="foreignkey")
    op.drop_constraint("fk_spans_experiment_id", "spans", type_="foreignkey")
    op.drop_constraint("fk_online_scoring_configs_experiment_id", "online_scoring_configs", type_="foreignkey")
    
    # Run → experiment
    op.drop_constraint("fk_runs_experiment_id", "runs", type_="foreignkey")


def downgrade():
    # Re-add all FK constraints
    op.create_foreign_key("fk_tags_run_uuid", "tags", "runs", ["run_uuid"], ["run_uuid"])
    op.create_foreign_key("fk_metrics_run_uuid", "metrics", "runs", ["run_uuid"], ["run_uuid"])
    op.create_foreign_key("fk_latest_metrics_run_uuid", "latest_metrics", "runs", ["run_uuid"], ["run_uuid"])
    op.create_foreign_key("fk_params_run_uuid", "params", "runs", ["run_uuid"], ["run_uuid"])
    op.create_foreign_key("fk_logged_model_metrics_experiment_id", "logged_model_metrics", "experiments", ["experiment_id"], ["experiment_id"])
    op.create_foreign_key("fk_logged_model_params_experiment_id", "logged_model_params", "experiments", ["experiment_id"], ["experiment_id"])
    op.create_foreign_key("fk_logged_model_tags_experiment_id", "logged_model_tags", "experiments", ["experiment_id"], ["experiment_id"])
    op.create_foreign_key("fk_spans_experiment_id", "spans", "experiments", ["experiment_id"], ["experiment_id"])
    op.create_foreign_key("fk_online_scoring_configs_experiment_id", "online_scoring_configs", "experiments", ["experiment_id"], ["experiment_id"])
    op.create_foreign_key("fk_runs_experiment_id", "runs", "experiments", ["experiment_id"], ["experiment_id"])
```

### Model changes

Remove `ForeignKey(...)` from column definitions but keep `relationship()` and indexes intact:

```python
# Before
run_uuid = Column(String(32), ForeignKey("runs.run_uuid"), nullable=False)

# After
run_uuid = Column(String(32), nullable=False, index=True)
```

The `relationship()` on `SqlRun` stays unchanged — it doesn't depend on the FK to function.

## Drawbacks

- **Orphan risk from direct DB access.** If someone bypasses the ORM and issues raw SQL that creates child rows with invalid `run_uuid` values, the database won't catch it. Mitigated by the fact that all access goes through `SqlAlchemyStore`.
- **Orphan risk from raw parent deletes.** If someone deletes a run via raw SQL (not through `session.delete()`), children won't be cleaned up automatically. Same mitigation.
- **Reduced schema self-documentation.** FK constraints make relationships visible at the schema level. Mitigated by the `relationship()` definitions and this RFC as documentation.

# Alternatives

**Do nothing.** The FK overhead is small per-row but adds up at scale. For deployments with heavy metric logging, this could be a meaningful optimization.

# Adoption strategy

- This is a non-breaking change. No API changes, no behavior changes for users going through the standard MLflow client.
- Deployed via a standard Alembic migration. Existing data is unaffected (dropping an FK constraint doesn't modify rows).
- The migration is backward-compatible — the downgrade path re-adds the constraints.

# Sections skipped

- **Basic example**: No API or UI changes — this is a schema-only optimization invisible to users.
