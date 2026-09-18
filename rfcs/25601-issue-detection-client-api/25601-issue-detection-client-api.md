# RFC: Typed Client API to Submit Issue Detection for Traces

| start_date | 2026-09-18 |
| :--- | :--- |
| **mlflow_issue** | https://github.com/mlflow/mlflow/issues/25601 |
| **rfc_pr** | https://github.com/mlflow/rfcs/pull/52 |
| **Author(s)** | [Rohitkanithi](https://github.com/Rohitkanithi), [Adam Gurary](https://github.com/adamgurary), [Joshua Wong](https://github.com/joshuawong-db), [Haoji Tang](https://github.com/tanghaoji) |
| **Date Last Modified** | 2026-09-18 |

---

## Table of Contents
- [Summary](#summary)
- [Basic Example](#basic-example)
  - [1. Submitting Issue Detection](#1-submitting-issue-detection)
  - [2. Querying and Filtering Issues](#2-querying-and-filtering-issues)
- [Motivation](#motivation)
  - [Background: Issue Detection in MLflow](#background-issue-detection-in-mlflow)
  - [The Problem: UI-Coupled Architecture](#the-problem-ui-coupled-architecture)
  - [Target Use Cases](#target-use-cases)
  - [Out of Scope](#out-of-scope)
- [Detailed Design](#detailed-design)
  - [1. System Architecture & End-to-End Component Flow](#1-system-architecture--end-to-end-component-flow)
  - [2. Request & Execution Lifecycle](#2-request--execution-lifecycle)
  - [3. Protobuf Protocol Specification](#3-protobuf-protocol-specification)
  - [4. Server Handler & Security Architecture](#4-server-handler--security-architecture)
  - [5. Store Layer Contracts](#5-store-layer-contracts)
  - [6. Python Client SDK Interface (`MlflowClient`)](#6-python-client-sdk-interface-mlflowclient)
  - [7. Entity Model & Schema Definitions](#7-entity-model--schema-definitions)
- [Performance & Scalability](#performance--scalability)
- [Drawbacks](#drawbacks)
- [Alternatives Considered](#alternatives-considered)
- [Adoption Strategy](#adoption-strategy)
- [Open Questions](#open-questions)

---

## Summary

This RFC proposes adding a formal, typed public client API and standard Protobuf RPC endpoint to MLflow for submitting GenAI **Issue Detection** jobs against logged traces.

Specifically, this proposal:
1. Promotes the internal, UI-specific AJAX route (`/ajax-api/3.0/mlflow/issues/invoke`) into an official, versioned Protobuf RPC (`POST /api/2.0/mlflow/issues/invoke`).
2. Introduces `client.submit_issue_detection(...)` on `MlflowClient`, returning a typed `IssueDetectionJob` handle (`job_id`, `run_id`).
3. Exposes `client.search_issues(...)` on `MlflowClient`, enabling callers to programmatically query and filter detected issues by `experiment_id`, `source_run_id`, severity, or category.
4. Preserves server-side credential resolution—ensuring no raw provider API keys or bearer tokens are ever accepted from or transmitted across client networks.

---

## Basic Example

### 1. Submitting Issue Detection
```python
from mlflow import MlflowClient

client = MlflowClient()

# Submit an issue detection job on traces for an experiment
job = client.submit_issue_detection(
    experiment_id="101",
    trace_ids=["tr-7a8b9c", "tr-1d2e3f"],
    categories=["hallucination", "tool_error", "safety"],
    provider="openai",
    model="gpt-4o",
)

print(f"Submitted background job: {job.job_id}")
print(f"Tracking run ID: {job.run_id}")
```

### 2. Querying and Filtering Issues
```python
# Query high-severity issues identified during the detection run
issues = client.search_issues(
    experiment_id="101",
    source_run_id=job.run_id,
    filter_string="severity = 'high' AND status = 'pending'",
    max_results=50,
)

for issue in issues:
    print(f"Issue: {issue.name} [{issue.severity}]")
    print(f"Description: {issue.description}")
    print(f"Categories: {issue.categories}")
    print(f"Root Causes: {issue.root_causes}")
```

---

## Motivation

### Background: Issue Detection in MLflow
MLflow Tracing records detailed multi-step execution graphs (spans, inputs, outputs, exceptions) for LLM applications and autonomous agents. To assist teams in monitoring production agent quality, MLflow introduced **Issue Detection**: an automated analyzer that evaluates batches of traces, identifies recurring regressions (such as invalid tool schemas, hallucinated function arguments, policy violations, or rate limit bottlenecks), and stores structured `Issue` entities.

### The Problem: UI-Coupled Architecture
While the backend execution engine (`invoke_issue_detection_job`) exists on the server, its invocation was originally implemented as an internal AJAX route (`/ajax-api/3.0/mlflow/issues/invoke`) designed exclusively for button clicks in the React web frontend.

Consequently:
* **No Client SDK**: `MlflowClient` provides no public methods to trigger detection or retrieve persisted issues.
* **Automation Blocked**: Developers cannot invoke issue detection programmatically from Python test scripts, continuous integration (CI/CD) pipelines, or scheduled orchestration workflows.
* **Untyped RPC**: External callers have no formal Protocol Buffer schema or typed response structures.

### Target Use Cases
1. **CI/CD Quality Gates**:
   An automated test pipeline runs synthetic test conversations through an agent, records the traces, calls `client.submit_issue_detection(...)`, and fails the build if any `high`-severity issues are returned.
2. **Scheduled Production Monitoring**:
   An hourly or nightly batch job (e.g., Airflow or Databricks Workflows) queries the latest production traces, runs issue detection, and dispatches automated alerts to Slack or PagerDuty.
3. **Agent Reflection & Self-Correction**:
   Autonomous coding or research agents programmatically trigger trace scans over their own recent execution histories to isolate failing tool calls and adjust system prompts dynamically.

### Out of Scope
* Implementing new core detector algorithms or modifying clustering logic.
* UI component modifications in the trace explorer or labeling review app.

---

## Detailed Design

### 1. System Architecture & End-to-End Component Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 CLIENT LAYER                                    │
│                                                                                 │
│   Python User Script / CI-CD Runner / Automated Orchestrator (Airflow / DBX)    │
│                                                                                 │
│   mlflow.tracking.MlflowClient                                                  │
│     ├── client.submit_issue_detection(experiment_id, trace_ids, categories...)  │
│     └── client.search_issues(experiment_id, filter_string, source_run_id...)    │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ HTTP POST (Protobuf JSON / Binary)
                                         │ Path: /api/2.0/mlflow/issues/invoke
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           TRACKING SERVER GATEWAY                               │
│                                                                                 │
│   mlflow.server.handlers._invoke_issue_detection_handler                        │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │ 1. Authentication & Permission Check                                    │   │
│   │    validate_can_update_experiment(experiment_id)                        │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │ 2. Server-Side Credential & Endpoint Resolution                         │   │
│   │    - Resolve provider secrets via AI Gateway / Server Environment       │   │
│   │    - REJECT any raw client-passed API keys                              │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │ 3. Run Initialization & Job Launch                                      │   │
│   │    - Create tracking Run with status = RUNNING                          │   │
│   │    - Tag: mlflow.issue_detection.job_id = <job_id>                      │   │
│   │    - Dispatch background job: invoke_issue_detection_job(...)           │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│   Returns: SubmitIssueDetection.Response { job_id, run_id }                     │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         │ Asynchronous Thread / Task Queue
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        ASYNC BACKGROUND EXECUTION ENGINE                        │
│                                                                                 │
│   invoke_issue_detection_job(job_id, run_id, experiment_id, trace_ids...)       │
│                                                                                 │
│   ┌───────────────────────┐   ┌───────────────────────┐   ┌─────────────────┐   │
│   │ 1. Load Trace Spans   ├──►│ 2. Execute LLM Judges ├──►│ 3. Aggregate &  │   │
│   │    from Tracking Store│   │    via AI Gateway     │   │    Cluster Bugs │   │
│   └───────────────────────┘   └───────────────────────┘   └────────┬────────┘   │
└────────────────────────────────────────────────────────────────────┼────────────┘
                                                                     │
                                                                     │ Store Entities
                                                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             TRACKING STORE (DATABASE)                           │
│                                                                                 │
│   ┌─────────────────────────────────┐     ┌─────────────────────────────────┐   │
│   │ runs Table                      │     │ issues Table                    │   │
│   │ - run_id                        │     │ - issue_id (UUID)               │   │
│   │ - experiment_id                 │     │ - experiment_id                 │   │
│   │ - status: RUNNING -> FINISHED   │     │ - name, description             │   │
│   │ - tags: job_id, categories      │     │ - severity: LOW/MEDIUM/HIGH     │   │
│   └─────────────────────────────────┘     │ - status: PENDING/RESOLVED      │   │
│                                           │ - source_run_id (FK to runs)    │   │
│                                           │ - root_causes, categories       │   │
│                                           └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 2. Request & Execution Lifecycle

```
Client                     Tracking Server              Async Worker            Tracking DB
  │                               │                          │                       │
  │  POST /mlflow/issues/invoke   │                          │                       │
  ├──────────────────────────────►│                          │                       │
  │                               │                          │                       │
  │                               │  validate_permissions    │                       │
  │                               │  resolve_credentials     │                       │
  │                               │                          │                       │
  │                               │  create_run(RUNNING)     │                       │
  │                               ├─────────────────────────────────────────────────►│
  │                               │                          │                       │
  │                               │  launch_job()            │                       │
  │                               ├─────────────────────────►│                       │
  │                               │                          │                       │
  │  SubmitIssueDetection.Response│                          │                       │
  │  { job_id, run_id }           │                          │                       │
  │◄──────────────────────────────┤                          │                       │
  │                                                          │                       │
  │                                                          │  fetch_traces()       │
  │                                                          ├──────────────────────►│
  │                                                          │                       │
  │                                                          │  evaluate_issues()    │
  │                                                          │  (Calls LLM/Rules)    │
  │                                                          │                       │
  │                                                          │  save_issues()        │
  │                                                          ├──────────────────────►│
  │                                                          │  end_run(FINISHED)    │
  │                                                          ├──────────────────────►│
  │                                                                                  │
  │  client.search_issues(source_run_id=...)                                         │
  ├─────────────────────────────────────────────────────────────────────────────────►│
  │  Returns PagedList[Issue]                                                        │
  │◄─────────────────────────────────────────────────────────────────────────────────┤
```

---

### 3. Protobuf Protocol Specification

#### Message Definition (`mlflow/protos/issues.proto`)

```protobuf
syntax = "proto2";

package mlflow.issues;

import "databricks.proto";

option java_package = "org.mlflow.api.proto";
option py_generic_services = true;

// Request message for submitting an asynchronous issue detection job
message SubmitIssueDetection {
  // Experiment ID to which traces belong.
  optional string experiment_id = 1 [(validate_required) = true];

  // Specific list of trace IDs to evaluate.
  repeated string trace_ids = 2;

  // Categories of issues to inspect (e.g. "hallucination", "tool_error").
  repeated string categories = 3;

  // Optional LLM provider identifier (e.g. "openai", "anthropic", "bedrock").
  optional string provider = 4;

  // Optional model identifier (e.g. "gpt-4o", "claude-3-7-sonnet").
  optional string model = 5;

  // Optional secret ID pointing to server-managed credentials.
  optional string secret_id = 6;

  // Optional AI Gateway endpoint name.
  optional string endpoint_name = 7;

  message Response {
    // Unique identifier for the asynchronous background job.
    optional string job_id = 1;

    // The MLflow tracking run ID created to record this detection job.
    optional string run_id = 2;
  }
}
```

#### RPC Service Registration (`mlflow/protos/service.proto`)

```protobuf
service MlflowService {
  // Submit issue detection for traces
  rpc submitIssueDetection (mlflow.issues.SubmitIssueDetection) 
      returns (mlflow.issues.SubmitIssueDetection.Response) {
    option (rpc) = {
      endpoints: [
        {
          method: "POST"
          path: "/mlflow/issues/invoke"
          since: {
            major: 3
            minor: 0
          }
        }
      ]
      visibility: PUBLIC_UNDOCUMENTED
      rpc_doc_title: "Submit issue detection for traces"
    };
  }
}
```

---

### 4. Server Handler & Security Architecture

In `mlflow/server/handlers.py`:

```python
@catch_mlflow_exception
def _invoke_issue_detection_handler():
    request_message = _get_request_message(
        SubmitIssueDetection(),
        schema={
            "experiment_id": [_assert_required, _assert_string],
            "trace_ids": [_assert_required, _assert_array],
            "categories": [_assert_required, _assert_array],
            "provider": [_assert_string],
            "model": [_assert_string],
            "secret_id": [_assert_string],
            "endpoint_name": [_assert_string],
        },
    )
    
    # 1. Authorization: Verify caller has write permissions on the experiment
    validate_can_update_experiment(request_message.experiment_id)
    
    # 2. Server-side credential isolation:
    # No raw provider keys are accepted from client payload.
    # Provider credentials are resolved via AI Gateway or server environment.
    
    # 3. Create run & launch background job
    job = invoke_issue_detection_job(...)
    
    response_message = SubmitIssueDetection.Response(
        job_id=job.job_id, 
        run_id=run_id
    )
    return _wrap_response(response_message)
```

---

### 5. Store Layer Contracts

In `mlflow/store/tracking/abstract_store.py`:

```python
@abstractmethod
def submit_issue_detection(
    self,
    experiment_id: str,
    trace_ids: list[str],
    categories: list[str],
    *,
    provider: str | None = None,
    model: str | None = None,
    secret_id: str | None = None,
    endpoint_name: str | None = None,
) -> IssueDetectionJob:
    """Submit an asynchronous issue detection job against traces."""
    pass
```

#### Implementation Details across Backends:
* **`RestStore` & `DatabricksRestStore`**:
  Encodes `SubmitIssueDetection` protobuf message to JSON and executes `self._call_endpoint(SubmitIssueDetection, req_body, endpoint="/mlflow/issues/invoke")`.
* **`SqlAlchemyStore`**:
  Direct local store execution without a running server raises:
  ```python
  raise MlflowException(
      "Submitting issue detection is only supported against a remote MLflow tracking server "
      "or when running the MLflow server."
  )
  ```

---

### 6. Python Client SDK Interface (`MlflowClient`)

Two methods are added to `mlflow.tracking.client.MlflowClient`:

```python
class MlflowClient:
    def submit_issue_detection(
        self,
        experiment_id: str,
        trace_ids: list[str],
        categories: list[str],
        *,
        provider: str | None = None,
        model: str | None = None,
        secret_id: str | None = None,
        endpoint_name: str | None = None,
    ) -> IssueDetectionJob:
        """
        Submit an asynchronous issue detection job for traces.

        Args:
            experiment_id: ID of the experiment containing the traces.
            trace_ids: List of trace IDs to analyze.
            categories: Categories of issues to evaluate (e.g. 'hallucination').
            provider: Optional provider name ('openai', 'anthropic', 'bedrock').
            model: Optional model name ('gpt-4o', etc.).
            secret_id: Optional server secret ID for authentication.
            endpoint_name: Optional MLflow Gateway endpoint name.

        Returns:
            An IssueDetectionJob handle containing job_id and run_id.
        """
        return self._tracking_client.submit_issue_detection(
            experiment_id=experiment_id,
            trace_ids=trace_ids,
            categories=categories,
            provider=provider,
            model=model,
            secret_id=secret_id,
            endpoint_name=endpoint_name,
        )

    def search_issues(
        self,
        experiment_id: str | None = None,
        filter_string: str | None = None,
        max_results: int | None = None,
        page_token: str | None = None,
        source_run_id: str | None = None,
        include_trace_count: bool = False,
    ) -> PagedList[Issue]:
        """
        Search for issues matching the given filters.

        Args:
            experiment_id: Optional experiment ID to scope the search.
            filter_string: SQL-like filter expression (e.g. "severity = 'high'").
            max_results: Maximum number of issues to return (default 100).
            page_token: Pagination token.
            source_run_id: Filter by the run ID that discovered the issues.
            include_trace_count: Whether to compute affected trace counts.

        Returns:
            A PagedList of Issue objects.
        """
        return self._tracking_client.search_issues(
            experiment_id=experiment_id,
            filter_string=filter_string,
            max_results=max_results,
            page_token=page_token,
            source_run_id=source_run_id,
            include_trace_count=include_trace_count,
        )
```

---

### 7. Entity Model & Schema Definitions

Defined in `mlflow/entities/issue.py`:

```python
class IssueStatus(str, Enum):
    PENDING = "pending"
    REJECTED = "rejected"
    RESOLVED = "resolved"

class IssueSeverity(str, Enum):
    NOT_AN_ISSUE = "not_an_issue"
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

@dataclass
class Issue(_MlflowObject):
    issue_id: str
    experiment_id: str
    name: str
    description: str
    status: IssueStatus
    created_timestamp: int
    last_updated_timestamp: int
    severity: IssueSeverity | None = None
    root_causes: list[str] | None = None
    source_run_id: str | None = None
    categories: list[str] | None = None
    created_by: str | None = None
    trace_count: int | None = None

@dataclass
class IssueDetectionJob(_MlflowObject):
    job_id: str
    run_id: str
```

---

## Performance & Scalability

* **Decoupled Asynchronous Execution**:
  Because trace analysis involves calling LLM judges or pattern detectors across multiple traces, `submit_issue_detection` returns immediately with HTTP 200 and a lightweight `IssueDetectionJob` handle. The HTTP request never blocks waiting for LLM inference to complete.
* **Server-Side Threading**:
  Jobs execute inside the server's background thread/task runner. Resource limits and concurrency are governed by server configuration rather than client connection lifespan.
* **Paginated Queries**:
  `search_issues` supports cursor-based pagination (`page_token`, `max_results`) to ensure querying thousands of issues remains performant.

---

## Drawbacks

* Running issue detection requires an MLflow tracking server with background job execution capability. It cannot execute against a purely local embedded SQLite/filesystem store without a running server instance.

---

## Alternatives Considered

1. **Keep Untyped AJAX Route and Make Raw HTTP Calls**:
   * *Rejected*: MLflow's standard architecture requires typed Protobuf RPC definitions for schema stability, cross-language SDK support, and backwards compatibility.
2. **Client-Side Evaluation**:
   * *Rejected*: Forcing clients to download bulk trace data and distributing sensitive model provider API keys to every developer machine creates severe security and bandwidth overhead.

---

## Adoption Strategy

* **Backward Compatibility**: Fully backward-compatible. This change is purely additive. Existing UI routes continue functioning without modification.
* **Documentation**: Add examples to the MLflow Tracing and GenAI Evaluation documentation demonstrating automated issue detection in CI/CD pipelines.

---

## Open Questions

1. Should `submit_issue_detection` support an optional synchronous polling parameter (`wait: bool = False`) in this version?  
   *Recommendation*: Keep the initial version asynchronous-only to avoid connection timeout issues over large trace sets. A client helper like `client.wait_for_issue_detection(...)` can be introduced as a fast follow-up.
