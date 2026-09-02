---
start_date: 2026-03-20
mlflow_issue:
rfc_pr: https://github.com/mlflow/rfcs/pull/3
---

<!-- markdownlint-disable-file MD041 -->

| Author(s)                      | [Matthew Prahl](https://github.com/mprahl), [Humair Khan](https://github.com/humairAK) |
| :----------------------------- | :------------------------------------------------------------------------------------- |
| **Date Last Modified**         | 2026-04-22                                                                             |

<!-- markdownlint-disable MD025 -->

# Summary: Docker and Kubernetes Job Execution Plugins

The [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2) defines the plugin interface, framework lifecycle, routing model, the remote execution framework
contract, and the built-in `LocalJobExecutor`. That foundation is necessary, but it is not sufficient for isolated
execution. Custom scorers created with the `@scorer` decorator reconstruct Python source during deserialization and
require a real isolation boundary. This RFC introduces the first two remote executors, Docker and Kubernetes, plus the
shared container-runtime configuration they rely on.

Both backends implement the same `AbstractJobExecutor` interface introduced by the [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2). The framework continues to
own job state transitions, retries, locking, scheduling, routing, token generation, permission enforcement, and result
reporting. The remote executors contribute isolation and backend-specific control-plane operations. This RFC keeps
Docker and Kubernetes together because most of the implementation detail is shared.

## Motivation

The [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2) deliberately keeps the lightweight built-in path separate from isolated execution. That preserves the
current low-friction MLflow experience for declarative jobs, but it leaves one important problem unsolved: how custom
scorers run safely in OSS MLflow.

The remote executor design addresses that gap by providing first-party isolated backends that plug into the shared
framework contract. The framework already knows when a job includes custom scorer code, when an operator has enabled
that code path, and when a job is eligible for remote execution. The missing piece is a pair of first-party backend
implementations that can run the job in a separate container while using the shared framework contract defined in the
[framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2).

Docker and Kubernetes cover the two most important deployment shapes. Docker is the lower-friction option for single
hosts, remote Docker daemons, and Podman-compatible environments. Kubernetes is the cluster-native option for
operators who want namespace isolation, service accounts, quotas, and cleanup through Kubernetes primitives. Combining
both backends in one RFC keeps the shared remote contract in one place while still making the backend-specific trade-offs
explicit.

### Goals

1. Remote executors will provide the first first-party isolated execution path for custom scorers in OSS MLflow.
2. Both Docker and Kubernetes will implement the shared remote execution contract defined in the [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2).
3. Shared container configuration such as image selection, startup behavior, labels, and resource overrides will be
   defined once and used by both backends.
4. Docker will support local daemons, Podman-compatible sockets, and remote Docker hosts.
5. Kubernetes will support namespace selection, service account configuration, and automatic cleanup of finished jobs.

### Out of scope

1. This RFC does not redefine the core framework, router, or lock semantics. Those are introduced by the [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2).
2. This RFC does not define a long-running guardrail worker model for the MLflow AI Gateway. The container isolation
   primitives should be reusable there, but the guardrail framework definition is out of scope because that is a
   different latency and lifecycle problem.

## Detailed design

The [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2) already defines the shared remote execution contract:
token generation, permission derivation including the union of inferred resources with scorer-declared
`required_resources`, API enforcement, result reporting, recovery semantics, the Gateway-only rule for remote LLM
access, and the submission-time refactor for prompt optimization resource resolution. This RFC focuses only on how
Docker and Kubernetes implement that contract and provide the required isolation boundary.

At the plugin-interface level, both executors advertise `remote_execution = True`. That allows the framework router to
select them for remote-eligible jobs, including custom-scorer jobs when admins have enabled that code path, while still
allowing operators to configure them as the default backend for other job types if desired.

### Shared container configuration

Docker and Kubernetes share the same basic container contract.

| Env var | Default | Purpose |
| --- | --- | --- |
| `MLFLOW_JOB_TRACKING_URI` | Required for remote execution | HTTP tracking URI reachable from the remote container |
| `MLFLOW_JOB_GATEWAY_URI` | Unset | HTTP base URI the remote runtime should use for MLflow AI Gateway traffic when needed |
| `MLFLOW_JOB_IMAGE` | `python:<server_python_version>-slim` | Base image used for remote job containers |
| `MLFLOW_JOB_SKIP_MLFLOW_INSTALL` | `false` | Skips `pip install mlflow==<server_version>` when the image already contains MLflow |
| `MLFLOW_JOB_EXTRA_LABELS` | Unset | JSON object of string key-value pairs applied to Docker containers or Kubernetes Jobs/Pods |

Image selection is shared:

1. If `MLFLOW_JOB_IMAGE` is set, use it.
2. Otherwise default to `python:<server_python_version>-slim`.
3. Docker and Kubernetes ignore `_PythonEnv.python` for runtime and image selection in this proposal. Each deployment
   pins one baseline Python image, while `_PythonEnv` remains relevant for install-related settings such as extra
   packages. Support for mapping multiple Python versions to distinct images can be added later if needed.

Container startup is also shared:

1. Install `mlflow==<server_version>` unless `MLFLOW_JOB_SKIP_MLFLOW_INSTALL=true`.
2. Install any job-specific Python dependencies from `_PythonEnv`.
3. Execute `python -m mlflow.server.jobs._job_subproc_entry`.

Resource defaults also use one precedence model across both backends:

1. Per-job-type environment variable override
2. `@job` decorator defaults
3. No explicit resource override

The exact mapping differs by backend. Kubernetes turns the values into pod requests and limits. Docker maps memory to
`mem_limit` and CPU to `nano_cpus`.

For remote executors, the framework populates `JobExecutionContext.tracking_uri` from `MLFLOW_JOB_TRACKING_URI` and,
when needed, `JobExecutionContext.gateway_uri` from `MLFLOW_JOB_GATEWAY_URI`. When `MLFLOW_JOB_GATEWAY_URI` is unset,
the tracking server URI is the default Gateway entrypoint for remote jobs.

The per-job-type override pattern is:

- `MLFLOW_JOB_RESOURCE_REQUESTS_<job_name>`
- `MLFLOW_JOB_RESOURCE_LIMITS_<job_name>`

These overrides use the same logical JSON object shape as the `@job` decorator metadata, for example
`{"cpu": "500m", "memory": "512Mi"}`. Each backend then maps those values to its own resource model.

Containers receive the `JobExecutionContext` fields and the serialized job params. They do not receive server-internal
environment variables or provider API keys.

For remote custom scorers, the framework-scoped token already reflects the union of inferred resources and any
scorer-declared `required_resources`. Docker and Kubernetes consume that token and the execution context; they do not
parse scorer code or infer permissions themselves.

Both backends also apply a small standard label set so recovery can find workloads reliably:

- `mlflow.job_id`
- `mlflow.job_name`

Additional label keys may be supplied through `MLFLOW_JOB_EXTRA_LABELS`.

If `_PythonEnv` is extended with package-install settings that embed credentials, those values are treated as sensitive
and should be delivered to the runtime through the same secure environment/Secret path as other sensitive
configuration.

### Docker executor

The Docker executor uses `docker-py` and is exposed through the optional `mlflow[docker]` extra. It is intended to
work with local Docker, Podman-compatible sockets, and remote Docker hosts.

#### Docker configuration

| Env var                 | Default        | Purpose                         |
| ----------------------- | -------------- | ------------------------------- |
| `MLFLOW_DOCKER_HOST`    | Docker default | Daemon URL for Docker or Podman |
| `MLFLOW_DOCKER_NETWORK` | `host`         | Container network mode          |

#### Docker lifecycle

- `start_executor()` validates connectivity with `client.ping()`.
- `submit_job()` resolves the image, builds the command and runtime environment variables, and starts a detached
  container with labels that include the MLflow job ID.
- `wait_for_job()` blocks on `container.wait()`, then reads the final job row. If the container has already exited but
  the job process never reported a result, the executor returns a container failure derived from the exit code. When
  possible, the executor should capture container logs before cleanup so startup failures and crash loops are
  diagnosable. After a terminal outcome has been observed, the container should then be removed by executor cleanup,
  including after successful completion.
- `cancel_job()` kills the container and performs the same cleanup path.
- `recover_jobs()` reconnects to labeled running containers after restart and lets the framework resume execution loops
  for those jobs by looking up containers associated with the unfinished MLflow job IDs. Running containers map to
  `reattach`. If the container has already exited, the executor should inspect any remaining terminal container state
  and the persisted job row; if there is still no reported result, it should return an infrastructure-level failure
  rather than silently treating the job as successful.
- `stop_executor()` is a no-op because Docker containers outlive the MLflow process.

#### Networking

The default Docker network mode is `host`, which keeps the initial configuration simple on Linux hosts. Operators can
override this with `MLFLOW_DOCKER_NETWORK` for bridge networks, Docker Compose environments, or remote daemons.

The important requirement is not the specific network mode. It is that `MLFLOW_JOB_TRACKING_URI` must resolve from the
container's network namespace. The executor should not try to guess that address.

For hardened deployments, the recommended setup is a dedicated operator-hardened Docker network for MLflow job
containers where the only intended outbound destinations are `MLFLOW_JOB_TRACKING_URI` and, when configured,
`MLFLOW_JOB_GATEWAY_URI`. In practice, that usually means attaching all job containers to one configured Docker network
and then using host-level firewall rules, an egress proxy, or equivalent infrastructure controls to enforce the
allowlist, because Docker networking alone does not provide a portable per-container egress policy.

The Docker executor should therefore treat restricted egress as an operator-provided deployment property rather than an
executor guarantee. Its responsibility is to attach the container to the configured network and require that the
tracking URI and optional Gateway URI be reachable from that network namespace.

### Kubernetes executor

The Kubernetes executor uses the official Kubernetes Python client and is exposed through the optional
`mlflow[kubernetes]` extra. It first tries in-cluster configuration and falls back to kubeconfig.

#### Kubernetes configuration

| Env var                              | Default           | Purpose                                         |
| ------------------------------------ | ----------------- | ----------------------------------------------- |
| `MLFLOW_K8S_NAMESPACE`               | `default`         | Namespace where Jobs are created                |
| `MLFLOW_K8S_USE_WORKSPACE_NAMESPACE` | `false`           | Uses the MLflow workspace name as the namespace |
| `MLFLOW_K8S_SERVICE_ACCOUNT`         | Namespace default | Service account for the Job pod                 |
| `MLFLOW_K8S_JOB_TTL`                 | `900`             | `ttlSecondsAfterFinished` for completed Jobs    |
| `MLFLOW_K8S_EXTRA_ANNOTATIONS`       | Unset             | Extra pod annotations                           |

On Kubernetes, `MLFLOW_JOB_EXTRA_LABELS` is applied as labels on both the Job and Pod, while
`MLFLOW_K8S_EXTRA_ANNOTATIONS` is applied as annotations on the Pod only.

#### Namespace selection

Namespace resolution is:

1. If `MLFLOW_K8S_USE_WORKSPACE_NAMESPACE=true` and the job has a workspace context, use the workspace name.
2. Otherwise use `MLFLOW_K8S_NAMESPACE`.

This makes workspace-to-namespace isolation possible without making it mandatory. When workspace namespaces are used,
the namespace must already exist and be reachable by the MLflow service account.

#### Networking and network policy

For hardened deployments, the recommended setup is a Kubernetes NetworkPolicy for these job pods that allows egress
only to `MLFLOW_JOB_TRACKING_URI` and, when configured, `MLFLOW_JOB_GATEWAY_URI`, plus any required cluster services
such as DNS. As with Docker, this restricted egress posture is an operator-provided deployment property rather than an
executor guarantee. The executor's responsibility is to launch the Job into a namespace where the tracking URI and
optional Gateway URI are reachable under that policy.

#### Job and Secret lifecycle

Kubernetes needs one extra step that Docker does not: sensitive environment variables should not be placed directly on
the Job spec.

Submission therefore creates resources in this order:

1. Create the Kubernetes Job with the non-sensitive environment, labels, resource settings, timeout, service account
   configuration, and a deterministic Secret reference name derived from the MLflow job ID.
2. Read the created Job UID from the API response.
3. Create a Secret with that deterministic name containing sensitive values such as `MLFLOW_JOB_TOKEN` and any package
   index credentials, with an `ownerReference` to that Job so Kubernetes garbage collection removes it with the Job.

This ordering intentionally favors narrower RBAC over eliminating every startup race. The MLflow service account only
needs to create Secrets, rather than patch them later, and it does not need update or patch access on Jobs. The
tradeoff is that the Job may briefly exist before the Secret does, so the pod may remain pending or fail to start until
the Secret is created.

If Secret creation fails after Job creation, the executor should best-effort delete the Job during submission error
handling so that a non-runnable workload is not left behind.

#### Kubernetes lifecycle

- `start_executor()` loads Kubernetes config and validates API connectivity.
- `submit_job()` resolves the namespace, image, Secret name, and Job spec, sets `restartPolicy: Never` because the
  framework owns retries, and caches the Kubernetes identifier keyed by MLflow job ID for live wait/cancel operations
  in the current process.
- `wait_for_job()` watches the Kubernetes Job selected by the MLflow job ID label rather than busy-waiting. When the
  Job reaches a terminal condition, it reads the persisted result from the job row. If the pod has already terminated
  without reporting a result, the executor classifies infrastructure failure from pod termination reason where
  possible.
- `cancel_job()` deletes the Kubernetes Job with cascading cleanup.
- `recover_jobs()` reconnects to matching Kubernetes Jobs after restart using the MLflow job ID label to find the
  backend workload for each unfinished job.
- `stop_executor()` is a no-op because Kubernetes Jobs outlive the MLflow process.

#### TTL cleanup

Finished Kubernetes Jobs use `ttlSecondsAfterFinished`, defaulting to 15 minutes. The Job TTL handles pod cleanup. The
Secret owner reference ensures the Secret is collected with the Job. No extra MLflow-side garbage collector is needed.

### Backend comparison

| Backend    | Best fit                                          | Main operator burden                          | Main advantage                                          |
| ---------- | ------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------- |
| Docker     | Single host, Podman socket, remote Docker daemon  | Network reachability and image management     | Lowest-friction isolated backend                        |
| Kubernetes | Cluster deployments and multi-tenant environments | Namespace, RBAC, and Job/Secret configuration | Stronger integration with cluster isolation and cleanup |

### Why full Python containers

This RFC chooses Docker and Kubernetes because MLflow custom scorers need a real Python runtime, not just an isolated
evaluation environment. Restricted interpreter sandboxes can be a good fit for tightly controlled agent-authored logic,
but they are not a good fit for generic BYO scorer code that may import `mlflow` and other third-party libraries.

For MLflow's use case, a restricted interpreter would either break compatibility with normal Python scorer code or force
MLflow to predefine privileged host callbacks for scorer behavior, which moves sensitive logic back into the server's
trust boundary. Containers are therefore a better fit for this RFC: they preserve full Python compatibility, allow
dependencies to be preinstalled and cached in images, and map naturally to Kubernetes-style deployment and cleanup.

## Drawbacks

1. Remote execution adds operator burden. Users must provide a reachable `MLFLOW_JOB_TRACKING_URI`, container images,
   and either Docker/Podman or Kubernetes infrastructure.
2. The remote model is intentionally stricter than local execution. Remote jobs must use Gateway-backed LLM access and
   cannot assume direct provider credentials or runtime default-model fallbacks in the runtime environment.
3. Docker and Kubernetes each add deployment-specific failure modes such as image pull failures, namespace
   misconfiguration, or unreachable tracking URIs. Those failures are acceptable, but they must be surfaced clearly in
   executor error reporting.
4. Kubernetes in particular increases RBAC and API object surface area through Jobs, Secrets, service accounts, and
   namespace policy.

# Adoption strategy

This RFC is designed to layer on top of the [framework RFC PR #2](https://github.com/mlflow/rfcs/pull/2), but it may be reviewed in parallel when that helps make the
core abstractions clearer.

Adoption is opt-in:

1. Install the relevant extra, such as `mlflow[docker]` or `mlflow[kubernetes]`.
2. Configure `MLFLOW_JOB_TRACKING_URI` and the backend-specific environment.
3. Enable custom scorers with `MLFLOW_SERVER_ENABLE_CUSTOM_SCORERS=true` or the matching `mlflow server` CLI flag if
   you want this backend to run custom scorer code.
4. Optionally set `MLFLOW_JOB_CUSTOM_SCORER_EXECUTOR_BACKEND` to `docker` or `kubernetes` if you want custom scorers
   routed to a backend different from `MLFLOW_JOB_DEFAULT_EXECUTOR_BACKEND`.

The default path remains unchanged. Declarative jobs continue to use `local` unless an operator changes
`MLFLOW_JOB_DEFAULT_EXECUTOR_BACKEND`.

This means the remote executor rollout does not break existing users. It extends the execution model rather than
replacing the built-in path.
