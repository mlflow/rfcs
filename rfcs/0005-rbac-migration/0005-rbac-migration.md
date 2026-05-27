---
start_date: 2026-05-08
mlflow_issue:
rfc_pr:
---

<!-- markdownlint-disable-file MD041 -->

| Author(s)              | [Pat Sukprasert](https://github.com/PattaraS) |
| :--------------------- | :-------------------------------------------- |
| **Date Last Modified** | 2026-05-08                                    |

<!-- markdownlint-disable MD025 -->

# Summary: Role-Based Access Control for MLflow OSS

MLflow OSS today expresses authorization as per-user, per-resource permission
rows scattered across seven tables (one per resource type). There are no
roles, no groups, and no shared permission sets. Operators who want to grant
the same permission to multiple users do so one resource and one user at a
time. This RFC proposes replacing that model with a small role-based core:
**roles** carry permission rows, **users** are assigned to roles, and the
seven legacy tables collapse into a single `role_permissions` table.

The migration ships in two phases. Phase 1 added the role surface as a
parallel write path. Phase 2 (in progress) backfills the legacy tables into
`role_permissions` and removes all legacy permission endpoints — 25
per-resource methods plus 4 workspace-permission methods — outright at the
same upgrade. New `grant_user_permission` / `revoke_user_permission`
convenience APIs replace the per-resource calls for one-off direct grants
without a deprecation-warning surface. The legacy tables stay on disk as a
paused snapshot; operators can drop them manually with guidance in the
release notes. The end state is a single permission table and a coherent
grant API.

# Basic example

The shift in the user-facing flow is the main reason for this proposal, so we
ground the design in concrete scenarios that an operator faces today.

The admin UI surfaces two top-level entry points, both as modals with
pre-filled current state and a diff preview before submit:

- **Edit role** (from the Role detail page): sections for *Role details*,
  *Permissions*, *Assigned users*. Same shape as **Create role**.
- **Edit access** (from the User detail page): sections for *Role
  assignments*, *Direct permissions*, *Admin status*.

There is no separate "Assign user" or "Grant permission" button — both flows
live behind the unified modals.

### 1. Give Alice EDIT on experiment 42 (one-off)

**Today.** A single per-resource POST:

```http
POST /api/2.0/mlflow/experiments/permissions/create
{ "experiment_id": "42", "username": "alice", "permission": "EDIT" }
```

**Proposed.** Two paths, depending on whether the grant is genuinely one-off
or part of a recurring pattern.

*Direct permission* — closest 1-to-1 with the old call. New convenience
API:

```http
POST /api/3.0/mlflow/users/permissions/grant
{ "username": "alice", "resource_type": "experiment",
  "resource_id": "42", "permission": "EDIT" }
```

Internally backed by a synthetic per-user role (`__user_<id>__`); same
`role_permissions` storage, no special-casing in the resolver. Gated by
per-resource MANAGE (same authority model as the legacy endpoint, just
a different method name). UI: `/admin` → Users → Alice
→ **Edit access** → *Direct permissions* section → add `experiment:42 →
EDIT` → review → apply.

*Role-based* — better when the same grant will be reused.

```http
POST /api/3.0/mlflow/roles/create
  { "name": "exp-42-editor", "workspace": "ml-research" }
POST /api/3.0/mlflow/roles/permissions/add
  { "role_id": <id>, "resource_type": "experiment",
    "resource_pattern": "42", "permission": "EDIT" }
POST /api/3.0/mlflow/roles/assign
  { "username": "alice", "role_id": <id> }
```

UI: `/admin` → Roles → **Create role** → in the *Permissions* section add
`experiment:42 → EDIT`; in the *Assigned users* section add Alice → Create.

### 2. Give a team READ on every experiment in a workspace

**Today.** No good answer. Either set the workspace-level permission to
`READ` (a grab-bag that also covers registered models, gateway endpoints,
and scorers) or list every experiment and POST a permission row per (user ×
experiment), then redo it whenever a new experiment is created.

**Proposed.** A single role with a wildcard pattern. New experiments
inherit:

```http
POST /api/3.0/mlflow/roles/create
  { "name": "experiment-reader", "workspace": "ml-research" }
POST /api/3.0/mlflow/roles/permissions/add
  { "role_id": <id>, "resource_type": "experiment",
    "resource_pattern": "*", "permission": "READ" }
```

Then `roles/assign` for each team member (or add them all at once via the
modal). UI: Roles → **Create role** → `experiment` / `*` ("All experiments")
/ `READ`; *Assigned users* section → add the team → Create.

### 3. Make Alice a workspace admin

**Today.** A direct workspace-permission grant:

```http
POST /api/2.0/mlflow/workspaces/ml-research/permissions
{ "username": "alice", "permission": "MANAGE" }
```

The behavior of `MANAGE` here is opaque: depending on `default_permission`
configuration, it silently extended to every resource type.

**Proposed.** A default `admin` role is seeded on workspace creation. The
operator just assigns it:

```http
POST /api/3.0/mlflow/roles/assign
{ "username": "alice", "role_id": <admin-role-id> }
```

The role explicitly carries `(workspace, *, MANAGE)` — the same authority,
but the grant is named, listable, bulk-assignable, and that's the only thing
it confers. There is no implicit fan-out across resource types.

UI: Users → Alice → **Edit access** → *Role assignments* section → add
`ml-research/admin` → review → apply.

## Motivation

The legacy permission model has three structural problems.

First, there is no abstraction for "the same permission, applied to many
people." Every grant is a per-user-per-resource POST. Five users editing ten
experiments is fifty calls and fifty rows. Adding an eleventh experiment
requires fifty more calls. There is no built-in way to express "the
ml-research data scientists" as a unit.

Second, the storage is split across seven tables (`experiment_permissions`,
`registered_model_permissions`, the four `gateway_*_permissions` tables,
`scorer_permissions`, and `workspace_permissions`) with inconsistent
workspace-awareness. Some tables have a workspace column and some don't —
which itself reflects an asymmetry in resource identity (registered models
are keyed by `(name, workspace)`, experiments by a globally-unique id), but
the storage shape doesn't paper over the asymmetry, so each resolver path
has to handle it. Resolving "what can Alice do here?" requires walking
each table.

Third, the permission resolution logic carries cumulative complexity from
the per-table layout. Auth-server code that filters list / search results,
propagates resource deletes, and propagates resource renames has to fan out
across all seven shapes. The on-disk surface (~1000 LoC of per-resource
CRUD, ~16 REST endpoints, ~30 `AuthServiceClient` methods) is a function
of that fan-out.

Roles fix all three. A role is a named, reusable bundle of permissions.
Storage collapses to one `role_permissions` table indexed by role. Resolution
is "look up the user's roles, union the permissions, evaluate against the
request" — no per-table fan-out. (Role cardinality is typically small enough
that per-role access caching is tractable; not implemented here, but the
data shape supports it as a future optimisation.)

