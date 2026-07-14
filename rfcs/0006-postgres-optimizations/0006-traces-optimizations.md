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
- adding opt-in daily rollup tables, filled by a periodic job, for stable trace analytics ranges
- cleaning up duplicated metric, tag, and metadata rows once the denormalized columns become
  authoritative

On a load test with 10M traces, 30M spans, and 30M assessments in one experiment, the current Usage
page takes about 3 minutes to load and several Traces page metrics time out. This RFC aims to reduce
those query paths from minutes to seconds while keeping PostgreSQL as the analytics backend.

A prototype on the same 10M-trace dataset showed the following 32-day UI replay results. The replay
runs requests serially, so the total is cumulative request time rather than browser page-load wall
time with parallel requests. The rollup results below exclude the unbounded exact assessment-value
distribution grain; that query stays on the raw path.

| Request / metric path              | Upstream master | Prototype without rollups | Prototype with rollups | Rollup speedup vs master |
| ---------------------------------- | --------------- | ------------------------- | ---------------------- | ------------------------ |
| Full UI replay, 20 serial requests | 1,891.9s        | 119.4s                    | 36.8s                  | 51.5x                    |
| Cost over time by model            | 110.3s          | 21.3s                     | 56ms                   | 1,972x                   |
| Cost breakdown by model            | 24.2s           | 14.4s                     | 66ms                   | 367x                     |
| Token/latency daily time series    | 9.4-459.0s      | 5.4-9.8s                  | 64-74ms                | 135-6,788x               |
| Time-range trace count             | 315ms           | 558ms                     | 67ms                   | 4.7x                     |

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
- Changing UI pagination behavior beyond guarding expensive exact assessment-value distributions.
- A full request-shape guardrail design for trace metrics queries.
- A broad covering index on `trace_info` beyond the existing `(experiment_id, timestamp_ms)` access
  path.

## Current storage and query shape

Trace analytics currently combine trace rows with EAV-style metric tables.

At a high level:

- `trace_info` stores one row per trace, including experiment, request id (trace id in API terms),
  timestamp, status, and execution duration.
- `trace_metrics` stores trace-level metric key/value pairs keyed by `(request_id, key)`. Token
  usage metrics are stored here today.
- `spans` stores one row per span.
- `span_metrics` stores span-level metric key/value pairs keyed by `(trace_id, span_id, key)`. Cost
  metrics are stored here today.
- `assessments` stores one row per assessment and points back to a trace via `trace_id`.

This layout is flexible, but common dashboard queries must join through these tables before
aggregating. For example, a token time-series query currently has to filter traces by experiment and
time range, join to `trace_metrics`, filter by metric key, and then group by time bucket:

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

The proposed design moves the hottest analytics fields onto the rows already filtered by experiment
and time, so these paths become direct column aggregations instead of EAV joins plus runtime value
extraction.

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
- `input_cost`
- `output_cost`
- `total_cost`

These values are derived from the write paths that already produce them:

- `trace_name` from trace tags
- `session_id` from trace metadata
- token values from `TOKEN_USAGE`
- trace-level cost values from trace `COST` metadata, falling back to sums of legacy span cost
  metrics during backfill

After migration, the five token fields above (`input_tokens`, `output_tokens`, `total_tokens`,
`cache_read_input_tokens`, and `cache_creation_input_tokens`) become the single-source-of-truth
fields on `trace_info`. MLflow should:

1. backfill the new columns from `trace_metrics`
2. stop writing those token keys to `trace_metrics`
3. delete existing rows for those token keys from `trace_metrics` in batches
4. reconstruct those rows from `trace_info` on downgrade before dropping the denormalized columns

Non-denormalized custom trace metrics remain in `trace_metrics`.

The trace-level cost fields store authoritative trace totals for gateway and trace-level readers.
They are distinct from span cost fields, which support model/provider analytics. Both should be
backfilled and kept consistent by their respective write paths before legacy cost rows are removed.

`trace_name` should also become the authoritative storage and query field on `trace_info` for
analytics and trace-name filtering. MLflow should:

1. backfill `trace_info.trace_name` from trace tags
2. retarget trace-name readers, including trace-name analytics filters and rollup builders, to
   `trace_info.trace_name`
3. synthesize the API-facing trace-name tag from `trace_info.trace_name` for compatibility
4. stop writing the trace-name tag separately
5. delete existing trace-name tag rows in batches
6. reconstruct those tag rows from `trace_info.trace_name` on downgrade before dropping the
   denormalized column

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

