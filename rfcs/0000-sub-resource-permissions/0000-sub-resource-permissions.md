start_date:   2026-08-06
mlflow_issue: https://github.com/mlflow/mlflow/issues/11496
rfc_pr:

# Summary

MLflow 3.13's RBAC model recognizes a fixed set of resource types (`experiment`,
`registered_model`, `prompt`, `scorer`, `workspace`). Resources that are children
of a permissionable parent (e.g., runs under experiments, model versions under
registered models) are not independently permissionable — they always resolve to
their parent's permission level.

This RFC proposes making **all DB-indexed resource types** independently
permissionable, gated behind a configuration flag:

- **`permission_granularity = simplified`** (default): today's behavior. Only
  existing resource types are recognized. Sub-resources resolve to their parent.
- **`permission_granularity = fine_grained`** (opt-in): all resource types are
  recognized. Admins grant on each resource type independently.

The design is fully backward compatible: the default mode is identical to today.
Operators opt into fine-grained mode when they need per-resource-type control.

# Basic example

### Simplified mode (default — today's behavior)

```python
# EDIT on experiment = EDIT on everything (runs, traces, assessments, logged models)
client.add_role_permission(ds_role.id, "experiment", "*", "EDIT")
# Sub-resources are not independently permissionable. Identical to today.

# Sub-resource grants are accepted with a warning but have no effect:
client.add_role_permission(evaluator_role.id, "trace", "*", "READ")
# ⚠️ Warning: "Resource type 'trace' is only enforced in fine_grained mode.
#    Grant will be stored but has no effect until permission_granularity
#    is changed to 'fine_grained'."
# The grant is stored — ready for when the operator switches modes.
```

