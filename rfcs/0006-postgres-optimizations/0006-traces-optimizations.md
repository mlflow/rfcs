---
start_date: 2026-06-26
mlflow_issue: # TBD — will file enhancement issue in mlflow/mlflow
rfc_pr: # leave empty until PR is created
---

# PostgreSQL Trace Query Optimization

## Summary

Optimize PostgreSQL query performance for MLflow's trace analytics (Usage page and Traces page) through indexes and schema denormalization.

A load test with 10M traces (in a single experiment) and 3 assessments per trace showed that the Usage page takes ~3 minutes to load and the Traces page aborts most metric queries due to browser timeouts. Under the same dataset, a columnar data store completed the Usage page in ~23 seconds and the Traces page in ~5 seconds.

The optimizations proposed are theoretical based on the bottlenecks observed during load test. They will help reduce the gap - but the indexes has to be tested at scale.

## Basic example

Before (current state at 10M traces):
```
Usage page load: ~3 minutes (62s warm, 145s cold)
Top query (input_tokens daily SUM): 114.8s
```

## Motivation

MLflow's trace analytics queries degrade severely at scale. With 10M traces, 30M spans, and 30M assessments, the Usage page takes ~3 minutes to load and the Traces page aborts 6 out of 7 metric queries due to browser timeouts.

The root causes are:

1. **Missing indexes** — no composite index on `(experiment_id, timestamp_ms)` on the `trace_info` table forces full table scans for every time-bounded query

2. **EAV schema for metrics** — token counts stored as key-value pairs require a JOIN across 10M × 30M rows on a string column

**Who is affected:**
- Self-hosted MLflow users with PostgreSQL backends at scale (>1M traces)
- Enterprise deployments tracking LLM token usage and assessments
- Any deployment where the Usage page or Traces page is slow

### Out of scope

- Switching to a different database engine.
- Changes to the MLflow REST API contract.
- UI-level pagination or query-limiting changes.
- Approximate percentile functions (worthwhile but can be a follow-up RFC)
- Assessment score pre-computation (worthwhile but can be a follow-up RFC)

## Detailed design

### 1. Composite and covering indexes

**Problem:** Every trace query filters on `experiment_id` and
`timestamp_ms`, but there is no composite index covering both columns. The planner falls back to sequential scans. PostgreSQL reads every page of trace_info sequentially, evaluates experiment_id = 0 AND timestamp_ms BETWEEN ... per row, and discards rows outside the time range. With a composite index on (experiment_id, timestamp_ms), the planner can seek directly to the time range within that experiment and read only the matching rows.

**Solution:**

```sql
-- Primary filter path for all trace queries.
-- INCLUDE (request_id) stores the request_id value directly in the index so
-- that token aggregation JOINs (which need request_id to match against
-- trace_metrics) can be answered entirely from the index without going back
-- to the main table to look up each row — avoiding expensive random I/O.
CREATE INDEX CONCURRENTLY idx_trace_info_exp_ts_reqid
    ON trace_info (experiment_id, timestamp_ms DESC)
    INCLUDE (request_id);

-- assessments: replaces the existing index on (trace_id, created_timestamp)
-- with a covering version that adds name and value to the leaf pages. This
-- lets the aggregation query (GROUP BY name, AVG(value)) be served entirely
-- from the index without going back to the table for each row. Since
-- assessment values are short (numbers, booleans, short strings), the index
-- size increase could be modest.
CREATE INDEX CONCURRENTLY idx_assessments_traceid_created_ts
    ON assessments (trace_id, created_timestamp)
    INCLUDE (name, value);

-- Expression index for time-bucket GROUP BY
CREATE INDEX CONCURRENTLY idx_trace_info_exp_bucket
    ON trace_info (
        experiment_id,
        (floor(timestamp_ms / 86400000) * 86400000)
    );
```

**How `idx_trace_info_exp_bucket` works (in plain terms):**

Every dashboard query groups traces by day — "how many traces ran on Monday vs
Tuesday vs Wednesday." To do that, the query takes each trace's millisecond
timestamp and rounds it down to midnight of that day using
`floor(timestamp_ms / 86400000) * 86400000` (86,400,000 ms = 1 day).

