---
start_date: 2026-06-26
mlflow_issue: # TBD; will file enhancement issue in mlflow/mlflow
rfc_pr: # leave empty until PR is created
---

# PostgreSQL Trace Analytics Optimization

## Summary

Denormalize hot trace analytics and add optional daily rollups across supported SQL backends, with
PostgreSQL-specific query planning and indexes, by:

- denormalizing hot analytics fields onto `trace_info`, `spans`, and `assessments`
- rewriting hot queries to read those columns directly
- adding targeted indexes for session, assessment, and span-cost workloads
- adding SQL daily rollup tables whose population and use are opt-in
- cleaning up duplicated metric, tag, and metadata rows once the denormalized columns become
  authoritative

On a load test with 10M traces, 30M spans, and 30M assessments in one experiment, the current Usage
page takes about 3 minutes to load and several Traces page metrics time out. This RFC aims to reduce
those query paths from minutes to seconds while keeping PostgreSQL as the analytics backend.

On the same dataset, denormalization reduced a 20-request serial replay from 1,891.9 seconds to
119.4 seconds; enabling rollups reduced it further to 36.8 seconds. See
[Expected impact](#expected-impact) for detailed results and benchmark limitations.

## Motivation

The current schema is optimized for write flexibility, not for large analytic aggregations. At
scale, the slowest trace analytics queries share four problems:

1. The existing `(experiment_id, timestamp_ms)` access path on `trace_info` fans out into large
   joins for hot metrics and assessments.
2. Hot token metrics are stored in `trace_metrics`, which forces join-heavy EAV queries for common
   dashboard paths.
3. Hot span cost metrics are stored as EAV rows in `span_metrics`, forcing joins for span analytics
  and gateway cost aggregation.
4. Assessment analytics currently depend on `trace_info` joins plus runtime value conversion in hot
   queries.

## Out of scope

- Moving analytics to a different database engine.
- Trace-table pagination changes. Exact assessment distributions remain range-global; their
  guardrails are an open question below.
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
  AND ti.timestamp_ms >= :start_time_ms
  AND ti.timestamp_ms < :end_time_exclusive_ms
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
- token values from reserved token metric rows
- trace-level cost values from trace `COST` metadata

After migration, the five token fields above (`input_tokens`, `output_tokens`, `total_tokens`,
`cache_read_input_tokens`, and `cache_creation_input_tokens`) become the only stored token
representation. MLflow should:

1. backfill the columns from reserved `trace_metrics` rows
2. retarget reserved token writes to the columns
3. synthesize API-facing reserved metrics and `TOKEN_USAGE` metadata from the columns
4. delete the duplicated stored metric and metadata rows in batches
5. reconstruct the representations required by the old version on downgrade before dropping the
   columns

Non-denormalized custom trace metrics remain in `trace_metrics`.

Cost authority is query-view specific. The `trace_info` cost columns are reusable trace totals for
trace-level cost APIs and gateway budget accounting, avoiding span joins when a model/provider
breakdown is not needed. The `spans` cost columns are authoritative for span-view metrics, including
model/provider breakdowns. Readers must not substitute one source for the other.

`trace_name` should also become the authoritative storage and query field on `trace_info` for
analytics and trace-name filtering. MLflow should:

1. backfill `trace_info.trace_name` from trace tags
2. retarget trace-name readers, including analytics filters and raw grouped queries, to
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

Supported writes to denormalized reserved fields must update their authoritative columns rather than
recreate legacy duplicate rows. This includes setting or deleting the trace-name tag, writing
session metadata, and logging reserved token or cost keys. API-facing tags, metadata, and metrics
remain synthesized for compatibility, and trace search filters for these reserved keys must also be
retargeted to the authoritative columns.

### 2. Denormalize hot span cost analytics onto `spans`

Add the following columns to `spans`:

- `input_cost`
- `output_cost`
- `total_cost`
- `model_name`
- `model_provider`

Model name and provider are fixed, first-class dimensions of span-cost analytics. They should be
stored as nullable columns rather than in the existing `dimension_attributes` JSON column. Span cost
aggregation and grouping should read these columns directly. Span costs remain authoritative for
model/provider breakdowns, while gateway budget accounting should read the authoritative trace-level
total from `trace_info.total_cost` so each gateway trace is counted once without a span join.

After migration, MLflow should:

1. backfill cost columns from `span_metrics` and model/provider columns from `dimension_attributes`
2. write cost, model, and provider values directly to the new columns
3. retarget raw analytics and rollup construction to the new columns
4. delete existing cost rows from `span_metrics` in batches and drop `dimension_attributes`
5. reconstruct cost rows and `dimension_attributes` from non-null denormalized columns on downgrade
   before dropping those columns

### 3. Denormalize assessment analytics onto `assessments`

Add the following columns:

- `experiment_id`, without a foreign key constraint
- `trace_timestamp_ms`
- `aggregate_value`
- `is_numeric_value`

Assessment analytics should use `assessments` as the driving table. The hot query path becomes:

- filter directly on `experiment_id`
- filter and bucket on `trace_timestamp_ms`
- filter on `valid = true`
- use correlated `EXISTS` predicates only for trace-level filters such as status, tags, or metadata

Today, assessment aggregates join `trace_info` and convert JSON values at query time:

```sql
SELECT a.name,
     AVG(
       CASE
         WHEN jsonb_typeof(a.value::jsonb) = 'number'
           THEN (a.value::jsonb)::text::double precision
         WHEN jsonb_typeof(a.value::jsonb) = 'boolean'
           THEN CASE WHEN (a.value::jsonb)::boolean THEN 1.0 ELSE 0.0 END
         WHEN lower(trim(a.value::jsonb #>> '{}')) = 'yes' THEN 1.0
         WHEN lower(trim(a.value::jsonb #>> '{}')) = 'no' THEN 0.0
         ELSE NULL
       END
     )
FROM assessments a
JOIN trace_info ti ON ti.request_id = a.trace_id
WHERE ti.experiment_id = ?
  AND ti.timestamp_ms >= ?
  AND ti.timestamp_ms < ?
  AND a.valid = true
GROUP BY a.name;
```

`aggregate_value` materializes that numeric aggregation path so the hot query becomes:

```sql
SELECT name, AVG(aggregate_value)
FROM assessments
WHERE experiment_id = ?
  AND trace_timestamp_ms >= ?
  AND trace_timestamp_ms < ?
  AND valid = true
  AND aggregate_value IS NOT NULL
GROUP BY name;
```

Every assessment insert or value update computes both fields together. `aggregate_value` is
populated for finite JSON numbers, booleans, and case-insensitive trimmed `"yes"` / `"no"`; other
values, including numeric strings such as `"0.8"`, produce null. `is_numeric_value` is true only for
finite JSON numbers. Aggregate metrics use every non-null `aggregate_value`, while numeric
comparison filters additionally require `is_numeric_value = true`. This preserves existing filter
semantics while removing runtime JSON conversion.

MLflow already knows the experiment from the owning trace when it writes the assessment, so storing
`experiment_id` does not require an extra integrity lookup on every insert. Skipping the foreign key
keeps this high-volume write path cheaper while preserving the same logical relationship.

After migration, MLflow should:

1. backfill `experiment_id` and `trace_timestamp_ms` from the owning trace rows
2. backfill `aggregate_value` from existing assessment values using the same numeric coercion rules
   that analytics readers use today
3. backfill `is_numeric_value`, then enforce `BOOLEAN NOT NULL DEFAULT false`
4. retarget assessment analytics to `assessments.experiment_id`, `assessments.trace_timestamp_ms`,
   and `assessments.aggregate_value`, and retarget numeric assessment filters to `aggregate_value`
   guarded by `is_numeric_value = true`
5. keep denormalized ownership fields synchronized when trace routing fields change:
   - on an experiment change, update `experiment_id` on spans and assessments and invalidate the
     affected old and new partitions
   - on a trace timestamp change, update `assessments.trace_timestamp_ms` and invalidate the
     affected old and new trace and assessment days

### 4. Query execution changes

The main read-path changes are:

- trace token metrics aggregate directly from `trace_info`
- trace-level cost metrics and gateway budget accounting read from `trace_info`
- span cost metrics aggregate directly from `spans`
- trace name and session count read from `trace_info`
- completed-session queries read `session_id` from `trace_info`
- API-facing trace metadata can synthesize `TRACE_SESSION` from `trace_info.session_id` for
  compatibility
- assessment analytics query `assessments` directly instead of joining from `trace_info`

For Postgres span analytics, use a trace-first query shape:

1. split filters into trace-level and span-level predicates
2. build a `metric_trace_ids` CTE from the filtered trace set and mark it `MATERIALIZED`
3. join spans from that materialized trace-id set

This keeps PostgreSQL on the measured trace-first plan instead of inlining back to a slower
span-first scan. This was discovered from a proof of concept implementation.

### 5. Opt-in SQL daily rollups

Denormalization removes the EAV joins from hot paths, but large dashboard windows still repeatedly
aggregate over millions of rows. This RFC proposes daily rollups for all supported SQL backends.
`MLFLOW_SQL_TRACE_ROLLUPS_ENABLED` defaults to `false`; deployments opt into the extra storage and
maintenance work by setting it to `true`.

#### Schema and supported aggregates

Add three daily rollup tables and one partition rebuild table. These definitions use standard
identity syntax; the Alembic migration should emit the equivalent identity definition for each
database dialect.

```sql
CREATE TABLE sql_trace_metric_daily_rollups (
  id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  experiment_id INTEGER NOT NULL,
  rollup_day DATE NOT NULL,
  metric_name VARCHAR(250) NOT NULL,
  grouping_set VARCHAR(50) NOT NULL,
  trace_status VARCHAR(50),
  sample_count BIGINT NOT NULL,
  sum_value FLOAT(53),
  min_value FLOAT(53),
  max_value FLOAT(53),
  p50_value FLOAT(53),
  p90_value FLOAT(53),
  p99_value FLOAT(53)
);

CREATE TABLE sql_span_cost_daily_rollups (
  id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  experiment_id INTEGER NOT NULL,
  rollup_day DATE NOT NULL,
  metric_name VARCHAR(250) NOT NULL,
  grouping_set VARCHAR(50) NOT NULL,
  model_name VARCHAR(500),
  model_provider VARCHAR(500),
  sample_count BIGINT NOT NULL,
  sum_value FLOAT(53),
  min_value FLOAT(53),
  max_value FLOAT(53)
);

CREATE TABLE sql_assessment_daily_rollups (
  id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  experiment_id INTEGER NOT NULL,
  rollup_day DATE NOT NULL,
  metric_name VARCHAR(250) NOT NULL,
  grouping_set VARCHAR(50) NOT NULL,
  sample_count BIGINT NOT NULL,
  sum_value FLOAT(53),
  min_value FLOAT(53),
  max_value FLOAT(53)
);

CREATE TABLE sql_trace_rollup_rebuild_queue (
  experiment_id INTEGER NOT NULL,
  rollup_day DATE NOT NULL,
  rollup_family VARCHAR(50) NOT NULL,
  PRIMARY KEY (experiment_id, rollup_day, rollup_family)
);
```

Rollup rows use a database-generated primary key and store `experiment_id`, `rollup_day`,
`metric_name`, and `grouping_set` as ordinary columns. Because experiment IDs are globally unique,
workspace ownership is inherited from the parent experiment rather than duplicated in every rollup
row. Readers validate requested experiment IDs against the active workspace before accessing
rollups, as they do for raw queries. `grouping_set` identifies the dimensions used to produce the
row. Its values are `global` and `status` for trace rollups; `global`, `model`, `provider`, and
`model_provider` for span-cost rollups; and `global` for assessment rollups. Null dimension values
are not included in grouped sets. `global` means the metric is aggregated across all rows for the
experiment and day without grouping by any optional dimension. Global rows are identified by
`grouping_set`, not by null dimension columns.

The trace metric table stores one row per grouping set. For trace metrics it stores `sample_count`,
`sum_value`, `min_value`, `max_value`, and daily `p50_value`, `p90_value`, and `p99_value`. Daily
percentiles are not composable into coarser buckets, so readers should use these percentile columns
only for single-experiment, complete UTC-day `P50`, `P90`, and `P99` requests. Daily `COUNT`,
`SUM`, `AVG`, `MIN`, and `MAX` rollups are intended to be backend-agnostic. Exact percentile
rollup construction is backend-specific; this RFC defines a PostgreSQL implementation. Unsupported
backends should leave `p50_value`, `p90_value`, and `p99_value` null by skipping percentile rollup
population. Those backends, along with multi-experiment, non-daily, or unbucketed percentile
requests, stay on the raw exact path.

The span-cost rollup table stores daily `input_cost`, `output_cost`, and `total_cost` aggregates by
model and provider grains. The assessment rollup table stores global daily assessment counts and
numeric aggregates over `aggregate_value`. It intentionally does not group by assessment name or
exact assessment value because both are user-controlled and can be effectively unbounded, causing
the rollup to approach the raw assessment table's cardinality. Queries using either assessment
dimension therefore stay on the denormalized raw path.

Trace names are also user-controlled and can approach one distinct value per trace. Trace rollups
therefore materialize only global and status grouping sets. Queries grouped by trace name use the
denormalized raw path rather than creating a potentially trace-sized rollup.

#### Query routing

The rollup reader should use these tables for eligible daily `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`
queries and supported daily trace metric percentiles. Rollup eligibility, fallback, and maintenance
behavior should be centralized behind backend-specific planning code rather than scattered across
individual query paths. Requests with arbitrary filters, exact assessment-value dimensions,
assessment-name dimensions, trace-name dimensions, unsupported dimensions or aggregations,
unsupported percentile values, unsupported percentile-rollup backends, or bucket widths other than
one UTC-aligned day should fall back to the raw query path. Distinct session counts also remain raw
because daily counts are not composable when a session spans multiple days or experiments.

The API supports only UTC-aligned buckets today, which is why UTC daily rollups can serve these
requests.

Split the requested range at UTC midnight boundaries. Use rollups only for full UTC days entirely
inside the requested range when matching rollup rows exist and no partition rebuild entry exists.
Query raw rows for the partial first and last days, any day listed for rebuild, and any day without
matching rollup rows. The resulting non-overlapping ranges are combined by summing counts and sums,
taking the minimum of minima and maximum of maxima, and computing `AVG = total_sum / total_count`.
If no complete covered day exists, the entire request stays raw.

#### Scheduling and eligibility

A scheduled server job fills the tables. Deployments should control it with
`MLFLOW_TRACE_ROLLUPS_SCHEDULE`, a five-field UTC cron expression defaulting to `0 2 * * *` (daily
at 02:00 UTC). Deployments may optionally cap the number of partitions rebuilt per run. Until
[RFC 0002](../0002-job-executor-plugins/0002-job-executor-plugins.md) provides multi-replica job
coordination, the schedule runs in exactly one worker; all application replicas still read rollups
and mark partitions for rebuild. Each pass should:

1. compute eligible `(experiment_id, rollup_day)` partitions from raw trace, span, and assessment
   rows
2. include only complete UTC days whose contributing traces are complete and inactive
3. rebuild partitions listed in `sql_trace_rollup_rebuild_queue` before processing newly eligible
   days
4. skip partitions whose rollup rows already exist and have no rebuild entry
5. atomically replace the rollup rows and delete the rebuild entry for each rebuilt partition

`rollup_day` is the UTC day of `trace_info.timestamp_ms` for trace metrics,
`spans.start_time_unix_nano` for span-cost metrics, and `assessments.trace_timestamp_ms` for
assessment metrics. Coverage and invalidation use the same family-specific mapping.

A trace with spans is inactive when every span has a non-null `end_time_unix_nano` and the latest
span end time is at least 24 hours before the job starts. A completed trace without spans uses
`trace_info.timestamp_ms` for the same check. For trace and assessment rollups, contributing traces
are those assigned to the trace day. For span-cost rollups, they are traces with spans assigned to
the span day. A typical chat reuses a session ID across turns but creates a new trace ID per turn;
explicitly managed or delayed OTLP traces can remain open longer.

The 24-hour rule controls rollup eligibility; it is not a write cutoff. MLflow continues to accept
later spans and inserts rebuild entries for every affected trace, span-cost, and assessment
partition in the same transaction.

#### Invalidation and rebuild

`sql_trace_rollup_rebuild_queue` is keyed by `(experiment_id, rollup_day, rollup_family)` and acts as
a durable rebuild queue: row presence means that the family and day require rebuilding. A reader
uses a family's rollups for a day only when matching rollup rows exist and no rebuild entry exists.

Any committed mutation that can change a valid aggregate, including backdated trace creation, trace
moves or deletes, reserved analytic-field updates, delayed spans, and assessment creates, updates,
or deletes, inserts a rebuild entry for every affected old and new family partition in the same
transaction. Readers use raw rows for those days until the scheduled job atomically replaces the
rollup rows and deletes the rebuild entry. The entry survives restarts and repairs deletes or moves
even when no source rows remain in an old partition. Bulk operations deduplicate affected partition
keys.

Running the job on one worker prevents concurrent rebuild jobs, but application writes can still
race with publication. Before scanning source rows, the rebuilder inserts the partition's rebuild
entry if needed and takes a transactional row lock on it, using the dialect's equivalent of
`SELECT FOR UPDATE`. A writer that affects the same partition upserts and locks that entry in the
transaction that changes the source rows. If the writer races with a rebuild, it waits and leaves a
rebuild entry after publication rather than allowing stale rollups to be treated as valid. Writers
that affect different partitions do not block each other.

The rollup tables need targeted indexes for lookup and dimension filtering. Span-cost builds use the
experiment/day covering index listed below.

### 6. Targeted indexes

No new broad `trace_info` covering index is proposed; see Alternatives considered. This RFC adds the
following targeted indexes. The definitions below show PostgreSQL syntax; other backends should use
equivalent portable indexes without unsupported `INCLUDE` or partial `WHERE` clauses. In practice,
MySQL, MSSQL, and SQLite should keep the same leading key order and drop unsupported covering or
partial-index syntax rather than treating these indexes as PostgreSQL-only objects.

```sql
-- Supports session filtering, session-count aggregation, and completed-session queries.
CREATE INDEX index_trace_info_experiment_id_session_id
    ON trace_info (experiment_id, session_id);

-- Drives assessment time-series queries by experiment and trace timestamp without joining
-- through trace_info.
CREATE INDEX idx_assessments_exp_trace_ts
    ON assessments (experiment_id, trace_timestamp_ms);

-- Covers assessment time-series queries grouped or filtered by assessment name.
CREATE INDEX idx_assessments_exp_trace_ts_name
    ON assessments (experiment_id, trace_timestamp_ms, name);

-- Supports valid assessment queries primarily filtered by experiment and name.
CREATE INDEX idx_assessments_exp_name_valid
    ON assessments (experiment_id, name, valid);

-- Supports trace-first raw span-cost fallbacks: first select trace ids from trace_info,
-- then index into cost-bearing spans by trace id and span start time.
CREATE INDEX idx_spans_cost_trace_time_cover
    ON spans (trace_id, start_time_unix_nano)
  INCLUDE (input_cost, output_cost, total_cost, model_name, model_provider)
  WHERE input_cost IS NOT NULL
     OR output_cost IS NOT NULL
     OR total_cost IS NOT NULL;

-- Supports daily span-cost rollup build and repair jobs, which scan cost-bearing spans by
-- experiment and day rather than by trace id.
CREATE INDEX idx_spans_cost_exp_time_cover
  ON spans (experiment_id, start_time_unix_nano)
  INCLUDE (input_cost, output_cost, total_cost, model_name, model_provider)
  WHERE input_cost IS NOT NULL
     OR output_cost IS NOT NULL
     OR total_cost IS NOT NULL;

-- Supports rollup lookup by experiment, day, metric, and grouping set.
CREATE INDEX idx_trace_rollups_lookup
  ON sql_trace_metric_daily_rollups
  (experiment_id, rollup_day, metric_name, grouping_set, trace_status);

CREATE INDEX idx_span_cost_rollups_lookup
  ON sql_span_cost_daily_rollups
  (experiment_id, rollup_day, metric_name, grouping_set, model_name, model_provider);

CREATE INDEX idx_assessment_rollups_lookup
  ON sql_assessment_daily_rollups
  (experiment_id, rollup_day, metric_name, grouping_set);
```

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
so exact-value distributions use raw PostgreSQL. No source-row cap was applied for this measurement.
These timings predate the dedicated model/provider columns; their effect is not included.

## Trade-offs

- Additional columns and targeted indexes still increase storage, even after cleanup removes the
  duplicated hot EAV rows and session metadata.
- Exact assessment-value distributions are not accelerated because their user-defined grouping
  dimension is not reliably bounded. Without a source-row guard, large ranges can require expensive
  raw aggregation; with one, the UI may omit global distribution summaries for those ranges.
- Retargeting reads and writes, cleaning up duplicated rows, and supporting downgrade reconstruction
  increase implementation and migration complexity.

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

1. Stop the old server so writes cannot occur during the schema transition.
2. Deploy the new version and run its Alembic migration. The migration adds the columns, performs
   the synchronous backfill, validates the result, deletes legacy duplicates, and drops
   `dimension_attributes` before the server starts. Before dropping that column, validate that it
   contains only model and provider keys with string or null values.
3. Optionally use a prepopulate utility before the upgrade to reduce migration time. The migration
   must validate and complete any prepopulated state rather than assume it is complete.
4. Enable rollups separately after migration and scheduler setup. Configure the UTC cron schedule
   for a low-traffic period and run it on one worker until
   [RFC 0002](../0002-job-executor-plugins/0002-job-executor-plugins.md) is implemented.

In the Alembic migration code, `batch_op` should be used only for SQLite compatibility paths.
PostgreSQL, MySQL, and MSSQL should use direct `ALTER TABLE` and `CREATE INDEX` statements on large
tables to avoid expensive table recreation, while still emitting each dialect's portable equivalent
of the indexes above. Use `BIGINT` for denormalized token-count columns and `Float(53)` or
`DOUBLE PRECISION` for denormalized cost and aggregate-value columns. Large legacy-row cleanup may
require normal autovacuum follow-up or `VACUUM (ANALYZE)`.

Downgrade procedure: stop all writers; reconstruct and validate reserved token metrics and
`TOKEN_USAGE` metadata, trace `COST` metadata, span cost metrics, trace-name tags, session metadata,
and model/provider `dimension_attributes`; then run the schema downgrade and deploy the old version.

## Open questions

### 1. Query-shape caps

Should the trace metrics API enforce server-side query-shape caps, such as maximum bucket count,
minimum bucket width, or limits on raw sample work for exact percentile queries? Existing response
caps limit returned points, but do not necessarily bound total scan or aggregation cost.

### 2. Exact assessment distribution guard

Should exact assessment-value distributions have an opt-in source-row guard? These queries remain
raw and cover the complete filtered experiment/time range independently of trace-table pagination.
One option is `MLFLOW_ASSESSMENT_DISTRIBUTION_MAX_ROWS`, unset by default to preserve existing
behavior. A positive value would cap filtered source rows before grouping. If that cap or the
unpaginated API group limit is exceeded, the distribution should be unavailable rather than
truncated or computed from only the loaded trace page. Result pagination could address the
group-response limit, but not the source-row work.