_A warning rather than an error allows admins to pre-create sub-resource grants
before switching modes — see [Adoption strategy](#adoption-strategy) for the
safe transition path._

### Fine-grained mode (opt-in)

```ini
[mlflow]
permission_granularity = fine_grained
```

```python
# Evaluator: can view experiments and write assessments, nothing else
client.add_role_permission(evaluator_role.id, "experiment", "*", "READ")
client.add_role_permission(evaluator_role.id, "assessment", "*", "EDIT")

# Data scientist: full access to everything
client.add_role_permission(ds_role.id, "experiment", "*", "EDIT")
client.add_role_permission(ds_role.id, "run", "*", "EDIT")
client.add_role_permission(ds_role.id, "trace", "*", "EDIT")
client.add_role_permission(ds_role.id, "assessment", "*", "EDIT")
client.add_role_permission(ds_role.id, "logged_model", "*", "EDIT")

# Pipeline: can log runs and traces, but only read assessments
client.add_role_permission(pipeline_role.id, "experiment", "*", "EDIT")
client.add_role_permission(pipeline_role.id, "run", "*", "EDIT")
client.add_role_permission(pipeline_role.id, "trace", "*", "EDIT")
client.add_role_permission(pipeline_role.id, "assessment", "*", "READ")
client.add_role_permission(pipeline_role.id, "logged_model", "*", "EDIT")
```

Effective permissions in fine-grained mode:

| User | experiment | run | trace | assessment | logged_model |
|------|-----------|-----|-------|------------|--------------|
| evaluator | READ | — | — | **EDIT** | — |
| data-scientist | EDIT | EDIT | EDIT | EDIT | EDIT |
| pipeline | EDIT | EDIT | EDIT | **READ** | EDIT |

_(— = NO_PERMISSIONS, no grant configured)_

## Motivation

RFC 0005 established workspace-scoped roles with permission grants on resource
types. Its `VALID_RESOURCE_TYPES` set covers only top-level resources —
sub-experiment resources (runs, traces, assessments, logged models) are not
recognized, and their validators resolve to the parent experiment's permission
level. This was a reasonable simplification for v1, but as MLflow's resource
model grows — particularly with traces and assessments for LLM evaluation — the
limitation creates real access control gaps.

### Use cases that cannot be expressed today

**1. Run access without experiment management (#11496)**

An organization uses MLflow Model Registry. Users need to create runs (to log
models for registration) but should NOT be able to create or manage experiments
— only the admin should control experiment lifecycle. Today, run permissions
depend entirely on experiment permissions: restricting experiment access to
NO_PERMISSIONS also blocks run creation, and granting experiment EDIT to enable
run creation also grants experiment management (create new experiments, rename,
delete).

What they want: `NO_PERMISSIONS` on experiments + `EDIT` on runs.

Today: impossible. The only workaround is granting experiment EDIT, which also
lets users create new experiments and get automatic MANAGE on them — the opposite
of what the admin intended.

**2. Assessment-only write access (validation boundary)**

An enterprise has a validation workflow where production pipelines log traces
and human evaluators annotate them with assessments. Evaluators need to view
everything and write assessments, but should NOT be able to create runs, modify
traces, or alter experiments.

What they want: `READ` on experiments/runs/traces + `EDIT` on assessments.

Today: `CreateAssessment` and `CreateRun` both require experiment EDIT — granting
one inevitably grants the other. There is no way to separate assessment-write
from run-write or trace-write. This is because `CreateAssessment` uses
`validate_can_update_trace_by_trace_id`, which resolves to the parent
experiment's permission — conflating assessment writes with trace updates.

**3. Model version lifecycle separation**

A platform team owns the model registry namespace (which models exist, their
descriptions, aliases) while data scientists push new versions. Today, granting
experiment EDIT to enable `CreateModelVersion` also grants the ability to rename
or delete the registered model itself.

What they want: `MANAGE` on `registered_model` for platform team +
`EDIT` on `model_version` for data scientists.

Today: `model_version` inherits from `registered_model` — you cannot separate
"push a new version" from "manage the model entry."

### Out of scope

- **Fine-grained auth for non-DB-indexed resources** (e.g., individual spans
  within a trace) — these are stored in blob/artifact storage, not queryable as
  independent DB entities.
- **Fail-closed enforcement for unregistered resource types** — today, routes
  without a registered validator are fail-open (any authenticated user can
  access). This proposal preserves that behavior.
- **Parent-scoped sub-resource grants** — in fine-grained mode, a grant like
  `(trace, *, EDIT)` applies to all traces in the workspace, not to traces within
  a specific parent resource. There is no way to express "EDIT on traces, but only
  within experiment X." This would require a richer `resource_pattern` syntax or
  condition-based matching, which is not addressed here.

## Detailed design

### New resource types

The following resources have their own APIs but currently inherit permissions from
a parent resource type. In fine-grained mode, each becomes independently
permissionable.

_Verified against MLflow 3.15.1 source (`mlflow/server/auth/__init__.py`)._

**Inherits from `experiment`:**

| Resource | DB table | API examples | Current resolution | Proposed resolution (fine-grained) |
|----------|----------|--------------|-------------------|-----------------------------------|
| `run` | `runs` | CreateRun, UpdateRun, DeleteRun, LogMetric, LogBatch | `_get_permission_from_run_id()` → experiment | `_get_permission_from_run_id()` → run |
| `trace` | `trace_info` | StartTrace, EndTrace, GetTrace, SearchTraces, SetTraceTag | `_get_permission_from_trace()` → experiment | `_get_permission_from_trace()` → trace |
| `assessment` | `assessments` | CreateAssessment, UpdateAssessment, DeleteAssessment | `validate_can_update_trace_by_trace_id` → experiment | `_get_permission_from_assessment()` → assessment |
| `logged_model` | `logged_models` | CreateLoggedModel, GetLoggedModel, DeleteLoggedModel | `_get_permission_from_model_id()` → experiment | `_get_permission_from_model_id()` → logged_model |
| `prompt_optimization_job` | `jobs` | CreatePromptOptimizationJob, GetPromptOptimizationJob | `_get_permission_from_prompt_optimization_job_id()` → experiment | `_get_permission_from_prompt_optimization_job_id()` → prompt_optimization_job |
| `review_queue` | `review_queues` | CreateReviewQueue, UpdateReviewQueue, AddItemsToReviewQueue | `_get_permission_from_review_queue_id()` → experiment | `_get_permission_from_review_queue_id()` → review_queue |
| `label_schema` | `label_schemas` | CreateLabelSchema, UpdateLabelSchema, DeleteLabelSchema | `_get_permission_from_label_schema_id()` → experiment | `_get_permission_from_label_schema_id()` → label_schema |

**Inherits from `registered_model`:**

| Resource | DB table | API examples | Current resolution | Proposed resolution (fine-grained) |
|----------|----------|--------------|-------------------|-----------------------------------|
| `model_version` | `model_versions` | CreateModelVersion, UpdateModelVersion, DeleteModelVersion | `_get_permission_from_model_version()` → registered_model | `_get_permission_from_model_version()` → model_version |

**Inherits from `scorer`:**

| Resource | DB table | API examples | Current resolution | Proposed resolution (fine-grained) |
|----------|----------|--------------|-------------------|-----------------------------------|
| `scorer_version` | `scorer_versions` | ListScorerVersions | `validate_can_read_scorer` → scorer | `_get_permission_from_scorer_version()` → scorer_version |

**Inherits from `gateway_endpoint`:**

| Resource | DB table | API examples | Current resolution | Proposed resolution (fine-grained) |
|----------|----------|--------------|-------------------|-----------------------------------|
| `gateway_endpoint_binding` | `gateway_endpoint_bindings` | CreateGatewayEndpointBinding, DeleteGatewayEndpointBinding | `validate_can_update_gateway_endpoint` → gateway_endpoint | `_get_permission_from_gateway_endpoint_binding()` → gateway_endpoint_binding |

### Configuration

```ini
[mlflow]
# "simplified" = today's resource types only (experiment, registered_model, prompt, scorer, workspace)
# "fine_grained" = adds run, trace, assessment, logged_model, model_version,
#                  prompt_optimization_job, review_queue, label_schema,
#                  scorer_version, gateway_endpoint_binding
permission_granularity = simplified   # default
```

**Resource type sets:**

```python
# Current VALID_RESOURCE_TYPES (MLflow 3.15.1)
SIMPLIFIED_RESOURCE_TYPES = frozenset({
    "experiment",
    "registered_model",
    "prompt",
    "scorer",
    "gateway_secret",
    "gateway_endpoint",
    "gateway_model_definition",
    "mcp_server",
    "workspace",
})

FINE_GRAINED_RESOURCE_TYPES = SIMPLIFIED_RESOURCE_TYPES | frozenset({
    "run",
    "trace",
    "assessment",
    "logged_model",
    "model_version",
    "prompt_optimization_job",
    "review_queue",
    "label_schema",
    "scorer_version",
    "gateway_endpoint_binding",
})
```

**Methodology:** This list was derived by identifying every `_get_permission_from_*`
function in `mlflow/server/auth/__init__.py` (3.15.1) that resolves to a different
`resource_type` than the resource being accessed. Each entry has its own DB table,
its own API surface, and currently inherits permissions from a parent.

When `simplified` is active, the `add_role_permission` API accepts sub-resource
types but emits a warning — the grants are stored but have no effect until
fine-grained mode is enabled:

```python
def _validate_resource_type(resource_type: str):
    if resource_type not in FINE_GRAINED_RESOURCE_TYPES:
        raise MlflowException(
            f"Invalid resource type '{resource_type}'.",
            INVALID_PARAMETER_VALUE,
        )
    if permission_granularity == "simplified" and resource_type not in SIMPLIFIED_RESOURCE_TYPES:
        logger.warning(
            f"Resource type '{resource_type}' is only enforced in fine_grained mode. "
            f"Grant will be stored but has no effect until permission_granularity is "
            f"changed to 'fine_grained'."
        )
```

### Workspace resolution for sub-resources

Sub-resources don't have their own workspace column — they are identified by
UUID (globally unique) rather than by name, so they don't need workspace-scoped
uniqueness constraints. When workspaces were introduced (migration
`1b5f0d9ad7c1`), the `workspace` column was added only to named resources where
uniqueness is workspace-scoped (`experiments`, `registered_models`,
`model_versions`, `secrets`, `endpoints`, `model_definitions`, `jobs`).
Sub-resources like `runs`, `trace_info`, `assessments`, and `logged_models` were
left without a workspace column because their identity is a UUID, not a name.

Each validator already resolves the workspace today by loading the parent
resource (e.g., trace → experiment → workspace). This resolution is unchanged
in fine-grained mode — the only difference is what `resource_type` is passed to
the permission check after workspace is resolved.

Note: `assessment` maps to `experiment` directly (not to `trace`). This enables
"write assessments without trace-write access" — evaluators can annotate existing
traces without being able to `SetTraceTag` or `DeleteTraceTag`.

### Resolution logic

A helper method resolves the effective resource type and ID based on the
configured mode:

```python
def resolve_resource(resource_type: str, resource_id: str, parent_type: str, parent_id: str):
    """Return the (resource_type, resource_id) to check permissions against.

    In simplified mode: returns the parent (today's behavior).
    In fine-grained mode: returns the sub-resource itself.
    """
    if permission_granularity == "simplified":
        return parent_type, parent_id
    else:
        return resource_type, resource_id
```

Each validator calls this helper with both its own type and its parent's:

```python
def _get_permission_from_trace_request_id() -> Permission:
    trace = get_trace(request_id)
    experiment = get_experiment(trace.experiment_id)
    workspace = experiment.workspace

    check_type, check_id = resolve_resource(
        resource_type="trace",
        resource_id=trace.request_id,
        parent_type="experiment",
        parent_id=trace.experiment_id,
    )
    return store.get_role_permission_for_resource(user_id, check_type, check_id, workspace)
```

- **Simplified mode:** `resolve_resource` returns `("experiment", experiment_id)` — identical to today.
- **Fine-grained mode:** `resolve_resource` returns `("trace", trace_id)` — checks for a direct trace grant.

### Workspace MANAGE interaction

Workspace MANAGE (`(workspace, *, MANAGE)`) continues to fold into every
resource-type check, including sub-resources in both modes. A workspace admin has
unrestricted access to all resources in the workspace. This is unchanged.

### Search filtering

Search filtering follows the same pattern as today's `filter_search_experiments`:

```python
def filter_search_traces(user, traces, workspace):
    return [t for t in traces if resolve_permission(user_id, "trace", t.id, workspace).can_read]
```

In simplified mode, this resolves to experiment-level permission (existing
behavior). In fine-grained mode, it checks for trace-level grants directly.

### Extensibility

Any future resource type indexed in the DB can be added with:
1. One constant in `FINE_GRAINED_RESOURCE_TYPES`
2. A validator that resolves workspace (via parent lookup) and passes its own resource type

No other changes to the resolution engine, API surface, or DB schema.

### Performance

**Simplified mode:** Identical to today. No new code paths execute.

**Fine-grained mode:** One direct lookup per resource type in the already-loaded
role permissions (in-memory). No additional DB queries — role permissions for the
workspace are loaded once regardless of how many resource types are checked.

## Drawbacks

- **Fine-grained mode requires more grants per role.** Admins must explicitly
  grant on every sub-resource type a role needs. A role that previously needed one
  experiment grant now needs up to 5+. Mitigated: this is opt-in — only
  deployments that need fine-grained control enable it.

- **Admins must choose between two permission models.** The deployment has either
  simplified or fine-grained mode — admins need to understand which is active and
  how grants behave in that mode. Mitigated: the default is simplified (today's
  behavior), so admins who don't need fine-grained control never encounter the
  choice.

- **Migrating between modes requires temporarily over-scoping.** When switching
  from simplified to fine-grained, admins must pre-create sub-resource grants
  before flipping the flag. If an admin switches the flag without pre-creating
  grants, users lose access until grants are added. Mitigated: the recommended
  transition path (create grants first, then switch) avoids access disruption.

# Alternatives

### A. Inheritance-based permission model

Add an `inherit` flag to each grant, allowing some grants to flow to children
and others not.

```python
# inherit=True (default, backward compatible): EDIT flows to runs, traces, assessments, logged_models
client.add_role_permission(role.id, "experiment", "*", "EDIT", inherit=True)

# inherit=False: EDIT applies only to experiment metadata, children get nothing
client.add_role_permission(role.id, "experiment", "*", "EDIT", inherit=False)
# Must separately grant on children:
client.add_role_permission(role.id, "trace", "*", "READ")
```

**Rejected because:** creates conflicts (what if one grant says inherit=True and
another says inherit=False on the same resource type?). Complicates the mental
model — admins must understand per-grant inheritance semantics. Note: `inherit=True`
would be the default to ensure backward compatibility, but the per-grant flag
still introduces ambiguity when multiple grants on the same resource type
disagree.

### B. Capability-based permission levels

Split EDIT into `EDIT_RUNS`, `EDIT_TRACES`, etc.

**Rejected because:** breaks the scalar hierarchy that `max_permission()` depends
on. Contradicts the simplification direction of RFC 0005.

### C. Action-based permissions

Grant specific actions (`create_trace`, `delete_run`).

**Rejected because:** contradicts RFC 0005's design choice of permission levels
over per-action grants. Creates N² explosion. Unmanageable admin UX.

# Adoption strategy

**This is not a breaking change.** The default mode (`simplified`) is identical
to today's behavior.

**Adoption path for operators who want fine-grained control:**

1. Upgrade to the version shipping this change (no behavior change)
2. Audit existing roles and determine which sub-resources each role should access
3. Add explicit sub-resource grants to each role (can be done while still in
   simplified mode — the grants are stored but unused until mode switches)
4. Switch to fine-grained mode:
   ```ini
   permission_granularity = fine_grained
   ```
5. Verify access is as expected
6. Adjust parent grants as needed (e.g., downgrade `experiment` from EDIT to READ
   now that sub-resources are independently granted)

**Reverting to simplified mode:**

1. Ensure parent-level grants (experiment, registered_model) cover the desired
   access — in simplified mode, sub-resource grants are ignored and all access
   derives from the parent
2. Switch back to simplified:
   ```ini
   permission_granularity = simplified
   ```
3. Sub-resource grants remain stored but have no effect — they can be cleaned up
   or left in place for a future switch back

# Open questions

1. **How does this interact with a pluggable auth backend?** An
   `OPERATION_REGISTRY` approach would emit
   `AuthorizationRequirement(resource_type="trace", ...)` for trace operations.
   The `DefaultDbAuthorizationBackend` would implement `resolve_resource` with
   the mode-dependent logic. The designs are complementary.

2. **Should the admin UI show which mode is active and the effective permissions
   per resource type?** Proposed: yes. In fine-grained mode, the UI should
   clearly show which resource types a role has grants on and which are ungranted.

3. **Should new resource types added in future releases default to simplified or
   fine-grained behavior?** Proposed: new sub-resource types follow the same
   pattern — they are added to `FINE_GRAINED_RESOURCE_TYPES` and resolve to their
   parent in simplified mode.