Without this index, PostgreSQL has to:
1. Read every row in the table (10M rows)
2. Do the rounding math on each row
3. Group the results by the computed day

With this index, PostgreSQL pre-computes and stores the "which day does this
trace belong to" value at write time. At query time, it can jump straight to
"all traces in experiment 0, grouped by day" without reading the full table or
doing any math — the answers are already sorted and grouped in the index.


**Important caveat:** PostgreSQL only uses this index when the query's
expression is *textually identical* to what the index was built on. The MLflow
code already emits `floor(timestamp_ms / 86400000) * 86400000` consistently, so
this matches.

**Where it helps most:** Queries that only touch `trace_info` — like
`trace_count` daily (34s → ~1s) and latency percentiles daily (35s → ~5s, with
the sort being the remaining cost). For token queries, the dominant cost is the
JOIN to `trace_metrics`, not the GROUP BY — that's addressed by denormalization
in Section 2.

**Why these specific indexes:**

| Index | Eliminates | Expected improvement |
|-------|-----------|---------------------|
| `idx_trace_info_exp_ts_reqid` | Full table scan on trace_info + extra table lookups to get request_id for JOINs | 10M → ~300K rows (30-day window), all data needed for token JOINs served from index alone |
| `idx_assessments_traceid_created_ts` | Table lookups for `name` and `value` during assessment aggregation. Replaces the existing `(trace_id, created_timestamp)` index with a covering version — same key columns, but `name` and `value` are carried in the leaf pages so PostgreSQL never visits the heap. | Avoids random I/O back to table for each of 30M assessment rows |
| `idx_trace_info_exp_bucket` | Per-row floor() computation in GROUP BY | Pre-computed bucket values |

**Note:** The `trace_metrics` table already has a composite primary key on
`(request_id, key)` which provides an index for JOIN lookups. Since the hottest
token metrics are being denormalized off this table entirely, no additional
index on `trace_metrics` is proposed.


### 2. Denormalize frequently-accessed token metrics

**Problem:** Token counts (`input_tokens`, `output_tokens`, `total_tokens`) are
stored in the `trace_metrics` EAV table as key-value pairs. Aggregating them
requires a JOIN rows on a string key (`request_id`) filtered by
a string value (`key = 'input_tokens'`).

This is the one of the slow query pattern: **115 seconds** for a daily token SUM.

**Solution:** Add nullable columns directly on `trace_info` for the most
frequently-queried metrics:

```sql
ALTER TABLE trace_info
    ADD COLUMN input_tokens BIGINT,
    ADD COLUMN output_tokens BIGINT,
    ADD COLUMN total_tokens BIGINT;
```

**Write path changes:**

Since the migration runs during downtime, there is no dual-write period. After
migration, the tracking store writes token values only to the denormalized
columns on `trace_info`:

```python
# In SqlAlchemyTrackingStore._set_trace_metrics()
token_keys = {'input_tokens', 'output_tokens', 'total_tokens'}
token_updates = {k: v for k, v in metrics.items() if k in token_keys}
non_token_metrics = {k: v for k, v in metrics.items() if k not in token_keys}

# Token metrics go directly to trace_info columns
if token_updates:
    session.execute(
        update(SqlTraceInfo)
        .where(SqlTraceInfo.request_id == request_id)
        .values(**token_updates)
    )

# Other metrics still use the EAV table
for key, value in non_token_metrics.items():
    session.add(SqlTraceMetric(request_id=request_id, key=key, value=value))
```

**Read path changes:**

The metrics query builder reads token metrics directly from `trace_info` columns
instead of JOINing to `trace_metrics`:

```python
DENORMALIZED_METRICS = {'input_tokens', 'output_tokens', 'total_tokens'}

def build_metric_query(metric_key, ...):
    if metric_key in DENORMALIZED_METRICS:
        # Direct column access — no JOIN needed
        column = getattr(SqlTraceInfo, metric_key)
        return select(func.sum(column)).where(...)
    else:
        # Other metrics still use EAV path via trace_metrics JOIN
        return existing_join_query(metric_key, ...)
```