Add the following columns:

- `experiment_id`, without a foreign key constraint
- `trace_timestamp_ms`
- `aggregate_value`

Assessment analytics should use `assessments` as the driving table. The hot query path becomes:

- filter directly on `experiment_id`
- filter and bucket on `trace_timestamp_ms`
- filter on `valid = true`
- use correlated `EXISTS` predicates only for trace-level filters such as status, tags, or metadata

`aggregate_value` materializes the numeric aggregation path up front so hot queries do not
repeatedly cast or reinterpret assessment values. The difference in query shape looks like this:

Before (simplified; the existing coercion path also handles supported string values):

```sql
SELECT a.name,
       AVG(
           CASE
               WHEN jsonb_typeof(a.value::jsonb) = 'number' THEN (a.value::jsonb)::text::double precision
               WHEN jsonb_typeof(a.value::jsonb) = 'boolean' THEN
                   CASE WHEN (a.value::jsonb)::boolean THEN 1.0 ELSE 0.0 END
               ELSE NULL
           END
       )
FROM assessments a
JOIN trace_info ti
  ON ti.request_id = a.trace_id
WHERE ti.experiment_id = ?
  AND ti.timestamp_ms BETWEEN ? AND ?
  AND a.valid = true
GROUP BY a.name;
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
has to repeatedly inspect `value`, apply `CASE` / `CAST` logic, and decide whether JSON numbers,
booleans, and the supported `"yes"` / `"no"` categorical strings are aggregateable on every query.
The write path should populate `aggregate_value` only for those values and leave it null for other
strings, including JSON numeric strings such as `"0.8"`, to preserve existing reader semantics.

MLflow already knows the experiment from the owning trace when it writes the assessment, so storing
`experiment_id` does not require an extra integrity lookup on every insert. Skipping the foreign key
keeps this high-volume write path cheaper while preserving the same logical relationship.

After migration, MLflow should:

1. backfill `trace_timestamp_ms` from the owning trace rows
2. backfill `aggregate_value` from existing assessment values using the same numeric coercion rules
   that analytics readers use today
3. retarget assessment analytics to `assessments.experiment_id`, `assessments.trace_timestamp_ms`,
   and `assessments.aggregate_value`
4. whenever an owning trace's experiment or timestamp changes, transactionally update those fields
  on all existing assessments and invalidate both the old and new rollup partitions

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

### 5. Opt-in SQL daily rollups

Denormalization removes the EAV joins from hot paths, but large dashboard windows still repeatedly
aggregate over millions of rows. This RFC proposes an opt-in SQL rollup layer for PostgreSQL-backed
trace analytics. The rollups are disabled by default and enabled with a server-side configuration
flag, so deployments can choose the extra storage and maintenance work only when they need it.

Add three daily rollup tables:

| Table                            | Grain                                                                               | Dimension columns              | Measure columns                                                                              |
| -------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------ | -------------------------------------------------------------------------------------------- |
| `sql_trace_metric_daily_rollups` | `(workspace, experiment_id, rollup_day, metric_name, trace_status?, trace_name?)`   | `trace_status`, `trace_name`   | `sample_count`, `sum_value`, `min_value`, `max_value`, `p50_value`, `p90_value`, `p99_value` |
| `sql_span_cost_daily_rollups`    | `(workspace, experiment_id, rollup_day, metric_name, model_name?, model_provider?)` | `model_name`, `model_provider` | `sample_count`, `sum_value`, `min_value`, `max_value`                                        |
| `sql_assessment_daily_rollups`   | `(workspace, experiment_id, rollup_day, metric_name, assessment_name?)`             | `assessment_name`              | `sample_count`, `sum_value`, `min_value`, `max_value`                                        |

Each row should also have a deterministic, fixed-length `rollup_key` primary key derived from a
canonical type-tagged encoding of the grain. The encoding must distinguish null, empty string, and
dimension presence before hashing so legal user dimension values cannot collide or exceed the key
column length. Lookup indexes should start with `(workspace, experiment_id, rollup_day, metric_name)`,
with secondary dimension indexes for the common dimension filters.

The trace metric table stores one row per
`(workspace, experiment_id, rollup_day, metric_name, dimension grain)`. For trace metrics it stores
`sample_count`, `sum_value`, `min_value`, `max_value`, and daily `p50_value`, `p90_value`, and
`p99_value`. Daily percentiles are not composable into coarser buckets, so readers should use these
percentile columns only for single-experiment, complete UTC-day `P50`, `P90`, and `P99` requests.
Multi-experiment, non-daily, or unbucketed percentile requests should stay on the raw exact path.

The span-cost rollup table stores daily `input_cost`, `output_cost`, and `total_cost` aggregates by
model and provider grains. The assessment rollup table stores daily assessment counts and numeric
aggregates over `aggregate_value`, both globally and by assessment name. It intentionally does not
group by exact `assessment_value`: user-defined numeric scores, free-form strings, and structured
values can be effectively unbounded, causing the rollup to approach the raw assessment table's
cardinality. Queries grouped by exact assessment value therefore stay on the raw path. This does not
affect numeric `AVG` rollups over `aggregate_value`, which remain grouped by assessment name.

The rollup reader should use these tables for supported daily `COUNT`, `SUM`, and `AVG` queries and
for supported daily trace metric percentiles. Rollup eligibility, fallback, and maintenance behavior
should be centralized behind backend-specific planning code rather than scattered across individual
query paths. Requests with arbitrary filters, exact assessment-value dimensions, unsupported
dimensions or aggregations, unsupported percentile values, or bucket widths other than one day
should fall back to the raw query path.

The Traces table currently requests exact assessment-value counts when infinite pagination is
enabled, and infinite pagination is enabled by default. To keep that raw fallback bounded, add a
server setting named `MLFLOW_TRACE_METRICS_MAX_ASSESSMENT_VALUE_DISTRIBUTION_ROWS`, defaulting to
`1000000`. It limits the number of source assessment rows, after experiment, time, validity, and
request filters are applied, that an exact-value distribution may aggregate; it does not truncate
returned groups or counts.

Exact assessment-value distribution queries should be guarded by that setting so the raw fallback
remains bounded. This guard applies whether or not SQL rollups are enabled. If the guard blocks the
query, the UI should treat the global distribution as unavailable rather than presenting partial
page-local counts as complete results. The server should apply the same rule when the number of
result groups exceeds the API response limit, unless full pagination is implemented.

A periodic server job fills the tables. Each pass should:

1. compute eligible `(workspace, experiment_id, rollup_day)` partitions from raw trace, span, and
   assessment rows
2. exclude days newer than a configurable freshness threshold, controlled by
   `MLFLOW_SQL_TRACE_ROLLUPS_FRESHNESS_SECONDS` and defaulting to 24 hours
3. rebuild deleted/stale rollup partitions before processing newly eligible trace days
4. skip partitions that already have a coverage marker row
5. rebuild a bounded number of missing partitions per pass
6. upsert rollup rows by deterministic rollup key

Invalidated partitions should be tracked durably until a rebuild completes. This ensures deletes or
moves can repair an old partition even when no source rows remain there.

The freshness threshold keeps the job away from traces that are still being actively written.
Because SQL rollups are disabled by default, this late-mutation restriction is also off by default.
When operators enable rollups, this same server-side setting should define how old a trace can be
before late analytic updates are rejected.

Rollup invalidation and rebuild for a given partition should be serialized, and a rebuilt partition
should only become visible once its rows and coverage marker are published together. This prevents
concurrent rebuilders or late writes from making a stale partition appear complete again.

When SQL rollups are enabled, to keep rollups stable, MLflow should reject late mutations that
change rollup facts once a trace is older than the freshness window. Specifically, the MLflow API
should not allow adding spans, adding trace metrics, or updating trace-info fields such as
timestamp, duration, status, token usage, span cost, `trace_name`, `session_id`, or other analytic
facts for traces older than that window. Comments, display metadata, and similar non-analytic
annotations can remain mutable. Adding assessments to older traces should also remain allowed, since
human review and evaluator backfills often happen after the trace is closed.

Assessment writes or trace deletes against already-rolled-up traces should delete stale rollup rows
for the affected `(workspace, experiment_id, rollup_day)`. Until the next scheduled rollup job
rebuilds that day, readers should compute the missing day from raw rows and use rollups for the
remaining covered days. The job should prioritize rebuilding deleted rollup days before processing
newly eligible trace days, and bulk deletes should group affected traces by partition so each day is
deleted and rebuilt once.

The rollup tables need targeted indexes for lookup and dimension filtering. Span-cost rollup builds
also need a raw-table covering index that matches their day-range access pattern:

```sql
CREATE INDEX idx_spans_cost_exp_time_cover
  ON spans (experiment_id, start_time_unix_nano)
  INCLUDE (input_cost, output_cost, total_cost, dimension_attributes)
  WHERE input_cost IS NOT NULL
     OR output_cost IS NOT NULL
     OR total_cost IS NOT NULL;
