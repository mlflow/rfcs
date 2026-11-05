start_date: 2026-09-15
mlflow_issue: https://github.com/mlflow/mlflow/issues/25601
rfc_pr: https://github.com/mlflow/rfcs/pull/52

# Summary

Expose typed Python client and public REST APIs on `MlflowClient` to submit asynchronous Issue Detection jobs on traces and query detected issues by experiment and `source_run_id`.

This bridges MLflow's existing private UI endpoint (`POST /ajax-api/3.0/mlflow/issues/invoke`) into public version-three RPC routes (`/api/3.0/mlflow/issues/invoke` and `/api/3.0/mlflow/issues/search`), enabling automated CI/CD pipelines, scheduled jobs, and programmatic evaluation scripts.

---

# Basic Example

```python
import mlflow

client = mlflow.MlflowClient()

# 1. Submit asynchronous issue detection for selected traces
job = client.submit_issue_detection(
    experiment_id="42",
    trace_ids=["tr-019234a", "tr-019234b"],
    categories=["hallucination", "tool_call_error"],
    provider="openai",
    model="gpt-4o-mini",
)

print(f"Submitted job: {job.job_id} (tracking run: {job.run_id})")

# 2. Check execution status via the tracking run
run = client.get_run(job.run_id)
print(f"Job Status: {run.info.status}")  # RUNNING, FINISHED, or FAILED

# 3. Search detected issues for this run once finished
issues = client.search_issues(
    experiment_id="42",
    source_run_id=job.run_id,
    filter_string="status = 'pending'",
)

for issue in issues:
    print(f"[{issue.severity}] {issue.name}: {issue.description}")
```

---

# Motivation

MLflow Tracing allows developers to capture GenAI execution traces and evaluate them for quality issues. MLflow provides a UI-specific endpoint that detects issues across selected traces in the background and writes them to the tracking database.

However, typed Python SDK callers currently cannot programmatically invoke issue detection or query detected issues by `source_run_id`. Automated orchestration tools (e.g. nightly test suites, evaluation scripts, or Airflow pipelines) must either make raw, undocumented AJAX calls or inspect the SQLite database directly.