**Migration (runs during downtime):**

The Alembic migration adds the columns, backfills from `trace_metrics`, then
deletes the migrated token rows from `trace_metrics`:

```sql
-- 1. Add columns
ALTER TABLE trace_info
    ADD COLUMN input_tokens BIGINT,
    ADD COLUMN output_tokens BIGINT,
    ADD COLUMN total_tokens BIGINT;

-- 2. Backfill from trace_metrics
UPDATE trace_info ti
SET input_tokens = (
    SELECT CAST(tm.value AS BIGINT)
    FROM trace_metrics tm
    WHERE tm.request_id = ti.request_id AND tm.key = 'input_tokens'
),
output_tokens = (
    SELECT CAST(tm.value AS BIGINT)
    FROM trace_metrics tm
    WHERE tm.request_id = ti.request_id AND tm.key = 'output_tokens'
),
total_tokens = (
    SELECT CAST(tm.value AS BIGINT)
    FROM trace_metrics tm
    WHERE tm.request_id = ti.request_id AND tm.key = 'total_tokens'
);

-- 3. Remove migrated rows from trace_metrics
DELETE FROM trace_metrics
WHERE key IN ('input_tokens', 'output_tokens', 'total_tokens');
```

For large tables, steps 2 and 3 run in batches of 100K rows to avoid long-running
transactions. The migration assumes no traffic is served during execution.

**Expected performance improvement:**

| Query | Before (JOIN) | After (direct column) | Improvement |
|-------|--------------|----------------------|-------------|
| `input_tokens` SUM daily | 114.8s | ~2-3s | **~50x** |
| `output_tokens` SUM daily | 115.4s | ~2-3s | **~50x** |
| `total_tokens` SUM+AVG | 1.4s | ~200ms | **7x** |

**Trade-offs:**
- Only benefits the three promoted metrics; other trace_metrics still use EAV
- If new "hot" metrics emerge, requires another migration to denormalize them
- Migration requires downtime (estimate: ~30 min for 10M traces)

### 3. Add `experiment_id` to `assessments`

**Problem in plain terms:**

Today, the `assessments` table knows which *trace* each assessment belongs to
(via `trace_id`), but it doesn't know which *experiment* that trace is part of.
That information lives in a separate table (`trace_info`).

So every time the dashboard asks "show me average assessment scores for
experiment X in the last 30 days," PostgreSQL has to:

1. Go to `trace_info`, find all trace IDs for that experiment and time range
2. Take those IDs to the `assessments` table and look up matching rows
3. Then aggregate the results

This is like looking up a book by topic in a library where the topic catalog and
the actual books are in separate buildings — you have to visit both for every
query. With 30M assessments, this JOIN is the dominant cost in assessment
queries (105.8 seconds in our load test).

**Solution:** Add `experiment_id` directly to the `assessments` table. The table
already has `last_updated_timestamp` which serves as a time-range filter for
dashboard queries — no new timestamp column needed:

```sql
-- Add the column
ALTER TABLE assessments
    ADD COLUMN experiment_id INTEGER;

-- Index for experiment-scoped dashboard queries.
-- This lets PostgreSQL jump directly to "assessments in experiment 0,
-- updated in the last 30 days" without touching trace_info.
-- INCLUDE (name, value) carries the aggregation payload in the index so
-- the entire query (filter + group + aggregate) is served from a single
-- index scan.
CREATE INDEX CONCURRENTLY idx_assessments_exp_ts
    ON assessments (experiment_id, last_updated_timestamp)
    INCLUDE (name, value);
```

**How queries change:**

Before (requires JOIN):
```sql
SELECT assessments.name, avg(...)
FROM trace_info
JOIN assessments ON assessments.trace_id = trace_info.request_id
WHERE trace_info.experiment_id = 0
  AND trace_info.timestamp_ms BETWEEN ...
GROUP BY assessments.name
```

