---
start_date: 2026-06-26
mlflow_issue: # TBD — will file enhancement issue in mlflow/mlflow
rfc_pr: # leave empty until PR is created
---

# PostgreSQL Trace Analytics Optimization

## Summary

Optimize PostgreSQL trace analytics for MLflow's Usage page and Traces page by:

- denormalizing hot analytics fields onto `trace_info`, `spans`, and `assessments`
- rewriting hot queries to read those columns directly
- adding targeted indexes for assessment and span-cost workloads
- cleaning up duplicated EAV rows once the denormalized columns become authoritative

On a load test with 10M traces, 30M spans, and 30M assessments in one experiment, the current Usage
page takes about 3 minutes to load and several Traces page metrics time out. This RFC aims to reduce
those query paths from minutes to seconds while keeping PostgreSQL as the analytics backend.

## Motivation

The current schema is optimized for write flexibility, not for large analytic aggregations. At
scale, the slowest trace analytics queries share four problems:

1. The existing `(experiment_id, timestamp_ms)` access path on `trace_info` still fans out into
   large joins for hot metrics and assessments.
2. Hot token metrics are stored in `trace_metrics`, which forces join-heavy EAV queries for common
   dashboard paths.
3. Hot span cost metrics are stored in `span_metrics`, which creates the same duplication for span
   analytics and gateway cost aggregation.
4. Assessment analytics currently depend on `trace_info` joins plus runtime value conversion in hot
   queries.

## Out of scope

- Moving analytics to a different database engine.
- UI pagination or client-side query shaping.
- A full request-shape guardrail design for trace metrics queries.
- A broad covering index on `trace_info` beyond the existing `(experiment_id, timestamp_ms)` access
  path.

## Current storage and query shape

Trace analytics currently combine trace rows with EAV-style metric tables.

At a high level:

- `trace_info` stores one row per trace, including experiment, request id
  (trace id in API terms), timestamp, status, and execution duration.
- `trace_metrics` stores trace-level metric key/value pairs keyed by
  `(request_id, key)`. Token usage metrics are stored here today.
- `spans` stores one row per span.
- `span_metrics` stores span-level metric key/value pairs keyed by
  `(trace_id, span_id, key)`. Cost metrics are stored here today.
- `assessments` stores one row per assessment and points back to a trace via
  `trace_id`.

This layout is flexible, but common dashboard queries must join through these
tables before aggregating. For example, a token time-series query currently has
to filter traces by experiment and time range, join to `trace_metrics`, filter
by metric key, and then group by time bucket:

```sql
SELECT floor(ti.timestamp_ms / :bucket_ms) * :bucket_ms AS time_bucket,
       SUM(tm.value) AS input_tokens
FROM trace_info ti
JOIN trace_metrics tm
  ON tm.request_id = ti.request_id
WHERE ti.experiment_id = :experiment_id
  AND ti.timestamp_ms BETWEEN :start_time_ms AND :end_time_ms
  AND tm.key = 'input_tokens'
GROUP BY floor(ti.timestamp_ms / :bucket_ms) * :bucket_ms
ORDER BY time_bucket;
```

The proposed design moves the hottest analytics fields onto the rows already
filtered by experiment and time, so these paths become direct column
aggregations instead of EAV joins plus runtime value extraction.

## Detailed design

### 1. Denormalize hot trace analytics onto `trace_info`

Add the following columns to `trace_info`:

- `trace_name`
- `session_id`
- `input_tokens`
- `output_tokens`
- `total_tokens`
- `cache_read_input_tokens`
- `cache_creation_input_tokens`

These values are derived from the write paths that already produce them:

- `trace_name` from trace tags
- `session_id` from trace metadata
- token values from `TOKEN_USAGE`

After migration, these five token keys become single-source-of-truth fields on `trace_info`. MLflow
should:

1. backfill the new columns from `trace_metrics`
2. stop writing those token keys to `trace_metrics`
3. delete existing rows for those token keys from `trace_metrics` in batches
4. reconstruct those rows from `trace_info` on downgrade before dropping the denormalized columns

Non-denormalized custom trace metrics remain in `trace_metrics`.

`session_id` should become the authoritative storage and query field on `trace_info` for analytics
and session-oriented queries. MLflow should:

1. backfill `trace_info.session_id` from trace metadata
2. retarget session readers, including session-count aggregation and completed session queries, to
   `trace_info.session_id`
3. synthesize `TRACE_SESSION` in API-facing trace metadata from `trace_info.session_id` for
   compatibility
4. stop writing `TRACE_SESSION` to trace metadata
5. delete existing `TRACE_SESSION` metadata rows in batches
6. reconstruct those metadata rows from `trace_info.session_id` on downgrade before dropping the
   denormalized column

### 2. Denormalize hot span cost analytics onto `spans`

Add the following columns to `spans`:

- `input_cost`
- `output_cost`
- `total_cost`

Span cost aggregation should read these columns directly. Any remaining cost readers, including
`sum_gateway_trace_cost()`, should also be retargeted to `spans.total_cost` so cost metrics do not
remain duplicated across code paths.

After migration, MLflow should:

1. backfill the new columns from `span_metrics`
2. stop writing these cost keys to `span_metrics`
3. delete existing rows for those cost keys from `span_metrics` in batches
4. reconstruct those rows from `spans` on downgrade before dropping the denormalized columns

This RFC also keeps the `spans.experiment_id` denormalization and does not require a foreign key on
that column.

### 3. Denormalize assessment analytics onto `assessments`

Add the following columns to `assessments`:

- `experiment_id`
- `trace_timestamp_ms`
- `aggregate_value`

Assessment analytics should use `assessments` as the driving table. The hot query path becomes:

- filter directly on `experiment_id`
- filter and bucket on `trace_timestamp_ms`
- filter on `valid = true`
- use correlated `EXISTS` predicates only for trace-level filters such as status, tags, or metadata

`aggregate_value` materializes the numeric aggregation path up front so hot queries do not
repeatedly cast or reinterpret assessment values. The difference in query shape looks like this:

Before:

```sql
SELECT name,
       AVG(
           CASE
               WHEN jsonb_typeof(value::jsonb) = 'number' THEN (value::jsonb)::text::double precision
               WHEN jsonb_typeof(value::jsonb) = 'boolean' THEN
                   CASE WHEN (value::jsonb)::boolean THEN 1.0 ELSE 0.0 END
               ELSE NULL
           END
       )
FROM assessments
WHERE experiment_id = ?
  AND trace_timestamp_ms BETWEEN ? AND ?
  AND valid = true
GROUP BY name;
```

After:

```sql
SELECT name, AVG(aggregate_value)
FROM assessments
WHERE experiment_id = ?
  AND trace_timestamp_ms BETWEEN ? AND ?
  AND valid = true
  AND aggregate_value IS NOT NULL
GROUP BY name;
```

The optimized query aggregates directly on a numeric column. Without `aggregate_value`, the hot path
has to repeatedly inspect `value`, apply `CASE` / `CAST` logic, and decide whether JSON booleans,
numeric strings, and true numeric values are aggregateable on every query. The write path should
populate `aggregate_value` only for assessment values that MLflow intentionally treats as numeric in
analytics and leave it null otherwise.

`experiment_id` should remain denormalized but the foreign key constraint removed. MLflow already
knows the experiment from the owning trace when it writes the assessment, so storing that value does
not require an extra integrity lookup on every insert. Skipping the foreign key keeps this
high-volume write path cheaper while preserving the same logical relationship.

### 4. Query execution changes

The main read-path changes are:

- trace token metrics aggregate directly from `trace_info`
- span cost metrics aggregate directly from `spans`
- trace name and session count read from `trace_info`
- completed-session queries read `session_id` from `trace_info`
- API-facing trace metadata can synthesize `TRACE_SESSION` from `trace_info.session_id` for
  compatibility
- assessment analytics query `assessments` directly instead of joining from `trace_info`

For Postgres span analytics, use a trace-first query shape:

1. split filters into trace-level and span-level predicates
2. build a `metric_trace_ids` CTE (common table expression e.g. SELECT) from the filtered trace set
3. mark that CTE as `MATERIALIZED` on Postgres
4. join spans from that materialized trace-id set

This keeps PostgreSQL on the measured trace-first plan instead of inlining back to a slower
span-first scan. This was discovered from a proof of concept implementation.

### 5. Targeted indexes