This proposal addresses [Issue #25601](https://github.com/mlflow/mlflow/issues/25601) by defining the public version-three Protobuf RPCs and typed `MlflowClient` methods, reusing the existing server launch, auth, and worker execution paths.

### Out of Scope
- **Databricks Tracking Store**: `DatabricksRestStore.submit_issue_detection()` explicitly raises `MlflowNotImplementedException`. Databricks support is out of scope for this OSS RFC.
- **Custom Issue Evaluators**: Custom evaluator plug-ins or new issue detection prompts are not part of this API bridge.

---

# Detailed Design

## 1. Routes to be Added

Two public version-three RPC endpoints will be added under `MlflowService`:

| Method | Path | Visibility | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/3.0/mlflow/issues/invoke` | `PUBLIC_UNDOCUMENTED` | Submits an asynchronous trace issue detection job. |
| `POST` | `/api/3.0/mlflow/issues/search` | `PUBLIC_UNDOCUMENTED` | Queries detected issues scoped to an experiment. |

---

## 2. Request & Response Schemas

### `SubmitIssueDetection`
Defined in `mlflow/protos/issues.proto`:

```protobuf
message SubmitIssueDetection {
  optional string experiment_id = 1 [(validate_required) = true];
  repeated string trace_ids = 2;
  repeated string categories = 3;
  optional string provider = 4;
  optional string model = 5;
  optional string secret_id = 6;
  optional string endpoint_name = 7;

  message Response {
    optional string job_id = 1;
    optional string run_id = 2;
  }
}
```

### `SearchIssues`
Defined in `mlflow/protos/issues.proto`:

```protobuf
message SearchIssues {
  optional string experiment_id = 1 [(validate_required) = true];
  optional string filter_string = 2;
  optional int32 max_results = 3;
  optional string page_token = 4;
  optional bool include_trace_count = 5;

  message Response {
    repeated Issue issues = 1;
    optional string next_page_token = 2;
  }
}
```

### Data Model & Types
- **`issue_id`**: String formatted as `iss-<uuid_hex>` (limit 36 characters).
- **`IssueStatus`**: `pending`, `rejected`, `resolved`.
- **`IssueSeverity`**: `not_an_issue`, `low`, `medium`, `high`.
- **Run Tag Constant**: `mlflow.issueDetection.jobId = "<job_id>"`.

---

## 3. Invocation Modes & Validation

Callers specify the execution LLM using one of two mutually exclusive modes:

1. **AI Gateway Mode**:
   - Caller supplies `endpoint_name`.
   - Routes through the server's AI Gateway endpoint (`gateway:/<endpoint_name>`).
   - `provider`, `model`, and `secret_id` must be omitted.

2. **Direct Provider Mode**:
   - Caller supplies `provider` (e.g. `openai`, `anthropic`, `bedrock`) and `model` (e.g. `gpt-4o-mini`).
   - Credentials are resolved from `secret_id` (decrypted via AI Gateway) or fallback server environment variables (e.g. `OPENAI_API_KEY`).
   - `endpoint_name` must be omitted.

**Validation Rules**:
- `trace_ids` must not be empty.
- Either `endpoint_name` OR (`provider` AND `model`) must be provided. Conflicting combinations (e.g. `endpoint_name + provider` or `endpoint_name + secret_id`) are rejected with `INVALID_PARAMETER_VALUE`.

---

## 4. Server Execution & Authorization Contract

### Authorization Contract
The route is registered in `mlflow/server/auth` using `validate_can_invoke_issue_detection()`:
1. **Experiment Permission**: Requires `UPDATE` permission on `experiment_id`.
2. **Secret Permission**: If `secret_id` is supplied, checks `_get_gateway_secret_permission(secret_id).can_use` to ensure the caller is authorized to decrypt the secret.
3. **Cross-Experiment Trace Protection**: The handler Derives the parent experiment for each supplied `trace_id` (reusing the `validate_can_batch_get_traces` pattern) and requires `READ` permission on all of them, preventing callers from running analysis on foreign traces.

### Execution Flow & Failure Handling
```text
1. Validate request body (experiment_id, trace_ids, invocation mode).
2. Authorize experiment UPDATE, secret USE, and trace READ.
3. Start tracking Run in experiment with tags:
     mlflow.runType = "issue_detection"
     categories = "<comma_separated_categories>"
     model = "gateway:/..." or "<provider>:/<model>"
4. Submit asynchronous job via submit_job(function=invoke_issue_detection_job, ...).
5. Tag run with mlflow.issueDetection.jobId = job.job_id.
6. Return SubmitIssueDetection.Response { job_id, run_id }.
```

**Queue-Submission Failure Cleanup**:
If `submit_job()` raises an exception (e.g., job queue saturated), the handler catches the exception, terminates the newly created Run with status `FAILED` (matching the GenAI evaluation handler cleanup), and re-raises the error to avoid leaving orphaned `RUNNING` runs.

---

## 5. Query Grammar & Pagination Contract

### Search Scoping
`experiment_id` is **required** on `search_issues`. Unscoped search is rejected by the server authorization layer.

### Filter Grammar
Search queries parse through `SearchIssuesUtils`:
- **Allowed Filter Attributes**: `status` and `source_run_id` (e.g. `status = 'pending' AND source_run_id = 'run-123'`).
- **Operators**: `=` and `!=`.
- **Conjunctions**: `AND` only. Parentheses and unsupported attributes (e.g. `severity`) are rejected with `INVALID_PARAMETER_VALUE`.

### Pagination
Pagination uses token-encoded offset pagination:
- `page_token` encodes a numeric offset (`query.offset(...)`).
- Returns `next_page_token` when additional results exist.

---

## 6. Python Client SDK Interface

Exposed on `MlflowClient`:

```python
class MlflowClient:
    def submit_issue_detection(
        self,
        experiment_id: str,
        trace_ids: list[str],
        categories: list[str],
        *,
        endpoint_name: str | None = None,
        provider: str | None = None,
        model: str | None = None,
        secret_id: str | None = None,
    ) -> IssueDetectionJob:
        """
        Submit an asynchronous issue detection job on traces.

        Returns:
            IssueDetectionJob containing `job_id` and `run_id`.
        """

    def search_issues(
        self,
        experiment_id: str,
        *,
        filter_string: str | None = None,
        source_run_id: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
        include_trace_count: bool = False,
    ) -> PagedList[Issue]:
        """
        Search detected issues within an experiment.

        Returns:
            PagedList of Issue entities.
        """
```

### Store Layer Contract
`AbstractStore` defines concrete base methods that raise `MlflowNotImplementedException`:
- Implemented in `RestStore` targeting `/api/3.0/mlflow/issues/...`.
- `SqlAlchemyStore` and `DatabricksRestStore` raise `MlflowNotImplementedException`.

---

# Open Questions

### Job Lifecycle & Completion Semantics
The handler returns `IssueDetectionJob(job_id, run_id)`. The primary public lifecycle entity is the associated MLflow Run (`run_id`):
- Callers poll execution state using `client.get_run(run_id).info.status` (`RUNNING`, `FINISHED`, `FAILED`).
- Searching issues filtered by `source_run_id=job.run_id` distinguishes between a completed job with 0 issues vs. a failed job.

*Question for maintainers*: Is checking the tracking run status via `client.get_run(run_id)` sufficient for the v1 client API, or should we introduce a typed `client.get_issue_detection_job(job_id)` endpoint to query job queue status directly?