After (direct filter, no JOIN):
```sql
SELECT name, avg(...)
FROM assessments
WHERE experiment_id = 0
  AND last_updated_timestamp BETWEEN ...
GROUP BY name
```

Same result, but PostgreSQL reads a single index instead of joining two large
tables. The query planner goes directly to the right rows — no detour through
`trace_info`.

**Time semantics trade-off:**

The current query filters on `trace_info.timestamp_ms` (when the trace *ran*).

The new query filters on `assessments.last_updated_timestamp` (when the
assessment was last *scored*). These differ for retroactively-evaluated
assessments — e.g., a trace that ran on June 1st but was scored by a batch job
on June 15th.

In practice this is acceptable because:
- Assessments are almost always created/updated *after* their trace runs, so
  `last_updated_timestamp >= trace_info.timestamp_ms` holds in the vast majority
  of cases
- For dashboard aggregation, "recent assessment activity" is as useful a signal
  as "recent trace execution" — both answer "what do my scores look like lately?"
- When the user drills into a specific trace (trace explorer), more deep dive
  is needed to figure out how this edge case would be handled

So the separation is:
- **Dashboard (analytical):** `(experiment_id, last_updated_timestamp)` — fast,
  slightly different time semantics, no JOIN
- **Trace explorer (precise):** `(trace_id, created_timestamp)` — exact
  per-trace lookups via existing covering index

**No foreign key constraint on `experiment_id`:**

The `experiment_id` column is intentionally added *without* a foreign key
constraint to `experiments.experiment_id`. A FK would force PostgreSQL to take a
shared lock on the referenced `experiments` row and perform a lookup on every
assessment INSERT to verify the parent exists. At high write throughput this
adds measurable latency per insert and increases lock contention.

Since `experiment_id` is always derived from `trace_info.experiment_id` at write
time (which itself has referential integrity to `experiments`), the value is
guaranteed valid by construction — the FK check would be redundant verification
that slows down every write for no practical safety gain.

**Write path changes:**

When an assessment is created, the tracking store already has the `trace_id`
available. It looks up the trace's `experiment_id` from `trace_info` (typically
already in memory or a single-row lookup) and writes it alongside the
assessment:

```python
# In SqlAlchemyTrackingStore._create_assessment()
trace = session.get(SqlTraceInfo, trace_id)
assessment = SqlAssessment(
    trace_id=trace_id,
    experiment_id=trace.experiment_id,   # denormalized
    name=name,
    value=value,
    ...
)
session.add(assessment)
```

**Migration:**

```sql
-- Backfill from trace_info (batched, 100K rows at a time)
UPDATE assessments a
SET experiment_id = ti.experiment_id
FROM trace_info ti
WHERE a.trace_id = ti.request_id
  AND a.experiment_id IS NULL;
```

**Expected performance improvement:**

| Query | Before (JOIN) | After (direct) | Improvement |
|-------|--------------|----------------|-------------|
| `assessment_value` AVG/P90/P99 | 105.8s | ~8-10s | **~11x** |
| `assessment_count` by name+value | 60.5s | ~5-7s | **~9x** |

The remaining time is dominated by `percentile_cont` sorting, which this
change doesn't address (that's a follow-up for approximate percentiles).

**Why following indexes are needed:**

| Index | Serves |
|-------|--------|
| `(trace_id, created_timestamp) INCLUDE (name, value)` | Per-trace lookups: "get all assessments for trace X" (used by trace detail view, search filters, trace explorer) |
| `(experiment_id, last_updated_timestamp) INCLUDE (name, value)` | Experiment-scoped aggregation: "average scores across all traces in experiment X" (used by dashboard/Usage page) |

They cover different access patterns. The first is row-level (one trace), the
second is analytical (thousands of traces aggregated).


## Combined impact projection

With all three optimizations applied to the benchmark dataset (10M traces,
30M spans, 30M assessments):