This RFC keeps the existing `(experiment_id, timestamp_ms)` access path on `trace_info` and does not
propose an additional broad covering index on `trace_info`.

That is intentional:

- a wide covering index on `trace_info` would increase index size and write amplification
- once hot metrics are denormalized, dense dashboard queries may still prefer grouped or parallel
  scan plans over wide index access

This RFC does propose the following targeted indexes:

```sql
CREATE INDEX idx_assessments_exp_trace_ts
    ON assessments (experiment_id, trace_timestamp_ms);

CREATE INDEX idx_assessments_exp_trace_ts_name
    ON assessments (experiment_id, trace_timestamp_ms, name);

CREATE INDEX idx_assessments_exp_name_valid
    ON assessments (experiment_id, name, valid);

CREATE INDEX idx_spans_cost_trace_time_cover
    ON spans (trace_id, start_time_unix_nano)
    INCLUDE (total_cost, dimension_attributes)
    WHERE total_cost IS NOT NULL;
```

## Implementation details

- Migration order:
  1. add columns
  2. backfill `trace_info`
  3. backfill `spans`
  4. backfill `assessments`
  5. retarget reads and writes
  6. batch-delete duplicated EAV rows
  7. create indexes
- Batched deletes from `trace_metrics`, `span_metrics`, and trace metadata will create dead tuples.
  Rollout guidance should mention normal autovacuum follow-up or `VACUUM (ANALYZE)` after large
  cleanup operations.
- Downgrade should reconstruct token rows in `trace_metrics` from `trace_info`, cost rows in
  `span_metrics` from `spans`, and `TRACE_SESSION` metadata rows from `trace_info.session_id` before
  dropping the denormalized columns.
- Use `Float(53)` / `DOUBLE PRECISION` for denormalized token, span-cost, and aggregate numeric
  columns to match the existing analytics numeric path.
- In the Alembic migration code, `batch_op` should be used only for SQLite compatibility paths.
  Large Postgres or MySQL tables should use direct `ALTER TABLE` / `CREATE INDEX` statements to
  avoid expensive table recreation.

## Expected impact

Representative target improvements on the benchmark dataset:

| Query                          | Current | Target  |
| ------------------------------ | ------- | ------- |
| Usage page total               | ~3 min  | <10s    |
| `input_tokens` daily sum       | 114.8s  | ~2-3s   |
| `assessment_value` AVG/P90/P99 | 105.8s  | ~8-10s  |

These numbers are benchmark-based targets, not guarantees. Exact percentile queries may still
retain significant sort cost even after the denormalization work, and are expected to remain the
main residual cost on the slowest paths.

## Drawbacks

- Additional columns and targeted indexes still increase storage, even after cleanup removes the
  duplicated hot EAV rows and session metadata.
- Retargeting reads and writes, cleaning up duplicated rows, and supporting downgrade reconstruction
  increase implementation and migration complexity.
- Assessment trace-time columns must stay consistent if later write paths update trace timing.

## Alternatives considered

### Separate analytics store

A separate columnar analytics store can be faster, but it adds operational complexity and is outside
the scope of this RFC.

### Broad covering index on `trace_info`

Not selected. The existing `(experiment_id, timestamp_ms)` path remains the primary trace-range
access path in this RFC. Any broader covering or expression index on `trace_info` should be
benchmarked separately before adoption.

### `generate_series + LATERAL` bucket aggregation

This query shape was investigated for Postgres bucketed aggregations and was materially slower on
the benchmark dataset than the grouped aggregate plan. On the tested dataset, repeated per-bucket
probes lost to PostgreSQL's grouped or parallel scan plan, so this is not part of the recommended
path for this RFC. It could be revisited later for sparser datasets or different index shapes if new
benchmarks justify it.

## Adoption strategy

- Ship through a standard Alembic migration.
- Require a maintenance window for the backfill, write-path cutover, and EAV cleanup in the database
  migration.
- Downgrade should reconstruct the removed EAV rows before dropping the denormalized columns.

## Open question

Should the trace metrics API enforce server-side query-shape caps, such as maximum bucket count,
minimum bucket width, or limits on raw sample work for exact percentile queries? Existing response
caps limit returned points, but do not necessarily bound total scan or aggregation cost.