```

This index avoids repeatedly scanning all spans when building or repairing daily span-cost rollups.

### 6. Targeted indexes

This RFC keeps the existing `(experiment_id, timestamp_ms)` access path on `trace_info` and does not
propose an additional broad covering index on `trace_info`.

That is intentional:

- a wide covering index on `trace_info` would increase index size and write amplification
- once hot metrics are denormalized, dense dashboard queries may still prefer grouped or parallel
  scan plans over wide index access

This RFC does propose the following targeted indexes:

```sql
-- Drives assessment time-series queries by experiment and trace timestamp without joining
-- through trace_info.
CREATE INDEX idx_assessments_exp_trace_ts
    ON assessments (experiment_id, trace_timestamp_ms);

-- Covers assessment time-series queries grouped or filtered by assessment name.
CREATE INDEX idx_assessments_exp_trace_ts_name
    ON assessments (experiment_id, trace_timestamp_ms, name);

-- Supports assessment distribution and name-filtered queries that aggregate valid rows.
CREATE INDEX idx_assessments_exp_name_valid
    ON assessments (experiment_id, name, valid);

-- Supports trace-first raw span-cost fallbacks: first select trace ids from trace_info,
-- then index into cost-bearing spans by trace id and span start time.
CREATE INDEX idx_spans_cost_trace_time_cover
    ON spans (trace_id, start_time_unix_nano)
  INCLUDE (input_cost, output_cost, total_cost, dimension_attributes)
  WHERE input_cost IS NOT NULL
     OR output_cost IS NOT NULL
     OR total_cost IS NOT NULL;