### Out of scope

- **Teams / groups.** Roles can be reused across users, but there is no
  group abstraction yet. We expect this to come later if demand emerges; the
  role surface is forward-compatible with adding `group_role_assignments`.
- **Per-user / per-team budget policies.** Global gateway budgets exist;
  per-identity budgets are tracked separately and don't depend on this
  migration.
- **Cross-workspace roles.** Roles are workspace-scoped. A user who needs
  the same set of permissions in two workspaces is assigned to two
  workspace-scoped roles. We considered global roles but it complicates
  authorization without obvious benefit at this scale.
- **A "permission viewer" UI for end users.** The current admin UI is for
  operators. End users don't yet have a "what can I see?" view; would be a
  follow-up if requested. The new
  [`check_user_permission` endpoint](#wire-surface) is the building block
  it would use.
- **Pluggable authorization resolver.** The resolver is SQL-store-only in
  this RFC. The interface is small enough that an external integration
  (e.g. Kubeflow's `mlflow-integration` resolving via Kubernetes
  SubjectAccessReview) could plug in later; flagged as a known
  extension point in the Wire surface section, not designed here.

## Detailed design

### Storage

```text
users  ─assigned─→  user_role_assignments  ─→  roles  ─have─→  role_permissions
                                                                      │
                                                                      └─→ (resource_type, resource_pattern, permission)
```

Three tables replace seven:

- `roles` — `(id, name, workspace, description)`. Workspace-scoped; a role
  named `editor` in workspace `foo` is a different row from `editor` in
  workspace `bar`.
- `role_permissions` — `(id, role_id, resource_type, resource_pattern,
  permission)`. The pattern is either a literal resource id or `*`
  (workspace-wide for that resource type). `resource_pattern` absorbs the
  identity asymmetry between resource types: a `registered_model` pattern
  matches by name within the role's workspace, while an `experiment`
  pattern matches by globally-unique id.
- `user_role_assignments` — `(id, user_id, role_id)`. A user can hold any
  number of roles in any number of workspaces.

The legacy tables remain on disk post-migration as a paused snapshot for
rollback safety. They are no longer read or written by the auth server,
but no automated migration drops them — operators who want to reclaim the
disk space drop them manually, with timing called out in the release notes.
We considered an automated graduation migration and decided the small
permanent overhead is preferable to a destructive automated step on a
feature this new.

#### Resource types

`VALID_RESOURCE_TYPES` enumerates the resource slots the resolver
recognises: `experiment`, `registered_model`, `prompt`, `scorer`,
`gateway_secret`, `gateway_endpoint`, `gateway_model_definition`, plus the
special `workspace` slot for workspace-wide grants. The `prompt` type is
new in this RFC; in the legacy permission model, prompts reused
`registered_model` endpoints, which made permissioning awkward (granting
`(registered_model, foo, READ)` on a prompt-typed name purely because the
wire surface was shared). The
backfill migration rewrites existing `(registered_model, <name>, *)` rows
to `(prompt, <name>, *)` for any name that the registry classifies as a
prompt.

**Discriminator at request time.** The OSS REST surface for prompts
shares the registered-model route prefix (`/api/2.0/mlflow/registered-models/*`),
so there is no prompt-only endpoint to map at the route level. The
authorization layer dispatches by querying the **persisted entity** for
the `mlflow.prompt.is_prompt` tag — never the inbound request body. The
body tag is only written at create time, and trusting a body tag at
dispatch would let a caller with `(prompt, foo, MANAGE)` spoof the
discriminator on a non-CREATE registered-model route (`delete`,
`update`, `set-tag`, …) and escalate into the registered-model
namespace. Persisted-state classification closes that path. CREATE goes
through `validate_can_create_registered_model` (workspace-level create
rights apply to both shapes), so the after-handler — not the validator
— is responsible for typing the granted permission on the newly created
entity.

**Migration is workspace-scoped.** A name `foo` may be a prompt in
workspace A and a registered model in workspace B. The classifier joins
`role_permissions` to `roles` to recover each row's workspace and
filters `registered_model_tags` by `(workspace, name)` — only rows
whose workspace classifies the name as a prompt flip. Wildcard rows
(`resource_pattern = '*'`) are left untouched: they cover all RMs in
the workspace and were never prompt-specific. When the registry tables
aren't reachable on the auth connection (split-DB deployment), the
migration is a documented no-op; operators run the equivalent UPDATE
by hand. Matches the same-DB precedent of the legacy-permissions
backfill migration.

### Per-user direct grants

Pre-RBAC, an operator could POST a per-resource permission for a single
user. We preserve that affordance through two related mechanisms.

Under the hood, each user has at most one `__user_<id>__` role per
workspace, owned by the auth system. Direct permissions are stored as
`role_permissions` rows under that synthetic role — no special-casing in
the resolver. The `__user_<id>__` name pattern is reserved by the auth
store: `create_role` and `update_role` reject it, so operators cannot
collide a regular role with a user's synthetic slot.

On the API surface, the new `grant_user_permission` /
`revoke_user_permission` convenience methods (gated by per-resource
MANAGE) are the typical write path; the admin UI's *Direct permissions*
section is the same operation through the UI. Internally both paths
write to the synthetic role. The legacy
`create_experiment_permission` / `update_*_permission` family also wrote
to this slot, but those methods are removed in Phase 2 — the convenience
APIs are their non-deprecated replacement.

The result: the same permission resolution path handles both "permission
via shared role" and "permission via direct grant." The difference is
purely authorial — a direct grant is one user's name on a permission, a
role is a named bundle that anyone can be assigned to.

### Workspace-level grants

The legacy `workspace_permissions` table held `READ` / `EDIT` / `MANAGE`
levels with implicit fan-out to every resource type. The new model collapses
this to a single workspace permission slot:
`(resource_type='workspace', resource_pattern='*', permission)` where
`permission` is `USE` (member: read everything in the workspace, create
experiments / registered models) or `MANAGE` (admin: full authority,
including role/user management within the workspace). The two-tier shape
reflects the natural USE-vs-MANAGE split for workspace-level grants;
deeper tiers (READ / EDIT at workspace scope) would proliferate without
clear operational value, and per-resource grants already cover the
intermediate levels where needed.

### Permission resolution

The resolver runs on every protected request:

1. **Admin bypass.** If `is_admin = true`, short-circuit to allow.
2. **Role-derived grant on the resource.** Look up the user's roles in the
   request's workspace; union their permissions; check for a row with
   `(resource_type, resource_id)` that grants the requested permission.
   Specific patterns win over `*`.
3. **Workspace-level grant.** If no resource-specific row matched, check for
   a `(workspace, *, …)` row that satisfies the requested permission.
4. **Server `default_permission`.** Configured fallback.

This is the same chain the legacy resolver used; the table topology is
different but the precedence rules are unchanged.

### Identity model

Three identity tiers are enforceable at the validator layer.

A **super admin** has `is_admin = true` on the user row. Super admins
short-circuit all permission checks in `_before_request` — they are the sole
bearer of system-wide operations such as deleting users and bulk operations.

A **workspace admin** holds `(workspace, *, MANAGE)` (via any role) in one
or more workspaces. They can manage roles, users, and direct grants within
those workspaces but cannot perform system-wide operations. The auth-server
helper `list_workspace_admin_workspaces(user_id)` returns the set of
workspaces in which a user holds workspace-admin authority; the validator
layer composes this with the request's workspace to authorize role and user
edits.

A **regular user** is any other authenticated identity. Their authorization
flows entirely through role-derived permissions.

### UI workspace context

This section describes the frontend routing detail that surfaces workspace
context in the admin UI; it is not part of the authorization model itself.

Workspaces are conveyed through the URL via the `?workspace=<name>` query
param. A frontend component `WorkspaceRouterSync` extracts the param on
every navigation and sets the global `activeWorkspace` state, which most
hooks and outgoing-link helpers consult. Routes are partitioned into
*workspace-aware* (the param is auto-injected) and *global* (the param is
stripped); `/admin` is a hybrid: `/admin` itself is global (cross-workspace
platform-admin view), and `/admin/ws?workspace=…` is the per-workspace
management view.

### Filter-and-search behavior

The list / search endpoints (`search_experiments`, `search_logged_models`,
`search_registered_models`, `search_model_versions`, plus the GraphQL
model-version filter) need to apply role-derived authorization to results.
Pre-RBAC, each path had its own inline filter logic that walked the
appropriate per-resource permissions table. We collapse them onto a shared
helper, `_role_based_read_predicate(username, resource_type)`, which builds
a read-permission callable from `list_role_grants_for_user_in_workspace`
once per request. The handler then filters its result set with the
predicate.

### Wire surface

Phase 2 removes the legacy per-resource and workspace permission surfaces
outright; there is no deprecation-warning window. basic-auth is still
marked experimental, so a clean cut is preferable to carrying 25 warning-
emitting methods through a release. A one-time backfill migration moves
all existing grant rows into the new `role_permissions` storage; after
the upgrade, callers use either the role API directly or the new
convenience APIs (below).

**Removed** (calls 404):

- `POST/GET/PATCH/DELETE` on `/mlflow/{experiments,registered-models,scorers}/permissions`
- `POST/GET/PATCH/DELETE` on `/mlflow/gateway/{secrets,endpoints,model-definitions}/permissions`
- `POST/GET/PATCH/DELETE` on `/mlflow/workspaces/<workspace>/permissions`
- The corresponding 25 per-resource `AuthServiceClient` methods
  (`create_experiment_permission`, `update_registered_model_permission`, etc.)
- The four `*_workspace_permission` `AuthServiceClient` methods

**New role API** (the primary path):

- `POST /mlflow/roles/create`, `GET /mlflow/roles/get`,
  `GET /mlflow/roles/list`, `PATCH /mlflow/roles/update`,
  `DELETE /mlflow/roles/delete`
- `POST /mlflow/roles/permissions/add`,
  `PATCH /mlflow/roles/permissions/update`,
  `POST /mlflow/roles/permissions/remove`,
  `GET /mlflow/roles/permissions/list`
- `POST /mlflow/roles/assign`, `POST /mlflow/roles/unassign`,
  `GET /mlflow/users/roles/list`, `GET /mlflow/roles/users/list`

Gated by workspace MANAGE or Platform Admin
(`validate_can_manage_roles`). Creating roles is an administrative action;
this is by design.

**New convenience APIs** (the read/write counterparts for one-user,
one-resource direct grants):

- `POST /mlflow/users/permissions/grant`
  `{ username, resource_type, resource_id, permission }`
- `POST /mlflow/users/permissions/revoke`
  `{ username, resource_type, resource_id }`
- `POST /mlflow/auth/check`
  `{ username, resource_type, resource_id }`
  `→ { allowed: bool, permission: str }`

Client methods: `grant_user_permission`, `revoke_user_permission`,
`check_user_permission` on `AuthServiceClient`.

`grant_user_permission` and `revoke_user_permission` are the
non-deprecated replacement for the legacy
`create_experiment_permission` / `update_*_permission` family. They
write to the user's `__user_<id>__` synthetic role under the hood — same
storage path the legacy methods used — and are gated by **per-resource
MANAGE** (same validator as the legacy endpoints), so a user with
`(experiment, 42, MANAGE)` can still delegate on experiment 42. This
preserves the per-resource owner-delegation workflow without carrying
a deprecation-warning surface.

The convenience APIs explicitly reject `resource_type = 'workspace'` at
both the handler and the store layer. Workspace-tier grants have their
own surface (`set_workspace_permission` / `delete_workspace_permission`)
and the `allowed` semantics on a workspace slot would be ambiguous (which
permission tier — USE or MANAGE — counts as allowed?). The handler-level
rejection is load-bearing because super-admins skip the
`validate_can_manage_resource` gate via `sender_is_admin()`; the
store-level rejection is defense in depth.

`check_user_permission` mirrors Kubernetes SubjectAccessReview: it
resolves through the same code path as the runtime authorization check
(`_get_*_permission` family), so the answer is guaranteed consistent
with what an actual request would see. Useful for admin-UI debugging
("why can't Bob delete experiment 42?"), for the UI to gate elements
before issuing the real request, and for automation that wants to
dry-run access decisions.

The `allowed` boolean tracks `permission.can_use` — i.e. the regular
access tier, matching the `_user_can_create_in_workspace` convention.
`READ`-only access yields `allowed = false`; `USE` / `EDIT` / `MANAGE`
yield `allowed = true`. Callers that need a different cut (e.g. "can
the user *delete*?") read the `permission` field directly.
Authorization for `check_user_permission` itself is scoped to the
resource's workspace: self-checks are always allowed; a non-admin,
non-self requester must hold workspace MANAGE in the workspace where
`(resource_type, resource_id)` lives. Cross-workspace probes by a
workspace-A admin against resources in workspace B are denied; unknown
resources also deny rather than leak a deterministic `allowed = false`
across all workspaces. This is fail-closed by design.

### Inspection / debugging workflow

For "why can't Bob delete experiment 42?" the workflow is two existing
calls:

1. `list_user_roles(username)` — returns every role assigned to the user,
   including the synthetic `__user_<id>__` role that holds direct
   grants. Shared-role grants and direct grants both surface in one
   call, as raw `(resource_type, resource_pattern, permission)` rows.
2. `check_user_permission(username, resource_type, resource_id)` —
   returns the effective permission after `max_permission` folds
   wildcard grants, specific grants, and direct grants together. Same
   code path as the runtime check, so the answer can't drift.

What is _not_ currently exposed: "enumerate every resource the user has
any access to". That would require resource enumeration across the
tracking, registry, and gateway stores; a future
`POST /mlflow/auth/list-permissions` could expose it if operator demand
emerges.

### API style: RPC

All new auth endpoints follow the RPC convention already established by
MLflow's auth surface (`POST /mlflow/users/create`,
`POST /mlflow/users/delete`, `POST /mlflow/experiments/create`, …). A
REST-shaped role API
(`PUT /mlflow/roles/{id}/users/{username}`, etc.) was considered and
deferred — mixing styles across the auth surface would be worse than the
current consistency. A future major-version revision could re-shape both
the user-management and role-management halves at once, but this RFC
keeps the established style.

### Extension point: resolver interface

Authorization resolution is SQL-store-only in this RFC. The relevant
interface is small — essentially `get_role_permission_for_resource(user,
resource_type, resource_id, workspace)` and
`list_accessible_workspace_names(username)` — and the obvious future
abstraction is a protocol that lets a deployment plug in an external
resolver. A concrete motivating case is Kubeflow's `mlflow-integration`,
which could answer authorization queries via Kubernetes
SubjectAccessReview rather than maintain a parallel grant store. Out of
scope for this RFC; called out so the SQL-store coupling is intentional
rather than accidental, and so a future extension RFC has a known
hook-in point.

## Drawbacks

**Breaking wire change.** All legacy permission endpoints — the 25
per-resource ones and the four workspace-permission ones — are removed
outright at the upgrade that lands the simplified RBAC model. There is no
deprecation-warning window. Any client that called
`POST /experiments/permissions/create` or a sibling needs to be rewritten
to either the role API (for reusable grants) or the new convenience APIs
(`grant_user_permission` / `revoke_user_permission`, for one-off direct
grants) before upgrading. basic-auth is still experimental, so the
breakage cost is acceptable; the alternative — carrying 25
warning-emitting methods through a release — has a higher long-term
maintenance cost.

**Two grant shapes for one logical operation.** "Grant Alice EDIT on
experiment 42" can land as a direct permission or as a role. The two have
the same effect at the resolver but different operational semantics — a
role is reusable and listable; a direct permission is convenient and
one-off. Operators have to make a choice each time. We mitigate this with
documentation: prefer roles, use direct permissions when the grant is
genuinely one user.

**Synthetic-role artifacts in role listings.** The `__user_<id>__`
synthetic roles exist as `roles` rows. They are returned by `list_roles`
alongside admin-managed roles (we do not filter them at the API layer)
and the admin UI does not specially hide them either; an operator
inspecting the database or the Roles tab will see them. The
`_reject_synthetic_role_name` guard prevents collisions but doesn't make
them invisible. We treat this as acceptable: keeping the resolver
single-path is worth more than a hidden row, and the name pattern makes
them easy to recognise. A future iteration could filter them at the
handler layer if operator feedback suggests it.

**Migration backfill complexity.** The Alembic migration has to translate
seven tables' worth of grants into role rows correctly. The mapping is not
mechanical: workspace `READ` rewrites to `(workspace, *, USE)`; workspace
`EDIT` fans out to `(workspace, *, USE)` plus a type-wildcard `EDIT` on
every concrete resource type; scorer names need URL-encoding to match the
new pattern format. The migration is the load-bearing piece of the
backfill. We extensively tested it but the worst-case incorrect mapping
would be a silent over- or under-grant that survives until someone notices.

**Per-workspace UX double-bind.** The admin URL splits cleanly between
`/admin` (platform admin) and `/admin/ws?workspace=…` (per-workspace
management), but two URL shapes mean two breadcrumb-back targets. Our
detail pages currently land on `/admin` regardless of where the user came
from; that's a known cosmetic gap.

**No flag-controlled rollout.** We considered a feature flag that would
allow the auth server to read from either the legacy tables or
`role_permissions`. We rejected it because the cost of maintaining two
read paths exceeds the benefit, and because the simplified model is the
target end state regardless. Operators who need rollback fall back to the
retained legacy tables and a downgraded server build.

# Alternatives

**Keep per-resource permissions, add roles as a new layer.** Roles would
become bundles of "users to grant per-resource permissions to," not the
storage primitive. This was the natural minimal change but doesn't address
the fundamental cost of the per-resource model — it just adds another layer
on top.

**Per-table role tables.** A `experiment_roles`, `registered_model_roles`,
etc. set of tables, mirroring the legacy structure. We rejected this
because it preserves the per-table fan-out we want to eliminate. The whole
point is to consolidate.

**Path-based workspace routing.** `/admin/workspace/:name` instead of the
hybrid `/admin/ws?workspace=…`. Cleaner URLs, but `WorkspaceRouterSync` and
`prefixRouteWithWorkspace` are built around the query param and would need
extension. We chose the hybrid as the smaller change. A future RFC may
revisit this if a broader workspace-routing refactor is undertaken.

**Group / team primitive.** Teams as first-class identities, with role
assignments at the team level. Out of scope for this proposal but
forward-compatible with the role surface.

# Adoption strategy

The change is breaking, so adoption is staged.

**Phase 1 (landed)** added the role tables, role API, and role-based
permission resolver as a parallel surface. The legacy tables remained the
source of truth for permission decisions; roles existed but were not yet
load-bearing. This is a non-breaking addition.

**Phase 2 (in progress)** is the breaking part. The backfill migration
walks the seven legacy tables and writes equivalent rows into
`role_permissions` under synthetic per-user roles. The auth server flips
to reading from `role_permissions` only. All legacy permission endpoints
(25 per-resource + 4 workspace-permission) are removed at the same
upgrade; there is no deprecation-warning window. The legacy tables remain
on disk as a paused snapshot for rollback safety; no automated migration
drops them.

Operators upgrading need to:

1. **Audit clients and migrate calls.** Replace
   `create_experiment_permission(exp_id, username, permission)` etc.
   with either
   - `grant_user_permission(username, "experiment", exp_id, permission)`
     for one-off direct grants (closest 1-to-1 replacement), or
   - The role API (`create_role` + `add_role_permission` + `assign_role`)
     for reusable grants.
   The changelog includes a per-method migration table.
2. **Re-evaluate workspace permissions.** Workspace `MANAGE` semantics no
   longer fan out implicitly to every resource type. Operators who relied
   on the implicit fan-out need to either grant explicit `(resource_type,
   *, …)` rows under a workspace role, or use the seeded `admin` role
   (which carries `(workspace, *, MANAGE)`).
3. **Validate the backfill.** The Alembic migration runs at upgrade time.
   Operators with non-trivial permission setups should validate that
   role-derived grants match expectations on a staging copy before
   upgrading production. The migration is reversible by downgrade if
   needed.
4. **Optionally drop legacy tables.** After the simplified model has
   shipped for at least one minor release and operators have confirmed
   their grants, the release notes document a SQL snippet to drop the
   seven legacy permission tables for any deployment that wants to
   reclaim the disk space. No automated migration runs this; it's an
   opt-in cleanup.

**Owner-delegation.** Per-resource MANAGE retains delegation via the new
`grant_user_permission` / `revoke_user_permission` convenience APIs (same
per-resource MANAGE validator as the legacy endpoints they replace). A
user with `(experiment, 42, MANAGE)` can still share experiment 42 with
another user one-off. The role API itself remains admin-only — creating
roles, assigning users, attaching grants to roles is a workspace-admin /
super-admin action by design. The behavioural shift versus the legacy
permission model is purely API-surface: same authority model, different
method name.

# Open questions

**Group / team support.** We deferred groups out of scope, but the
`role_permissions` storage shape is forward-compatible with adding
`group_role_assignments`. Should we sketch the group surface in this RFC
to lock in the upgrade path, or leave it for a future RFC when the demand
crystallizes?

**Direct-permission ergonomics.** The synthetic per-user role pattern is
clean at the storage layer but slightly awkward to explain. The new
`grant_user_permission` / `revoke_user_permission` convenience APIs hide
the synthetic role at the surface; operators see a familiar
`(username, resource, permission)` shape. An alternative would be a
separate `direct_permissions` table (one row per direct grant), keeping
`role_permissions` strictly for shared roles. We chose the synthetic-role
path because the resolver stays single-path; flagging in case reviewers
feel the storage-layer simplicity isn't worth the slight
explanation overhead.

**Per-workspace admin UI structure.** The current implementation splits
`/admin` (platform) from `/admin/ws?workspace=…` (per-workspace). A
broader per-workspace UX (every workspace gets its own admin entry point,
no global view) was considered and deferred. We treat it as a follow-up;
no consensus needed in this RFC.