| Query | Current | Projected | Factor |
|-------|---------|-----------|--------|
| `input_tokens` SUM daily | 114.8s | ~2s | **57x** |
| `output_tokens` SUM daily | 115.4s | ~2s | **58x** |
| `trace_count` daily | 34.1s | ~1s | **34x** |
| `latency` P50/P90/P99 daily | 34.9s | ~5s | **7x** |
| `assessment_value` AVG/P90/P99 | 105.8s | ~8s | **13x** |
| `assessment_count` by name+value | 60.5s | ~5s | **12x** |
| **Usage page total** | **~3 min** | **<10s** | **~18x** |

Note: Latency percentiles still retain O(n log n) sorting. A follow-up RFC
addressing approximate percentiles would further reduce these.

## Drawbacks

- **Increased storage:** Indexes add disk overhead at scale.
- **INSERT overhead:** More indexes = slower writes. For high-throughput trace ingestion this could be noticeable.
- **Migration requires downtime:** The backfill and cleanup of `trace_metrics` may take significant time. No traffic can be served during this window.

## Alternatives

### Alternative A: Switch to a columnar data store for analytics

Our benchmark shows columnar stores are 8-77x faster for these analytical
queries. However:
- Requires users to operate a second database
- Columnar stores are typically not ACID-compliant for the CRUD operations MLflow needs
- Significantly increases deployment complexity
- Not practical for small/medium deployments

**Verdict:** Best as a complementary option for large deployments, not a
replacement for PostgreSQL optimization.

### Alternative B: Materialized views for pre-aggregation

```sql
CREATE MATERIALIZED VIEW mv_daily_token_sums AS
SELECT experiment_id,
       floor(timestamp_ms / 86400000) * 86400000 AS day_bucket,
       key,
       sum(value::float) AS total
FROM trace_info ti
JOIN trace_metrics tm ON ti.request_id = tm.request_id
WHERE key IN ('input_tokens', 'output_tokens', 'total_tokens')
GROUP BY experiment_id, day_bucket, key;
```

Pros: Fast reads after refresh. Cons: Stale data between refreshes, operational
burden of scheduling `REFRESH MATERIALIZED VIEW CONCURRENTLY`, doesn't help
ad-hoc time ranges.

**Verdict:** Could complement denormalization but adds complexity. Not
recommended as primary solution.

### Alternative C: Do nothing, limit UI time range

Reduce the default query window from 30 days to 7 days. This helps (~4x fewer
rows) but doesn't solve the fundamental scaling problem — users with 7 days of
high-volume data will hit the same wall.

**Verdict:** Good short-term UX mitigation but not a substitute for schema
optimization.

## Adoption strategy

- **Non-breaking change:** All optimizations are additive schema changes (indexes, columns). No existing APIs or behaviors change.
- **Alembic migration:** Ships as a standard MLflow database migration. User run `mlflow db upgrade` as they do for any release.
- **Requires maintenance window:** The migration assumes no traffic is served during execution. Estimated duration: ~30 min for 10M traces (need to be vetted).
- **New installs:** Get the optimized schema from the start (denormalized columns and all indexes). Token metrics are written directly to `trace_info`, assessments include `experiment_id` and `timestamp_ms` from creation — no JOINs needed for dashboard queries from day one.

## Open questions

1. **Which additional metrics should be denormalized?** The RFC proposes three (`input_tokens`, `output_tokens`, `total_tokens`). Should `total_cost` or `latency`-related metrics also be promoted?

2. **Should the expression index for time buckets use a generated column instead?** PostgreSQL 12+ supports `GENERATED ALWAYS AS` columns, which might be cleaner than an expression index for the floor() computation.

3. **Approximate percentiles:** Should this RFC include switching to `percentile_disc` (or a t-digest extension) for dashboard queries, or defer to a follow-up RFC?

4. **Add `experiment_id` to `trace_metrics` too?** This RFC proposes adding `experiment_id` to `assessments` (Section 3) and denormalizing the three hottest metrics off `trace_metrics` entirely (Section 2). For the remaining
   long-tail metrics still in the EAV table, should `experiment_id` be added to `trace_metrics` as well? The trade-off is additional storage and a backfill migration, but it would let *any* metric query filter by experiment without JOINing to `trace_info`.
