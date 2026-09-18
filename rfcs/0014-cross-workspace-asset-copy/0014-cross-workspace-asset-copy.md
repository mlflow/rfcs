---
start_date: 2026-09-02
mlflow_issue: TBD
rfc_pr: https://github.com/mlflow/rfcs/pull/51
---

# RFC 0014: Cross-workspace MLflow asset sharing and copying

| **Date Last Modified** | 2026-09-17 |
| :--------------------- | :--------- |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
  - [Fork and promote a prompt](#fork-and-promote-a-prompt)
  - [Sync and detach a prompt](#sync-and-detach-a-prompt)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [Goals](#goals)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
  - [Operation semantics](#operation-semantics)
  - [Asset-specific copy contents](#asset-specific-copy-contents)
    - [Model artifact access](#model-artifact-access)
  - [Authorization and visibility](#authorization-and-visibility)
  - [MLflow UI](#mlflow-ui)
  - [Transactions, conflicts, and retries](#transactions-conflicts-and-retries)
  - [API](#api)
    - [Typed RPC routes](#typed-rpc-routes)
    - [Copy requests](#copy-requests)
    - [Detach requests](#detach-requests)
    - [Source deletion requests](#source-deletion-requests)
    - [Canonical resource responses](#canonical-resource-responses)
    - [Native MlflowClient Python SDK](#native-mlflowclient-python-sdk)
    - [Store interfaces](#store-interfaces)
    - [HTTP status and error behavior](#http-status-and-error-behavior)
  - [Database schema](#database-schema)
  - [Read and mutation behavior](#read-and-mutation-behavior)
  - [Rename and delete lifecycle](#rename-and-delete-lifecycle)
- [Acceptance criteria](#acceptance-criteria)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)

# Summary

This proposal lets teams share prompts, registered models, and MCP server registry entries across MLflow workspaces through the MLflow UI, Python SDK, and REST APIs.

It supports two complementary workflows. Teams can make selected assets from a shared workspace discoverable in their own workspace, with a read-only view that follows source changes. They can also create editable forks, develop them independently, and promote approved changes back. An independent copy supports reuse without retaining a relationship to the source.

Users can detach a synced asset into an independent editable snapshot, or detach a fork to remove its source relationship. Replacing an existing asset requires explicit consent. Deleting a source requires explicit detachment of synced dependents; existing forks keep their content and lose their source relationship automatically. Model copying preserves artifact references; it does not transfer the underlying model files.

# Basic example

These examples use the proposed Python SDK. The same actions are available from asset pages in the MLflow UI, as described in [MLflow UI](#mlflow-ui).

## Fork and promote a prompt

An engineer forks an approved prompt into the team's workspace:

```python
from mlflow import MlflowClient

client = MlflowClient()
prompt = client.get_prompt("support-assistant", workspace="shared")

forked_prompt = client.fork_prompt(
    prompt,
    target_workspace="team-search",
    target_name="support-assistant-team",
)
```

The team edits the fork using the existing prompt APIs in `team-search`. After the changes have been approved, the engineer promotes the current fork back:

```python
updated_prompt = client.get_prompt(
    "support-assistant-team", workspace="team-search"
)
promoted_prompt = client.copy_prompt(
    updated_prompt,
    target_workspace="shared",
    target_name="support-assistant",
    overwrite=True,
)
```

Without `overwrite=True`, an existing destination is left intact and the call reports a conflict. The returned prompt is the destination resource, equivalent to fetching it after the operation. Copying to a new name uses the same method with the default `overwrite=False`.

## Sync and detach a prompt

An engineer makes the approved prompt discoverable in the team's workspace under a local name:

```python
synced_prompt = client.sync_prompt(
    prompt,
    target_workspace="team-search",
    target_name="support-assistant-reference",
)
live_prompt = client.get_prompt(
    "support-assistant-reference", workspace="team-search"
)
```

The synced prompt follows source changes and is read-only. When the team needs to edit it, the engineer detaches it in place:

```python
editable_prompt = client.detach_prompt(
    workspace="team-search",
    name="support-assistant-reference",
)
```

The result is an independent editable snapshot containing the current source content, with no source relationship. Its name stays the same, and subsequent source changes no longer update it.

# Motivation

## The problem

Workspace isolation prevents teams from discovering and reusing selected assets from another workspace through a supported sharing workflow. An organization may maintain approved assets centrally, while teams need to find those assets locally, adapt them, and contribute approved improvements back. MLflow does not currently support this workflow across workspaces.

## Goals

- Make curated assets discoverable in team workspaces, with read-only synced assets transparently included in ordinary search and list results.
- Let teams fork an asset, develop it independently, and identify its source.
- Support independent copies and promotion of approved changes back to another workspace.
- Let users turn a synced asset into an independent editable snapshot without changing its local name.
- Provide these workflows in the MLflow UI as well as programmatic interfaces.
- Prevent accidental replacement of existing assets and prevent source deletion from leaving broken relationships.
- Respect source and destination access controls throughout sharing, reading, and editing.

## User journeys

1. An organization curates approved AI assets in a shared workspace. An engineer selects the assets relevant to their team so teammates and agents can find and use current versions from the team's workspace.
2. A team needs to improve an approved asset. An engineer forks it into the team's workspace, iterates there, and promotes the changes back after review.
3. An engineer discovers a useful asset in another team's workspace and copies it into their own workspace to develop it independently, without maintaining a source relationship.

The [basic example](#basic-example) illustrates these workflows with SDK calls; the API contract is defined below.

## Out of scope

- Single-version copying or partial version selection.
- Experiments, runs, evaluation datasets, and traces.
- Bulk workspace-level migrations.
- Bidirectional synchronization.
- Copying model artifact bytes or replicating storage.
- Authorizing cross-workspace artifact downloads through destination assets in artifacts-only deployments.

# Detailed design

## Operation semantics

An asset is identified by its resource type, workspace, and name. Operations apply to the complete parent asset and its version history.

| Operation | Destination content | Relationship to source |
| :--- | :--- | :--- |
| Sync | Read-only, live view of source metadata | `SYNC` |
| Fork | Editable snapshot | `FORK` |
| Copy | Editable snapshot | None |
| Detach | Materializes current synced content in place, or preserves existing fork content | Removes the `SYNC` or `FORK` relationship |

`SYNC` stores a relationship without creating destination asset or version rows. `FORK` and `COPY` materialize destination metadata. Sync and fork fail if the destination identity is already occupied by a native asset or another link, including a sync to the same source. Ordinary asset creation also rejects names occupied by links. These name checks and destination creation must be atomic so concurrent requests cannot create duplicate identities.

`COPY` replaces the destination's metadata and full version history only when `overwrite=true`; with the default `overwrite=false`, an existing destination causes a conflict.

A copied destination has no incoming source relationship, including when it replaces a fork. Detach removes the destination's incoming relationship: for `SYNC`, it first materializes the current source content; for `FORK`, it preserves the existing local content. Both results are independent editable assets.

Only `SYNC` dependents block source deletion without explicit detachment. `FORK` dependents retain their content and automatically lose their relationship when the source is deleted. The complete behavior is defined in [Rename and delete lifecycle](#rename-and-delete-lifecycle).

## Asset-specific copy contents

| Asset Type | Copied Metadata | Excluded / System Managed |
| :--- | :--- | :--- |
| **Prompt** | Prompt template text, descriptions, version history, asset/version tags, and aliases. (Prompts map physically to registered_model schema). | Workspace ownership, internal database primary keys, permissions. |
| **Registered Model** | Model description, asset tags, version history, version tags, aliases, and exact `source`/`storage_location` URI strings. | Workspace ownership, internal primary keys, permissions. Artifact bytes are never copied. |
| **MCP Server** | Parent `display_name`, `description`, `icons`, and tags; complete version history including `version`, `server_json`, version `display_name`, `status`, `tools`, `source`, and tags; aliases and their version targets. | Workspace ownership, internal primary keys, permissions, `MCPAccessBinding` records, system-managed audit fields, secret values, and runtime container deployment state. Parent `status` and `latest_version` remain derived fields. |

The MCP fields above correspond to the server, version, tag, and alias entities in [RFC 0004](../0004-mcp-registry/0004-mcp-registry.md#entities-and-data-model). Endpoint declarations inside `server_json`, including `remotes[]`, are part of the copied definition metadata. The separate approved connection records, `MCPAccessBinding`, are excluded. Copying these declarations does not automatically create approved access bindings in the destination.

This exclusion also applies when materializing synced MCP entries. Sync does not expose source bindings as destination bindings. Approved connections are configured separately in the destination.

### Model artifact access

- Model versions store artifact locations as URI pointers (`source`, `storage_location`), which are preserved verbatim without transferring binary bytes.
- In full MLflow deployments serving artifacts, access to the destination workspace and `can_read` on the destination model authorize listing and downloading its referenced artifacts through the UI and SDK, without requiring source-workspace access. This applies to synced models, forks, copies, and detached models, and is limited to the authorized versions' artifact locations. It grants no access to unrelated source files or permission to modify source artifacts.
- Copies and detached models use their stored artifact references without requiring an active source relationship. Referenced files must still exist. Artifacts-only deployments retain their existing authorization requirements; destination-based artifact authorization for those deployments is out of scope.
- Downstream execution environments accessing storage directly still need read credentials for the referenced storage URI scheme (e.g., shared object storage bucket access, cross-account IAM permissions, or container registry credentials).
- Physical binary data replication across air-gapped or isolated storage repositories is an out-of-band operational concern (handled by dedicated storage replication pipelines), decoupled from MLflow's metadata catalog operations.

Fork lineage is recorded by the parent relationship in `workspace_asset_links`. No additional provenance tags are attached to copied versions.

## Authorization and visibility

Authorization for copy and detach uses native MLflow permission primitives, distinguishing workspace access and resource creation from permissions on an existing asset:

- **Source Read Permission**: The requesting user must have access to the source workspace and `can_read` on the source asset.
- **Target Creation Permission**: Creating a destination requires access to the target workspace and its existing resource-creation permission.
- **Target Update Permission**: Updating an existing destination requires access to its workspace and `can_update` on that asset. Asset-level `can_use` alone does not authorize updates.

Under the current [permission definitions](https://github.com/mlflow/mlflow/blob/master/mlflow/server/auth/permissions.py), workspace-level `USE` includes resource creation. At asset scope, `READ` provides `can_read`, `USE` does not provide `can_update`, `EDIT` provides `can_update`, and `MANAGE` also provides `can_delete` and `can_manage`.

Overwriting an existing destination with `copy` (including promote-back) requires source read permission and both destination `can_update` and `can_delete`. Full-history replacement removes existing versions, so this follows MLflow's [model and prompt deletion checks](https://github.com/mlflow/mlflow/blob/master/mlflow/server/auth/__init__.py#L2578-L2597). Under the current permission levels, this requires effective `MANAGE` on the destination asset; `EDIT` alone is insufficient. The `overwrite=true` flag confirms intent and does not replace authorization. Copying to a new destination uses target creation permission, even when `overwrite=true`.

Detaching an existing fork requires only destination update permission, since it removes lineage without reading source content.

Permissions checks are evaluated before any database mutations are performed.

Source deletion continues to use the existing delete authorization.

## MLflow UI

The asset list and detail pages for prompts, registered models, and MCP servers include cross-workspace actions. A user selects **Sync**, **Fork**, or **Copy**, then chooses an accessible destination workspace and target name. The dialog explains whether the result follows source updates, retains fork lineage, or becomes independent.

Synced assets appear in the destination's normal lists and search results under their local names. Lists and detail pages show a synced or forked indicator, and the detail page links to the source when the user can access it. Synced detail pages expose versions and aliases and provide **Detach to edit**. Content editing controls remain disabled until detachment succeeds. Forks also support detach, preserving their local content. Both detach flows explain that the source relationship will be removed.

When sharing MCP entries or materializing synced entries, the UI warns that source access bindings are not carried over and approved connections must be configured separately in the destination.

The copy dialog submits with `overwrite=false` by default. If the destination exists, it identifies the destination and warns that replacement includes its full version history. Only confirming that replacement sends `overwrite=true`. A conflict discovered after the dialog opened returns to this confirmation flow. Promoting a fork uses the same copy dialog, prefilled with its source workspace and name.

Deleting a source with synced dependents first returns a conflict. The UI explains that synced dependents will become independent snapshots and forks will lose their source relationship. **Delete and detach dependents** sends `detach_dependents=true` after explicit confirmation; cancelling leaves all assets intact. Sources with only forks use the ordinary delete flow.

Successful actions open or refresh the destination detail page using the returned resource. These UI flows ship with the server and SDK support.

## Transactions, conflicts, and retries

Copy and detach retain the single SQL transaction design:

1. Validate input parameters and permissions.
2. Read the source snapshot or validate the relationship state.
3. Check destination conflicts across native assets and links. Sync and fork fail if the target exists. A copy can replace an existing target only with `overwrite=true`.
4. Materialize destination metadata where required and insert or remove the parent relationship.
5. Commit the transaction and return the destination resource.

The server enforces the overwrite parameter; UI confirmation is the user-facing step that supplies it.

A repeated sync, fork, or copy with `overwrite=false` can report that the target already exists. A request with `overwrite=true` remains an explicit replacement.

## API

### Typed RPC routes

The route selects the asset type; the request cannot substitute another resource type.

| Asset | Copy, sync, and fork | Detach |
| :--- | :--- | :--- |
| Prompt | `POST /api/3.0/mlflow/prompts/copy` | `POST /api/3.0/mlflow/prompts/detach` |
| Registered model | `POST /api/2.0/mlflow/registered-models/copy` | `POST /api/2.0/mlflow/registered-models/detach` |
| MCP server | `POST /api/2.0/mlflow/mcp-servers/copy` | `POST /api/2.0/mlflow/mcp-servers/detach` |

### Copy requests

`CopyPrompt`, `CopyRegisteredModel`, and `CopyMCPServer` have the same fields:

| Field | Required / default | Meaning |
| :--- | :--- | :--- |
| `source_workspace` | Required | Source workspace |
| `source_name` | Required | Source asset name |
| `target_workspace` | Required | Destination workspace |
| `target_name` | Defaults to `source_name` | Destination asset name |
| `relationship_type` | Defaults to `"FORK"` | `"SYNC"`, `"FORK"`, or `"COPY"` |
| `overwrite` | Boolean, defaults to `false` | Allows `COPY` to replace an existing destination |

A `SYNC` or `FORK` request returns `409 Conflict` if the destination is occupied by a native asset or another link. A `COPY` request against an existing destination without `overwrite=true` also returns `409 Conflict`. These conflicts leave the destination unchanged. The overwrite parameter applies only to `COPY` and cannot permit sync or fork to replace an existing destination.

### Detach requests

`DetachPrompt`, `DetachRegisteredModel`, and `DetachMCPServer` accept only the public target identity:

```json
{
  "workspace": "team-search",
  "name": "support-assistant-reference"
}
```

Both fields are required. The route supplies the resource type, and the server resolves the incoming `SYNC` or `FORK` relationship internally. The response is the canonical destination resource with `relationship` omitted; the SDK property is `None`.

### Source deletion requests

Existing parent-asset delete APIs gain an optional boolean `detach_dependents`, defaulting to `false`. The asset name and workspace continue to use each delete API's existing path, body, and workspace-scoping conventions.

With any outgoing `SYNC` relationship, the default request returns `409 Conflict` without changing the source or its dependents. With `detach_dependents=true`, synced dependents are materialized and their relationships are removed before deletion. Outgoing `FORK` relationships do not block deletion and are removed automatically whenever the source is deleted, without modifying fork content. Existing delete authorization and the successful delete response format are retained.

### Canonical resource responses

Copy and detach return the same resource representation and response envelope as the corresponding destination `GET`, evaluated after the change commits. They do not return an operation record. The SDK returns a `Prompt`, `RegisteredModel`, or `MCPServer`, respectively.

Parent resource representations gain `workspace` where it is not already exposed, plus optional `relationship` metadata. For example, the following fields appear within a forked prompt's normal resource representation:

```json
{
  "name": "support-assistant-team",
  "workspace": "team-search",
  "description": "A prompt for customer support",
  "relationship": {
    "type": "FORK",
    "source": {
      "workspace": "shared",
      "name": "support-assistant"
    }
  }
}
```

Other normal resource fields are omitted from this illustration only. A synced asset has the same relationship shape with `"type": "SYNC"`. Parent `GET`, search, and list representations expose the same relationship metadata. Native, independently copied, and detached resources omit `relationship`; their SDK property is `None`. Internal relationship identifiers are not part of the public contract.

### Native MlflowClient Python SDK

The SDK uses separate methods for distinct operations. The parent getters below support selecting the source workspace, as in the basic example. Other reads retain their existing workspace-scoping interface.

Proposed public signatures, with method bodies omitted:

```python
class MlflowClient:
    def get_prompt(
        self, name: str, *, workspace: str | None = None
    ) -> Prompt | None: ...

    def get_registered_model(
        self, name: str, *, workspace: str | None = None
    ) -> RegisteredModel: ...

    def get_mcp_server(
        self, name: str, *, workspace: str | None = None
    ) -> MCPServer: ...

    def sync_prompt(
        self, prompt: Prompt, *, target_workspace: str,
        target_name: str | None = None,
    ) -> Prompt: ...

    def fork_prompt(
        self, prompt: Prompt, *, target_workspace: str,
        target_name: str | None = None,
    ) -> Prompt: ...

    def copy_prompt(
        self, prompt: Prompt, *, target_workspace: str,
        target_name: str | None = None, overwrite: bool = False,
    ) -> Prompt: ...

    def detach_prompt(self, *, workspace: str, name: str) -> Prompt: ...

    def sync_registered_model(
        self, model: RegisteredModel, *, target_workspace: str,
        target_name: str | None = None,
    ) -> RegisteredModel: ...

    def fork_registered_model(
        self, model: RegisteredModel, *, target_workspace: str,
        target_name: str | None = None,
    ) -> RegisteredModel: ...

    def copy_registered_model(
        self, model: RegisteredModel, *, target_workspace: str,
        target_name: str | None = None, overwrite: bool = False,
    ) -> RegisteredModel: ...

    def detach_registered_model(
        self, *, workspace: str, name: str
    ) -> RegisteredModel: ...

    def sync_mcp_server(
        self, server: MCPServer, *, target_workspace: str,
        target_name: str | None = None,
    ) -> MCPServer: ...

    def fork_mcp_server(
        self, server: MCPServer, *, target_workspace: str,
        target_name: str | None = None,
    ) -> MCPServer: ...

    def copy_mcp_server(
        self, server: MCPServer, *, target_workspace: str,
        target_name: str | None = None, overwrite: bool = False,
    ) -> MCPServer: ...

    def detach_mcp_server(self, *, workspace: str, name: str) -> MCPServer: ...

    def delete_prompt(
        self, name: str, *, detach_dependents: bool = False
    ) -> None: ...

    def delete_registered_model(
        self, name: str, *, detach_dependents: bool = False
    ) -> None: ...

    def delete_mcp_server(
        self, name: str, *, detach_dependents: bool = False
    ) -> None: ...
```

The SDK takes the source workspace and name from the supplied parent resource. The server reads the source asset's current state and copies the complete asset.

`sync_*`, `fork_*`, and `copy_*` select `SYNC`, `FORK`, and `COPY` respectively on the shared server endpoint. Only `copy_*` exposes overwrite. The methods copy the whole parent asset, independent of which versions the caller has loaded.

MCP methods that share definitions or materialize synced entries emit a Python warning that source access bindings are not carried over and approved connections must be configured separately in the destination.

### Store interfaces

Prompts and registered models belong to the [model registry AbstractStore](https://github.com/mlflow/mlflow/blob/master/mlflow/store/model_registry/abstract_store.py). The proposed additions use explicit identities and a mode at the store boundary, while the public SDK keeps its separate methods:

```python
from typing import Literal

class AbstractStore:
    # mlflow/store/model_registry/abstract_store.py

    def copy_prompt(
        self, *, source_workspace: str, source_name: str,
        target_workspace: str, target_name: str | None = None,
        relationship_type: Literal["SYNC", "FORK", "COPY"] = "FORK",
        overwrite: bool = False,
    ) -> Prompt: ...

    def detach_prompt(self, *, workspace: str, name: str) -> Prompt: ...

    def copy_registered_model(
        self, *, source_workspace: str, source_name: str,
        target_workspace: str, target_name: str | None = None,
        relationship_type: Literal["SYNC", "FORK", "COPY"] = "FORK",
        overwrite: bool = False,
    ) -> RegisteredModel: ...

    def detach_registered_model(
        self, *, workspace: str, name: str
    ) -> RegisteredModel: ...

    # Existing deletes retain their name argument and workspace context.
    def delete_prompt(
        self, name: str, *, detach_dependents: bool = False
    ) -> None: ...

    def delete_registered_model(
        self, name: str, *, detach_dependents: bool = False
    ) -> None: ...
```

MCP operations belong to the [tracking AbstractStore](https://github.com/mlflow/mlflow/blob/master/mlflow/store/tracking/abstract_store.py), through the `MCPServerRegistryMixin` described in [RFC 0004](../0004-mcp-registry/0004-mcp-registry.md#abstract-store-interface). Its additional effective signatures are:

```python
class AbstractStore(MCPServerRegistryMixin, GatewayStoreMixin):
    # mlflow/store/tracking/abstract_store.py; methods supplied by the MCP mixin.

    def copy_mcp_server(
        self, *, source_workspace: str, source_name: str,
        target_workspace: str, target_name: str | None = None,
        relationship_type: Literal["SYNC", "FORK", "COPY"] = "FORK",
        overwrite: bool = False,
    ) -> MCPServer: ...

    def detach_mcp_server(self, *, workspace: str, name: str) -> MCPServer: ...

    def delete_mcp_server(
        self, name: str, *, detach_dependents: bool = False
    ) -> None: ...
```

SQL store implementations perform the copy and detach transactions. REST store implementations forward these calls to the typed endpoints. The delete methods expose the option to detach dependents as part of source deletion.

### HTTP status and error behavior

| Status | Condition |
| :--- | :--- |
| `200 OK` / `201 Created` | Copy or detach succeeds and returns the canonical destination resource |
| `400 Bad Request` | Missing required fields, invalid parameters, or content mutation/deletion of a read-only synced asset; `INVALID_PARAMETER_VALUE` |
| `403 Forbidden` | Insufficient source or destination permissions |
| `404 Not Found` | Required source or target asset does not exist; `RESOURCE_DOES_NOT_EXIST` |
| `409 Conflict` | Destination occupied by a native asset or link for sync, fork, or copy without overwrite; `RESOURCE_ALREADY_EXISTS` |
| `409 Conflict` | Source deletion has synced dependents without `detach_dependents=true`; `RESOURCE_CONFLICT` |

## Database schema

One table, `workspace_asset_links`, stores active parent-level `SYNC` and `FORK` relationships. It is the source of truth for relationship metadata. Copying creates no relationship row, detach removes the incoming row, and no separate operation history or version-mapping table is needed.

```sql
CREATE TABLE workspace_asset_links (
    association_id VARCHAR(32) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    relationship_type VARCHAR(16) NOT NULL, -- "SYNC" or "FORK" ONLY
    source_workspace VARCHAR(63) NOT NULL,
    source_name VARCHAR(256) NOT NULL,
    target_workspace VARCHAR(63) NOT NULL,
    target_name VARCHAR(256) NOT NULL,
    lineage_id VARCHAR(32) NOT NULL,
    created_by VARCHAR(256),
    creation_time BIGINT NOT NULL,
    CONSTRAINT workspace_asset_links_pk PRIMARY KEY (association_id),
    CONSTRAINT workspace_asset_links_target_uk UNIQUE (target_workspace, target_name, resource_type)
);

CREATE INDEX idx_workspace_asset_links_source ON workspace_asset_links (source_workspace, source_name);
CREATE INDEX idx_workspace_asset_links_target ON workspace_asset_links (
    target_workspace, resource_type, relationship_type
);
```

The unique target identity lets detach resolve the relationship from workspace, name, and the route's resource type. The association and lineage identifiers remain internal; they are not exposed in requests or resource responses.

The target index supports search joins filtered by destination workspace, resource type, and `SYNC` relationship. The unique target constraint supports exact destination-identity lookups.

## Read and mutation behavior

Synced assets are reachable through standard reads using their destination identity. After syncing `shared/support-assistant` to `team-search/support-assistant-reference`, getting the latter returns current source metadata with `workspace="team-search"` and `name="support-assistant-reference"`. Callers do not need to resolve a relationship themselves.

The following SDK methods and their REST counterparts include synced entities:

| Read surface | Prompts | Registered models | MCP servers |
| :--- | :--- | :--- | :--- |
| Parent get | `get_prompt` | `get_registered_model` | `get_mcp_server` |
| Version get | `get_prompt_version` | `get_model_version` | `get_mcp_server_version` |
| Alias resolution | `get_prompt_version_by_alias`; alias passed to `get_prompt_version` | `get_model_version_by_alias` | `get_mcp_server_version_by_alias` |
| Version search/list | `search_prompt_versions` | `search_model_versions`, `get_latest_versions` | `search_mcp_server_versions`, `get_latest_mcp_server_version` |
| Parent search/list | `search_prompts` | `search_registered_models` | `search_mcp_servers` |

Version reads return source version content under the destination parent identity. Aliases and latest-version lookups follow current source assignments, including changes after the sync was created. UI lists use these same read surfaces.

Search filters on workspace and name apply to the destination identity, while content filters use the visible source metadata. Native and synced results participate in the same filtering, ordering, and pagination contract. Pagination must include eligible synced assets without duplicates or omissions in an unchanged result set. The implementation does not require callers to merge separate result streams.

Content mutations of synced destinations, such as creating versions, changing tags, or deleting the asset, return `INVALID_PARAMETER_VALUE` because synced assets are read-only. Detach enables independent editing and removes the source relationship. Forks, independent copies, and detached assets use ordinary destination reads and writes; only forks include source relationship metadata.

## Rename and delete lifecycle

Renaming an asset via `rename_registered_model` cascades atomically to `workspace_asset_links` within the exact same database transaction. If a source asset is renamed, active `SYNC` and `FORK` links update `source_name` to follow the renamed asset, preventing broken links or 404 errors. If a target asset is renamed in its destination workspace, `target_name` is updated.

Deleting a source with `SYNC` dependents returns `409 Conflict` by default, leaving all assets and relationships unchanged. `FORK` relationships alone do not block deletion. Once deletion is allowed, the transaction proceeds as follows:

1. If synced dependents exist, snapshot their current content into their destinations; this requires `detach_dependents=true`.
2. Remove all outgoing `SYNC` and `FORK` relationships. Existing fork content is unchanged; the dependents no longer reference the deleted source.
3. Delete the source asset.

These changes are committed together so that source deletion does not leave dangling links. Detaching a single synced asset or fork also removes its incoming relationship, while preserving the source itself.

Deleting a fork or copy target removes its local rows and incoming relationship, if present. If that target also serves as a source, the same source-deletion guard applies.

# Acceptance criteria

1. The MLflow UI, typed REST endpoints, and Python SDK support sync, fork, copy, and detach for all three asset types.
2. Synced assets appear under their destination identity in every read surface listed above, including versions and aliases. Source changes are reflected in subsequent server reads; normal filtering, ordering, pagination, and visibility rules apply.
3. Updating content, setting tags, creating versions, or deleting synced assets fails with `INVALID_PARAMETER_VALUE`. Detaching a sync produces an independent editable snapshot; detaching a fork preserves its local content. Both remove the incoming relationship without changing the destination identity.
4. Sync and fork reject destination names occupied by native assets or links. Ordinary asset creation also rejects names occupied by links. Copy rejects occupied names unless `overwrite=true`; conflicts leave the destination unchanged.
5. Confirmed copy replaces destination metadata and versions atomically and leaves no incoming source relationship on the copied asset. Replacing an existing destination requires both `can_update` and `can_delete`; insufficient permission returns `403 Forbidden` without changes, even with `overwrite=true`.
6. Copy and detach responses match the corresponding destination `GET`. Parent get/search/list representations include source relationship metadata only for `SYNC` and `FORK`.
7. The SDK exposes separate typed sync/fork/copy methods, accepts source entities with workspace identity, and returns destination entities. Detach accepts workspace and name.
8. The model registry and tracking stores implement the proposed contracts and use one parent relationship table.
9. The UI requires explicit overwrite confirmation and supports deletion with explicit dependent detachment. Server checks enforce the same rules for direct API clients.
10. Only `SYNC` dependents block source deletion by default. Explicit detachment snapshots synced dependents. Successful source deletion preserves fork content and removes all outgoing relationships; forks alone require no detachment flag.
11. Renames through the existing registered-model API update the corresponding relationship names.
12. Copies preserve the asset-specific metadata described above without transferring artifact bytes or adding provenance tags.
13. MCP access bindings are excluded, including automatic creation from copied `server_json.remotes[]`. The UI and Python SDK warn users that approved connections must be configured separately in the destination.
14. In full deployments serving artifacts, destination model permissions allow listing and downloading only the referenced artifacts through the UI and SDK, including after detachment. Artifacts-only deployments and direct external storage retain their existing access requirements.

# Drawbacks

- Resolving live `SYNC` references adds complexity to asset queries.
- Promoting back replaces parent asset metadata and version history with the fork's content rather than merging version trees.
- Detach removes lineage. Users who want an editable snapshot with a retained source relationship must create a fork instead.

# Alternatives

- **Copy artifact files as well as metadata** would make storage independently manageable, but requires a separate transfer and failure-handling design. This proposal preserves artifact references and leaves storage replication to existing tooling.
- **Separate per-asset table prefixes**: Using `prompt_links`, `model_links`, etc. Rejected in favor of the unified `workspace_asset_links` table.
- **Remove COPY and offer only SYNC and FORK** would require a separate detach for independent copies and a new overwrite mechanism for promotion. COPY supports both workflows directly.

# Adoption strategy

1. Add the relationship table through the applicable SQL store migrations.
2. Implement the store contracts, existing read and lifecycle integrations, typed REST endpoints, and SDK methods, including artifact authorization in full deployments and MCP access-binding warnings.
3. Expose the workflows in the MLflow UI, including overwrite confirmation, MCP access-binding warnings, and source deletion with explicit detachment of synced dependents.