-- Supports daily span-cost rollup build and repair jobs, which scan cost-bearing spans by
-- experiment and day rather than by trace id.
CREATE INDEX idx_spans_cost_exp_time_cover
  ON spans (experiment_id, start_time_unix_nano)
  INCLUDE (input_cost, output_cost, total_cost, dimension_attributes)
  WHERE input_cost IS NOT NULL
     OR output_cost IS NOT NULL
     OR total_cost IS NOT NULL;
```

## Implementation details

- Migration order:
  1. add columns
  2. backfill `trace_info`
  3. backfill `spans`
  4. backfill `assessments`
  5. create opt-in rollup tables and targeted indexes
  6. retarget reads and writes
  7. batch-delete duplicated EAV rows
  8. optionally enable the periodic SQL rollup job
- The rollout may be split into two phases. An optional online pre-migration phase can add nullable
  columns, build compatible indexes, and run idempotent backfills while the pre-RFC MLflow version
  is still serving traffic. The final cutover phase should then do the write/read retargeting,
  catch-up backfill, legacy-row cleanup, and any destructive DDL. This is to reduce downtime from
  long SQL operations on large databases.
- Batched deletes from `trace_metrics`, `span_metrics`, and trace metadata will create dead tuples.
  Rollout guidance should mention normal autovacuum follow-up or `VACUUM (ANALYZE)` after large
  cleanup operations.
- If such an online pre-migration phase is implemented, either represent it as an explicit additive
  Alembic revision or require the final migration to detect and validate pre-created columns and
  indexes before continuing. The final migration must not blindly recreate those objects.
- The migration that removes or permits cleanup of legacy EAV rows must reconstruct token rows in
  `trace_metrics` from `trace_info`, cost rows in `span_metrics` from `spans`, trace-name tag rows
  from `trace_info.trace_name`, and `TRACE_SESSION` metadata rows from `trace_info.session_id` before
  an old version can be deployed or the denormalized columns can be dropped.
- Use `Float(53)` / `DOUBLE PRECISION` for denormalized token, span-cost, and aggregate numeric
  columns to match the existing analytics numeric path.
- In the Alembic migration code, `batch_op` should be used only for SQLite compatibility paths.
  Large Postgres or MySQL tables should use direct `ALTER TABLE` / `CREATE INDEX` statements to
  avoid expensive table recreation.

## Expected impact

Representative measured improvements on the 10M trace / 30M span / 30M assessment benchmark dataset:

| Query / replay request                       | Upstream master | Prototype without rollups | Prototype with rollups | Rollup speedup vs master | Rollup speedup vs no rollups |
| -------------------------------------------- | --------------- | ------------------------- | ---------------------- | ------------------------ | ---------------------------- |
| Full 32-day UI replay, 20 requests           | 1,891.9s        | 119.4s                    | 36.8s                  | 51.5x                    | 3.2x                         |
| `cost_over_time_by_model`                    | 110.3s          | 21.3s                     | 56ms                   | 1,972x                   | 381x                         |
| `cost_breakdown_by_model`                    | 24.2s           | 14.4s                     | 66ms                   | 367x                     | 218x                         |
| `input_tokens` daily sum                     | 298.9s          | 5.5s                      | 74ms                   | 4,047x                   | 74x                          |
| `output_tokens` daily sum                    | 360.9s          | 5.4s                      | 66ms                   | 5,468x                   | 82x                          |
| `cache_read_input_tokens` daily sum          | 459.0s          | 5.5s                      | 68ms                   | 6,770x                   | 81x                          |
| `cache_creation_input_tokens` daily sum      | 434.4s          | 5.4s                      | 64ms                   | 6,788x                   | 84x                          |
| `total_tokens` daily P50/P90/P99 time series | 56.2s           | 9.8s                      | 66ms                   | 853x                     | 149x                         |
| `latency` daily P50/P90/P99 time series      | 9.4s            | 9.8s                      | 70ms                   | 135x                     | 140x                         |
| `trace_count_by_status` daily count          | 6.1s            | 6.6s                      | 199ms                  | 31x                      | 33x                          |
| `assessment_value` daily time series         | 105.8s          | 10.2s                     | 9.4s                   | 11.3x                    | 1.1x                         |
| `assessment_distribution` on the Traces tab  | 10.0s           | 8.6s                      | 7.9s                   | 1.3x                     | 1.1x                         |
| `time_range_trace_count`                     | 315ms           | 558ms                     | 67ms                   | 4.7x                     | 8.4x                         |

These numbers are benchmark results from the same 10M-trace dataset using daily buckets over 32-day
windows that contain all traces. The upstream master column uses the original EAV-backed schema; the
prototype columns use the optimized schema with SQL rollups disabled and enabled. The with-rollups
values are medians from three fixed-range replays after removing the exact assessment-value rollup,
so exact-value distributions use raw PostgreSQL. The proposed source-row cap was disabled for this
measurement.

## Trade-offs

- Additional columns and targeted indexes still increase storage, even after cleanup removes the
  duplicated hot EAV rows and session metadata.
- Opt-in rollup tables add more storage and operational state. Deployments that enable them need a
  periodic job, freshness policy, and dirty-partition repair path for deletes.
- Exact assessment-value distributions are not accelerated because their user-defined grouping
  dimension is not reliably bounded. The server rejects distributions above the configured source
  assessment-row cap, so the UI may omit global distribution summaries for large ranges.
- Retargeting reads and writes, cleaning up duplicated rows, and supporting downgrade reconstruction
  increase implementation and migration complexity.
- Assessment trace-time columns and rollup dimensions must stay consistent if later write paths
  update trace timing or analytic fields.

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
- Optionally support a phased rollout where operators first run an additive pre-migration utility to
  add the new columns, create compatible indexes, and backfill data idempotently before the main
  migration window. The final migration must detect and validate those pre-created objects, or the
  additive phase must be represented as its own Alembic revision.
- Reserve downtime for the final cutover steps: schema-version advancement, final catch-up backfill,
  read/write retargeting, legacy-row cleanup, and any destructive DDL such as constraint or column
  drops.
- Keep SQL daily rollups disabled by default. Operators can enable them after the denormalized
  fields are backfilled and the scheduler is running.
- The late-mutation restriction stays off until SQL daily rollups are enabled. When operators enable
  rollups, enforce a configurable immutability window for analytic trace facts, controlled by
  `MLFLOW_SQL_TRACE_ROLLUPS_FRESHNESS_SECONDS` and defaulting to 24 hours, so periodic rollup jobs
  only need to process closed days and delete-triggered dirty partitions.
- Before deploying an old reader, stop writes, reconstruct and validate removed metric, tag, and
  metadata rows, and only then drop denormalized columns that the old version cannot use.

## Open question

Should the trace metrics API enforce server-side query-shape caps, such as maximum bucket count,
minimum bucket width, or limits on raw sample work for exact percentile queries? Existing response
caps limit returned points, but do not necessarily bound total scan or aggregation cost.
