# Hybrid PostgreSQL + Iceberg 10M / 30-Day Benchmark

Benchmark captured on 2026-07-17 against the populated
`mlflow-hybrid-10m-30d-iceberg` dataset.

## Dataset

- Experiment: `1`
- Total traces: `10,000,000`
- Hot PostgreSQL traces: `2,982,749`
- Archived Iceberg traces: `7,017,251`
- Inclusive source bounds: `1781292342638` through `1783884342286`
- Replay bounds: `1781292342637` through `1783884342287`
- PostgreSQL rollup rows: trace `210`, span `300`, assessment `20`
- Iceberg rollup coverage: 22 UTC days, 2026-06-12 through 2026-07-03
- PostgreSQL rollup coverage: 10 UTC days, 2026-07-03 through 2026-07-12
- All-time API trace count: `10,000,000`

The replay uses one millisecond of padding because trace search applies strict `>` and `<`
timestamp predicates.

## Source

- Repository: `/home/mprahl/git/mlflow`
- Branch: `trace-poc`
- Base revision: `b6171bfc66c46528d83c1fe747c7b17a789e4316`
- Server: one MLflow worker on `127.0.0.1:55180`
- PostgreSQL: `127.0.0.1:55432/mlflow`
- Iceberg warehouse: `s3://rhods-dsp-dev/mlflow-hybrid-10m-30d-iceberg/iceberg/warehouse`

The MLflow working tree contained the query and handoff optimizations under evaluation. The
benchmark reports set `sql_rollups_prepared` to `False` because the runner did not rebuild SQL
rollups during these two invocations. SQL rollups had already been populated and verified before
the server started, and the server ran with `MLFLOW_SQL_TRACE_ROLLUPS_ENABLED=true`.

## Results

| Mode | Total request duration | Slowest request | Approx. server CPU |
| --- | ---: | ---: | ---: |
| Process/S3 cold | 72,747.24 ms | 30,392.48 ms | 13 s |
| Immediate warm replay | 8,441.10 ms | 1,631.76 ms | 5 s |

- Warm speedup: `8.62x`
- Warm duration reduction: `88.4%`
- Requests per replay: `20`
- HTTP statuses: all `200`

Largest cold-to-warm reductions:

| Request | Cold | Warm | Reduction |
| --- | ---: | ---: | ---: |
| `traces_tab.assessment_distribution` | 30,392.48 ms | 1,261.31 ms | 29,131.17 ms |
| `overview.usage.latency_time_series` | 17,115.47 ms | 671.15 ms | 16,444.32 ms |
| `overview.usage.cost_breakdown_by_model` | 7,358.47 ms | 99.56 ms | 7,258.91 ms |
| `overview.quality.assessment_value_time_series` | 8,176.65 ms | 1,631.76 ms | 6,544.89 ms |
| `overview.usage.total_tokens_time_series` | 3,749.56 ms | 524.22 ms | 3,225.34 ms |
| `traces_tab.time_range_trace_count` | 1,594.02 ms | 23.20 ms | 1,570.82 ms |
