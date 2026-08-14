# Hybrid PostgreSQL + Filesystem Iceberg 10M / 32-Day Benchmark

Benchmark captured on 2026-07-19 against the completed local-filesystem hybrid dataset.

## Dataset

- Experiment: `1`
- Total traces: `10,000,000`
- Hot PostgreSQL traces: `2,773,333`
- Archived Iceberg traces: `7,226,667`
- Inclusive source bounds: `1781707734967` through `1784299734615`
- Replay bounds: `1781707734966` through `1784299734616`
- Replay window label: 32 days
- All-time API trace count: `10,000,000`

The replay uses one millisecond of padding because trace search applies strict `>` and `<`
timestamp predicates.

## Source and Method

- MLflow branch: `trace-poc`
- MLflow revision: `c70c9b9c26eb19a2aa8377253ad352e79128258c`
- PostgreSQL: `127.0.0.1:55433/mlflow`
- Iceberg warehouse: `/home/mprahl/mlflow-hybrid-10m-30d-filesystem/iceberg`
- Archive payload root: `/home/mprahl/mlflow-hybrid-10m-30d-filesystem/trace-archive`
- Server: one MLflow worker on `127.0.0.1:55181`

The benchmark server was launched from a clean environment without
`MLFLOW_TRACE_ARCHIVAL_CONFIG`, and stale Huey consumers from data generation were stopped before
measurement. Hot/archive counts and SQL rollup row counts were identical before and after both
replays. SQL rollups were prebuilt before server startup; `sql_rollups_prepared` is `False` in the
raw reports because the replay command itself did not rebuild them.

The exact assessment-distribution safety cap was disabled so the workload could intentionally
query all 10 million traces. Production keeps that cap enabled by default.

## Results

| Mode | Total request duration | Slowest request | Approx. server CPU |
| --- | ---: | ---: | ---: |
| Fresh-process replay | 17,609.13 ms | 2,854.48 ms | 8 s |
| Immediate warm replay | 14,642.10 ms | 2,412.99 ms | 8 s |

- Immediate replay speedup: `1.20x`
- Immediate replay duration reduction: `16.8%`
- Requests per replay: `20`
- HTTP statuses: all `200`
