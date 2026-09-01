# RFC-0008: Skill Registry Implementation Details

This document contains implementation-level specifications for
RFC-0008 (Skill Registry). It covers database schema, entity
dataclasses, store interface method signatures, REST API endpoints,
pagination/filtering, SDK convenience functions, and CLI mapping. These details
support implementers; the main RFC covers the design rationale.

## Database schema

Tables are created via a single Alembic migration. All tables are
workspace-scoped.

### `skills`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `organization` | `String(256)` | PK, default `''` (empty string) |
| `name` | `String(256)` | PK |
| `description` | `String(5000)` | |
| `icons` | `JSON` | nullable; mutable presentation metadata, list of icon descriptors (see `RegistryIcon`) |
| `search_text` | `Text` | derived discovery projection of name and description (plus member `keywords` on import) |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

PrimaryKey: `(workspace, organization, name)`.

A skill name is unique within its `(workspace, organization)`: the primary
key admits exactly one `skills` row per name, so a name identifies a single
skill whether that skill was registered standalone or created by importing a
packaged plugin. There is no stored ownership field. Whether a skill was
created by an import, and which plugins reference it, is derived from
`agent_plugin_members` rows rather than a column on `skills` (see Registration
behavior). Import and standalone registration both rely on the primary-key
uniqueness constraint to detect a name collision.

### `skill_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skills` |
| `organization` | `String(256)` | PK, FK to `skills` |
| `name` | `String(256)` | PK, FK to `skills` |
| `version` | `Integer` | PK, server-assigned monotonic integer |
| `source_type` | `String(20)` | server-set; `git`, `oci`, `zip`, `mlflow` |
| `source` | `String(2048)` | nullable pointer to skill content |
| `ref` | `String(2048)` | nullable; git branch, tag, or commit |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `digest` | `String(64)` | content hash of the resolved skill content, computed client-side and submitted; stored and indexed but not server-verified and not used to drive import reuse (see Content digest) |
| `status` | `String(20)` | default `'active'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, organization, name)` references `skills`, CASCADE
delete. This supports administrative hard deletion of the parent
`Skill`; normal version deletion is a status transition to `deleted`
and does not physically remove the version row.

**Version ordering**: versions are monotonic integers assigned by
the server as `MAX(version) + 1` over the skill's existing rows, the same
allocation the model registry uses. The maximum is taken over all rows for the
skill, including soft-deleted ones, so a number is never reused while any
version, live or `deleted`, still holds it. Concurrent registrations that
compute the same next number collide on the `(workspace, organization, name,
version)` primary key; the losing writer catches the uniqueness violation,
recomputes `MAX(version) + 1`, and retries. Ordering is a simple integer
comparison.

Reuse of a number is bounded by the deletion lifecycle. Soft-deleting a version
(`delete_skill_version`) keeps its row, so allocation keeps climbing and a
soft-deleted number is never reused. A number restarts at 1 for an identity only
when the parent skill is hard-deleted (`delete_skill`), which physically removes
every row for that identity so `MAX(version)` resets to the empty case.

Reusing a number never reuses a content location. Each `source_type=mlflow`
upload stores its content at a unique server-chosen artifact path recorded in the
version's `source` (see Client-side upload flow), not at a path derived from the
version number, so a new version always writes to a fresh path even when it takes
a number a prior incarnation once held, and two concurrent registrations never
write the same path. A hard delete reclaims the stored content it owns: because
artifact storage is not transactional with the registry database, the server
captures the artifact paths recorded on the versions being removed, commits the
row deletion, and then deletes those paths best-effort, keeping a shared package
tree until its last referencing version is gone (see Deletion semantics).
Committing the row deletion before touching content ensures a rolled-back
deletion never leaves a restored row pointing at missing bytes; if the
post-commit cleanup instead fails, the orphaned bytes are the same small,
already-accepted leak as a crashed upload. Because it deletes only paths owned by
the versions being removed
rather than scanning the store for unreferenced paths, it never deletes a whole
identity prefix, never touches a recreated skill's fresh-token paths, and never
races an in-flight upload whose row has not yet committed (that path is recorded
on no committed version, so the delete never considers it). The only content a
delete leaves behind is bytes from a crashed upload that never committed a row;
those pre-commit paths are intentionally retained, because no committed row
distinguishes them from an upload still in progress, and the residue is a small,
rare leak that never affects correctness.
Remote-pointer registrations (`git`/`oci`/`zip`) store no content in the registry
and need none of this.

**Index**: `ix_skill_versions_latest_lookup` on `(workspace,
organization, name, status, version)` supports latest-resolution
lookups.

**Digest index**: `ix_skill_versions_digest` on `(workspace,
organization, name, digest)` supports grouping and lookup of a skill's
versions by content, which is the digest's read-side purpose (see Content
digest).

### `skill_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skills` |
| `organization` | `String(256)` | PK, FK to `skills` |
| `name` | `String(256)` | PK, FK to `skills` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skill_versions` |
| `organization` | `String(256)` | PK, FK to `skill_versions` |
| `name` | `String(256)` | PK, FK to `skill_versions` |
| `version` | `Integer` | PK, FK to `skill_versions` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skills` |
| `organization` | `String(256)` | PK, FK to `skills` |
| `name` | `String(256)` | PK, FK to `skills` |
| `alias` | `String(256)` | PK |
| `version` | `Integer` | target `skill_versions.version` under the same parent; see alias integrity below |

**Alias integrity.** An alias `version` targets a version row under the
same `(workspace, organization, name)` parent. Integrity is enforced by
the application rather than by a separate database FK: `set_skill_alias`
(and `set_agent_plugin_alias`) rejects a missing or `deleted`
target, and because a soft-deleted row persists a plain FK could not enforce the
not-`deleted` rule. Soft-deleting a
version removes any aliases that point to it. Agent plugin aliases (`agent_plugin_aliases`) follow the same
pattern against `agent_plugin_versions.version`, and additionally reject a
withdrawn target (a plugin version that contains a `deleted` member), so
an alias cannot bypass the kill switch (see Deletion semantics).

### `agent_plugins`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `organization` | `String(256)` | PK, default `''` (empty string) |
| `name` | `String(256)` | PK |
| `description` | `String(5000)` | |
| `icons` | `JSON` | nullable; mutable presentation metadata, list of icon descriptors (see `RegistryIcon`) |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

PrimaryKey: `(workspace, organization, name)`.

### `agent_plugin_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `version` | `String(256)` | PK; equal to canonical `plugin_json["version"]` |
| `version_major` | `Integer` | extracted SemVer major component |
| `version_minor` | `Integer` | extracted SemVer minor component |
| `version_patch` | `Integer` | extracted SemVer patch component |
| `plugin_json` | `JSON` | canonical Agent Plugins manifest; immutable after creation, with the version field canonicalized (normalized) on ingest |
| `search_text` | `Text` | derived discovery projection of name, mutable parent description, organization, and this version's manifest description, keywords, and author name; parent search matches the latest-resolved version's value |
| `source_type` | `String(20)` | server-set; `git`, `oci`, `zip`, `mlflow`, `assembled` |
| `source` | `String(2048)` | optional pointer to agent plugin |
| `ref` | `String(2048)` | nullable; git branch, tag, or commit |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `status` | `String(20)` | default `'active'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, organization, name)` references `agent_plugins`,
CASCADE delete.

**Version ordering.** All versions are valid SemVer (semverish inputs are
normalized on creation). Latest resolution first selects active candidates, or
non-deleted non-active candidates when none are active. Semantic precedence
determines the result. The numeric components narrow candidates efficiently;
full prerelease precedence is applied in application code. Creation time breaks
equal semantic precedence, including versions that differ only in build
metadata.

**Index:** `ix_agent_plugin_versions_latest_lookup` on `(workspace,
organization, name, status, version_major, version_minor, version_patch,
creation_timestamp)` supports both resolution paths.

### `agent_plugin_version_members`

| Column | Type | Notes |
|--------|------|-------|
| `plugin_workspace` | `String(63)` | PK, FK to `agent_plugin_versions` |
| `plugin_organization` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `plugin_name` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `plugin_version` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `member_organization` | `String(256)` | PK, FK to `skill_versions` |
| `member_name` | `String(256)` | PK, FK to `skill_versions` |
| `member_version` | `Integer` | PK, FK to `skill_versions` |

FK: `(plugin_workspace, plugin_organization, plugin_name,
plugin_version)` references `agent_plugin_versions`, CASCADE
delete. A FK to `skill_versions` via `(plugin_workspace,
member_organization, member_name, member_version)` enforces
referential integrity with RESTRICT delete. The RESTRICT applies to
physical (hard) deletion of a `skill_versions` row; because membership rows
held only by soft-deleted plugin versions are purged before the referential
check, the RESTRICT fires only on references from live (non-`deleted`) plugin
versions (see Deletion semantics). It does not block a
member's soft delete (status transition to `deleted`), which is allowed
and handled as a derived withdrawal of the containing plugin versions
across resolution, discovery, and pull (see Deletion semantics). Skills
and agent plugins share the same workspace;
`plugin_workspace` is reused for the skill FK.

**Member-name uniqueness.** A `UNIQUE` constraint on `(plugin_workspace,
plugin_organization, plugin_name, plugin_version, member_name)` enforces that
member names are distinct within an agent plugin version. The primary key alone
does not guarantee this, because it also includes `member_organization` and
`member_version`; those two columns are retained for the `skill_versions` FK and
as stored data, not to distinguish rows for uniqueness. For an assembled plugin
the `skills/<member-name>/` pull layout is keyed on the name alone, so a name
collision would be ambiguous on disk, and the server rejects a create request
whose member list repeats a name. For a packaged plugin the importer derives
each member skill's name from its `SKILL.md` and rejects a package in which
two skills resolve to the same name; distinct `skills/*/SKILL.md` directories do
not by themselves guarantee this, since a skill's name is declared in `SKILL.md`
rather than taken from the directory, so the check is on the derived names
rather than the directory paths. Because member names are unique within a
version, a plugin version cannot contain two skills of the same name from
different organizations. Create requests with duplicate member names are
rejected before insert.

### `agent_plugin_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `agent_plugin_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugin_versions` |
| `organization` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `name` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `version` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `agent_plugin_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `alias` | `String(256)` | PK |
| `version` | `String(256)` | target agent plugin manifest version |

**Canonical manifest storage.** `plugin_json` uses SQLAlchemy's `JSON` type,
following RFC-0004's `server_json` precedent. It maps to native JSON where the
database supports it and to the platform's text-backed JSON representation for
SQLite and SQL Server. The full payload is preserved, while identity, ordering,
and search projections are materialized separately.

**Workspace handling.** All tables carry a `workspace` column as part
of the composite key. Single-tenant deployments use `'default'`.
Child tables reference the parent by `(workspace, organization,
name)`, so workspace is part of every table's primary key.

**Timestamps.** Set at the application layer via
`get_current_time_millis()`, not via DDL defaults.

**Deletion semantics.** The registry follows the mixed deletion pattern
used by the Model Registry and RFC-0004:

- Top-level entity delete operations (`delete_skill` and
  `delete_agent_plugin`) are administrative hard deletes. They
  physically remove the parent row and cascade to child rows, subject
  to referential-integrity checks.
- `delete_agent_plugin` takes an explicit `cascade` option that controls
  what happens to the plugin's member skills:
  - **Without cascade (default):** only the agent plugin and its versions,
    tags, aliases, and membership rows are removed. Member skills, including
    those created by importing a packaged plugin, are left in place as
    ordinary standalone skills; each keeps its own versions, ACL, and derived
    source, and remains independently gettable and pullable. Because each
    imported member's source pointer, including the artifact base of an
    MLflow-stored package, is persisted on the skill version rather than
    resolved from membership, removing the membership rows never strands a
    surviving member. Nothing is removed just because a skill was once a
    member.
  - **With cascade:** the plugin's member skills are hard-deleted along with
    the plugin, subject to the same referential-integrity checks that guard
    any skill hard delete. The checks run before anything is removed: if any
    member is still referenced by a live (non-`deleted`) plugin version other
    than the one being deleted, or otherwise fails the integrity check, the
    entire cascade delete fails with an error naming the blocking references
    and nothing is removed. Membership rows held only by soft-deleted plugin
    versions do not block the cascade, and hard-deleting a member also purges
    those stale membership rows. The operator resolves a genuine conflict (for
    example, by deleting or re-versioning the other plugin) and retries. If two
    plugins reference each other's members so that each cascade blocks the
    other, the operator breaks the deadlock by deleting one plugin without
    cascade first, then retrying the cascade delete of the other. Failing
    atomically is less surprising than silently retaining some members while
    deleting the rest, and it still guarantees a cascade never breaks a
    different live plugin that shares a member.
    Externally-sourced skills (git/oci/zip) are deleted only as registry
    entities; their upstream content is untouched.
  MLflow-managed package content (`source_type="mlflow"`) is refcounted by
  the skill versions whose persisted `source` points into it and is retained
  until the last referencing skill version is gone, so a non-cascade delete
  that leaves members also leaves the stored package tree those members pull
  from. When a hard delete removes a skill (or agent plugin) version, the delete
  operation reclaims the artifact path recorded on that version: the server
  commits the row deletion, then deletes the recorded path best-effort, since
  artifact storage is not transactional with the registry database (see Version
  ordering). A standalone upload's path is unique to its one version and is
  reclaimed with it, while a shared package tree is reclaimed only once its last
  referencing version is gone (the refcount above). A delete thus reclaims only
  paths owned by the versions it removes, never a whole identity prefix.
- Version delete operations (`delete_skill_version` and
  `delete_agent_plugin_version`) are soft deletes. They set
  `status='deleted'` when allowed by the lifecycle transition rules,
  update `last_updated_timestamp`, remove aliases that point to the
  deleted version, and exclude the version from normal
  get/search/list/latest resolution. Active versions must first be
  unpublished or deprecated before they can be deleted. A soft-deleted
  skill version also withdraws every agent plugin version that contains
  it, whether as a pinned member (assembled) or a member of a packaged
  plugin: such a plugin version is treated as `deleted` for
  resolution, discovery, and pull, so soft delete acts as a kill switch
  across both plugin kinds. The withdrawal is derived from member status,
  so the containing plugin version's membership rows and stored `status`
  are unchanged; an operator can publish a replacement plugin version
  referencing a fixed member. A `deprecated` member, by contrast, does not
  trigger withdrawal and still resolves and pulls.
- The `deleted` status is terminal. Internal audit or provenance paths
  may retain enough metadata to explain historical agent plugin
  snapshots, but deleted versions are not surfaced to consumers.

## Entity dataclasses

### Skill entity

```python
from dataclasses import dataclass, field
from enum import StrEnum
from typing import Any, TypedDict

from typing_extensions import NotRequired


class RegistryIcon(TypedDict):
    """Icon descriptor for registry presentation metadata.

    Deliberately the same shape as RFC-0004's MCPIcon (which follows the
    upstream MCP server.json icon schema), so UIs share one icon
    renderer across the MCP, skill, and agent plugin registries.
    """

    src: str
    sizes: NotRequired[list[str]]
    mimeType: NotRequired[str]
    theme: NotRequired[str]


class SkillStatus(StrEnum):
    DRAFT = "draft"
    ACTIVE = "active"
    DEPRECATED = "deprecated"
    DELETED = "deleted"


@dataclass
class Skill:
    name: str
    organization: str = ""
    description: str | None = None
    icons: list[RegistryIcon] | None = None  # mutable user-defined icons; API returns null when unset
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, int] = field(default_factory=dict)  # read-only; populated from skill_aliases table, e.g. {"production": 2}
    latest_version: int | None = None  # read-only, shared latest-resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Human-readable name, unique within `(workspace, organization)` |
| `organization` | `str` | Scopes ownership (e.g., team or publisher); defaults to `""` (empty string) |
| `description` | `str` | Optional human-readable description of the skill |
| `icons` | `list[RegistryIcon] \| None` | Mutable MLflow-managed presentation metadata; returned exactly as stored (null when never set). No payload fallback exists: the Agent Skills format defines no icon field, so the UI shows its default glyph when unset |
| `status` | `SkillStatus \| None` | Read-only; derived from the parent-resolved version: highest active version number if present, otherwise highest non-deleted non-active version number. `None` when the skill has no non-`deleted` version |
| `aliases` | `dict[str, int]` | Stable version pointers (e.g., `{"production": 2}`); read-only, populated from `skill_aliases` table |
| `latest_version` | `int \| None` | Read-only; highest version number among `active` versions if one exists, otherwise highest non-`deleted` non-`active` version. `None` when the skill has no non-`deleted` version |
| `workspace` | `str` | Visibility boundary |

### SkillVersion entity

```python
class SkillSourceType(StrEnum):
    GIT = "git"
    OCI = "oci"
    ZIP = "zip"
    MLFLOW = "mlflow"
    ASSEMBLED = "assembled"  # agent plugin version composed from member references


@dataclass
class GitSource:
    url: str
    ref: str | None = None
    subpath: str | None = None


@dataclass
class OCISource:
    image: str
    subpath: str | None = None


@dataclass
class ZipSource:
    url: str
    subpath: str | None = None


@dataclass
class SkillVersion:
    name: str
    version: int
    organization: str = ""
    source: GitSource | OCISource | ZipSource | str | None = None  # plain str for mlflow content
    source_type: SkillSourceType | None = None
    digest: str | None = None
    status: SkillStatus = SkillStatus.ACTIVE

    tags: dict[str, str] = field(default_factory=dict)
    aliases: list[str] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Skill name (part of composite key with workspace and organization) |
| `version` | `int` | Server-assigned monotonic integer. Each new version receives the next integer |
| `organization` | `str` | Organization scope, from parent Skill |
| `source` | `GitSource \| OCISource \| ZipSource \| str \| None` | Typed source descriptor for external content (git, OCI, zip); a plain `str` for `mlflow` content. For `source_type="mlflow"`, `source` is a server-set artifact path: for a standalone MLflow-stored skill it is the unique path where the server stored the uploaded content, and for a skill created by importing an MLflow-stored packaged plugin it is the package's artifact base path (a self-contained internal `mlflow-artifacts:` pointer captured at import time, with `subpath` locating the skill within that tree). A skill created by importing a packaged plugin more generally carries a source derived from the package: the package's `source_type` and `source` with a `subpath` locating the skill within the package. Because an imported member's pointer is stored on the skill version itself, its content resolves without reference to any membership row. The REST API represents this as flat `source_type`, `source`, `ref`, `subpath` fields; the SDK wraps and unwraps the typed classes (an `mlflow` `source` is a plain artifact-path string rather than a typed class) |
| `source_type` | `SkillSourceType \| None` | Server-set discriminator (`git`, `oci`, `zip`, `mlflow`), populated on responses. On create, a client that knows the type of an external pointer (CLI subcommand, SDK typed class) submits it and the server validates it against the source value; without an explicit type the server infers it (see the field-inference rules below). `mlflow` is flow-derived and never client-supplied. Together with `source` it determines how content is stored and how `pull` routes |
| `digest` | `str \| None` | Content hash of the resolved skill content (the same notion as a dataset `digest`). Computed by the client during local inspection and submitted at registration and import; client-asserted and not server-verified. It identifies a version by content within a skill name. Stored, returned on get, and indexed so callers can group versions by content (clean diffs between agent plugin versions, and traces before and after a change linking to the same content); it does not drive import, which always creates a new member version (see Content digest) |
| `status` | `SkillStatus` | Per-version lifecycle: `draft`, `active`, `deprecated`, `deleted` |
| `aliases` | `list[str]` | Alias names currently pointing at this version (read-only, projected from alias table) |

### SkillVersion field details

**Source type extensibility.** The `source_type` enum is intentionally
small for the initial implementation. New source types (e.g., `s3`,
`azure-blob`, `opensharing`) can be added without schema changes
since the column stores a string value. In particular, the
[OpenSharing](https://github.com/OpenSharing-IO/OpenSharing) protocol
(Linux Foundation) defines AgentSkill as a first-class asset type
using the same SKILL.md directory structure. An `opensharing` source
type would let the registry govern and track skills whose content is
shared via OpenSharing's credential-vending protocol.

**Typed source classes.** Each source type has a dedicated class with
only the fields relevant to that type:

| Class | Fields | Description |
|---|---|---|
| `GitSource` | `url`, `ref`, `subpath` | Git repository. `url` is the clone URL, passed to git as given: any URL the caller's git can clone is valid, and a `.git` suffix is not required (the suffix matters only to the inference fallback, never to fetching). Provider web URLs (e.g., a GitHub `/tree/...` page) are not clone URLs; supply the clone URL and put the branch and directory in `ref` and `subpath`. `ref` is the branch, tag, or commit (optional; defaults to the repository's default branch). `subpath` is the path within the repo (optional). |
| `OCISource` | `image`, `subpath` | OCI image. `image` is the image reference, optionally supplied with the `oci://` scheme (e.g., `oci://ghcr.io/acme/plugin:v1`). The scheme is a hint only, used to infer `source_type` when no explicit type accompanies the request and validated against it when one does: the persisted `source` is the bare reference (`ghcr.io/acme/plugin:v1`). `subpath` is the path within the image (optional). |
| `ZipSource` | `url`, `subpath` | ZIP archive. `url` is the archive URL. `subpath` is the path within the archive (optional). |

MLflow artifact storage does not use a source class; its `source` is a plain
artifact-path string. For a standalone upload it is the unique path where the
server stored the content (see Client-side upload flow); the whole stored tree
is the skill, so `subpath` is null. For a skill imported from a package it is
the artifact base of the package tree, and `subpath` locates the skill within
it.

The REST API represents these as flat fields (`source_type`,
`source`, `ref`, `subpath`); the SDK converts between typed classes
and flat fields, and the conversion carries the class's type through as
the request's `source_type` (`GitSource` submits `source_type="git"`,
and so on), so the explicit choice the caller already made is not
discarded in the flat representation. The server validates a submitted
`source_type` against the source value, infers the type when none is
submitted, sets it from the creation flow for content it stores itself
(the upload flow; see the field-inference rules below),
and returns it in responses. The SDK surfaces `source_type` as a field on
the version and uses it to reconstruct the typed class for external
sources; for `mlflow` content the `source` is a plain artifact-path string,
the server-stored upload path for a standalone skill or the package artifact
base for an imported member.

**Content digest.** Each version carries a `digest`, a SHA-256 hash
(lowercase hex, 64 characters) of the resolved skill content: the `SKILL.md`
and the files under the skill's `subpath`. The digest is computed over a
canonical, length-delimited serialization of that content. The client collects
the regular files under the skill root, normalizes each file's path to a POSIX
relative path (forward-slash separators, UTF-8, Unicode NFC), and sorts the
files by the byte value of that path. If two distinct files normalize to the
same POSIX path (for example, names that differ only in Unicode composition),
the client rejects the tree as ambiguous rather than choosing an order, so the
sort key is always total and the digest is deterministic regardless of
traversal order. It then feeds each file to the hash as
four framed parts in order: the path's UTF-8 byte length as an unsigned 8-byte
big-endian integer, the path bytes, the content's byte length as an unsigned
8-byte big-endian integer, and the content bytes. The explicit lengths make the
serialization unambiguous, so distinct trees cannot collide (for example, path
`a` with content `bc` hashes differently from path `ab` with content `c`).
Symbolic links and other non-regular files are excluded: a skill is expected to
be a plain content tree, and content reached only through a symlink is not part
of the digest. Only file
paths and contents contribute, so the `source`/`ref`/`subpath` that locate the
content do not affect the result. For a standalone upload whose `subpath` is
null the serialization covers the whole uploaded skill directory. The client
computes the digest during the same local inspection that resolves the source
and submits it at registration and at import. The server stores the submitted
digest but never recomputes or verifies it: for external sources (git/oci/zip)
it never fetches the content, and for content uploaded directly to MLflow
artifact storage it likewise records the client-asserted value rather than
rehashing the bytes, keeping the digest uniformly client-asserted across every
source type and trusted the same way the client-submitted `name` and member
`description`/`keywords` are.
This mirrors the dataset `digest` concept already in MLflow: a hash that
identifies a version by content within a given skill name, independent of
where the content came from. Because it is content-only, the same bytes reached
through different sources (for example, the same skill fetched directly versus
discovered inside a package) produce the same digest.

The digest is a passive identity field: the registry stores it, returns it on
get, and indexes it, but never uses it to change what a write does. In
particular, import never consults the digest to reuse an existing skill version.
Every discovered member skill gets a new server-assigned version whose `source`
is derived from the newly imported package, so a version always has exactly one
immutable source and a plugin version never contains a member that points into a
different (older) package. This keeps the model coherent with one immutable
source per version, at the cost of a new member version on every re-import even
when a skill's content is unchanged.

The digest's value is entirely on the read side, where it is the join key for
grouping versions by content. Because the digest is client-asserted and not
server-verified, two skill versions with the same digest were asserted by their
registering clients to hold identical content; the registry does not itself
guarantee byte-for-byte identity. A consumer that trusts its own registration
pipeline can therefore treat an equal digest as equal content, for example to
show a clean diff between two agent plugin versions (unchanged members share a
digest even though their versions differ) or to recognize a trace recorded
before a re-import and one recorded after as exercising the same content. The
registry exposes the field and indexes it so these comparisons are cheap; it does
not itself build any diff or trace-linking feature on top of the field, and this
RFC does not commit to one. When a version's digest is absent, it
simply does not participate in digest grouping. The digest also drives an
integrity check on pull: when a pulled version has a `digest` set, the client
recomputes it over the fetched tree and fails on mismatch, and an agent plugin
pull applies the same check per member (see Pull semantics). This is the one
place the digest is recomputed after registration, and it stays client-side:
the check compares fetched content against the client-asserted identity and is
not a registry guarantee.

**MLflow artifact storage (`source_type="mlflow"`).** In addition to
external source pointers, the registry supports storing skill content
directly in MLflow's artifact storage. This serves users who do not
have external Git/OCI infrastructure, who want agent capabilities
stored alongside their models, or who operate in airgapped
environments where external sources are not reachable.

For a standalone MLflow-stored skill, content is stored as a directory tree
of individual files at a unique artifact path the server chooses for each
upload and records in the version's `source`, rather than at a path derived
from the version number. Workspace scoping is handled at the artifact store
level (each workspace has its own artifact root), and the path within the
store is `skills/@<organization>/<name>/<token>/` when the skill has an
organization and `skills/<name>/<token>/` when it does not (the
`@<organization>` segment, along with its slash, is omitted when the
organization is empty); `<token>` is a per-upload identifier, so the path is
never reused across versions or across a delete-and-recreate cycle. The
`source` field holds this path, and get and pull read it rather than deriving
a location by convention. (A skill imported from an MLflow-stored package
instead carries a `source` pointing at the package's artifact tree, with a
`subpath`, as described below.) Pull downloads the directory tree from that
path. The MLflow UI can browse individual files within a stored skill version
when artifact proxying is enabled.

The same unique-path scheme applies to agent plugin content stored
with `source_type="mlflow"`: `agent-plugins/@<organization>/<name>/<token>/`,
with the `@<organization>` segment omitted when there is no organization and
`<token>` a per-upload identifier recorded in the version's `source`.

When a packaged plugin whose content lives in MLflow artifact storage is
imported, its member skills are not copied into per-skill artifact paths.
Each member skill version carries `source_type="mlflow"` with `source` set
to the plugin version's artifact directory (above) and a `subpath` that
locates the skill within that stored package tree. This base pointer is
persisted on the skill version itself at import time, so it is **not**
resolved from the skill's membership:
even after the containing plugin is deleted, the surviving skill still
carries an explicit pointer to the retained tree. Pulling such a skill
fetches only the content under `subpath` from that package tree. Because
several member skills can point into the same stored tree,
the tree is refcounted by the skill versions whose `source` names it and is
retained until the last such skill version is gone. A non-cascade
`delete_agent_plugin` that leaves members therefore also leaves the stored
tree those members pull from, and each surviving member keeps its own
self-contained pointer to it; the tree is reclaimed only when no skill
version still references it.

**Client-side upload flow.** When `source` is a local path (detected
by the absence of a `://` scheme), the skill content is stored in MLflow
artifact storage as part of registration rather than treated as a remote
pointer. Because a skill is small (a `SKILL.md` and a handful of files, not
large model artifacts), registration is a single request that carries
the content, so there is no separate upload phase:

1. The client validates the local directory and, during local inspection,
   reads `name` and the other content-derived fields and computes the content
   digest.
2. The client submits the registration request carrying the packaged local
   content together with that metadata. The server writes the content to a
   unique artifact path it chooses for this upload
   (`skills/@<organization>/<name>/<token>/`, with the `@<organization>` segment
   omitted when there is no organization and `<token>` a server-generated
   identifier unique to this upload), allocates the version number as
   `MAX(version) + 1` (see Version ordering), and commits the `SkillVersion` row
   referencing that path, with `source_type="mlflow"`, `source` set to the stored
   artifact path (a standalone upload has no `subpath`; the whole stored tree is
   the skill), the client-asserted `digest`, and `status` set to the caller's
   requested final status (`active` by default, or `draft` when the caller
   explicitly registers a draft). If a concurrent registration already took that
   number, the insert collides on the `(workspace, organization, name, version)`
   primary key; the server recomputes `MAX(version) + 1` and retries the commit
   without rewriting the already-stored content.
3. Committing the version row only after its content is stored means a failure
   never leaves a content-less or partially uploaded version, and creates no
   registry row to clean up. Because the content path is unique per upload, a
   failed attempt's bytes sit at a path no committed version references and can
   never be mixed into another version. Because content is reclaimed only by
   hard-deleting the version that owns the path (see Version ordering) and never
   by scanning for unreferenced paths, reclamation never races an in-flight
   upload and never removes a path before its row commits; the rare bytes left by
   a crashed upload are intentionally retained, a small leak that is never a
   correctness concern.

Because the version is created only once its content is in place, there is no
draft-until-uploaded state and no finalize step: a version is complete the
instant it exists, so latest, alias, pull, and content resolution need no
special handling for in-flight content. A deliberately registered `draft` is a
complete version marked draft, not an upload-in-progress marker: it is
retrievable and pullable by explicit version, is a valid alias target, and
follows the ordinary latest-resolution rules for a `draft` (never preferred
while an `active` version exists; see Per-version status).

**Version uniqueness.** The combination of
`(workspace, organization, name, version)` is unique. A skill
version represents a single logical version of a capability;
`source` describes where to find it but is not part of its
identity.

**Immutability contract.** `source` and `version` are immutable
after creation. To point to different content, register a new
version. Mutable fields (`status`, `tags`) can be updated
independently.

### AgentPlugin entity

An agent plugin is the stable governed identity for an open Agent Plugins
package. It follows the same top-level MLflow pattern as Skill while deriving
its canonical name and version metadata from immutable manifests.

```python
@dataclass
class AgentPlugin:
    name: str
    organization: str = ""
    description: str | None = None
    icons: list[RegistryIcon] | None = None  # mutable user-defined icons; API returns null when unset
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, str] = field(default_factory=dict)  # read-only; manifest version targets
    latest_version: str | None = None  # read-only, Agent Plugin resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`AgentPlugin.status` is read-only and comes from the same latest-resolved
version as `latest_version`. Both are `None` when the plugin has no
non-`deleted` version. Parent `description` and `icons` are mutable MLflow
presentation metadata. When `description` is unset, the UI may fall back to
the resolved `plugin_json["description"]`; the API returns the parent value
as stored. `icons` has no fallback (the Agent Plugins manifest defines no
icon field): the UI shows its default plugin glyph when unset.

### AgentPluginVersion entity

A versioned snapshot of the canonical package manifest, source authority,
lifecycle state, and optional registered-skill membership. In this RFC, all
members are skills.

Agent plugin members are referenced by URI string rather than a separate
data class. The URI format follows MLflow's `models:/name/version`
convention:

- `skills:/name` resolves to the skill's latest `active` version at creation (never a `draft` or `deprecated`; the create fails if there is no `active` version)
- `skills:/name/version` pins a specific version
- `skills:/name@alias` resolves through an alias

All three forms are resolved to a concrete `member_version` at create time and
frozen into the member row (see the member-list URI resolution note below).

```python
@dataclass
class AgentPluginVersion:
    name: str
    version: str
    organization: str = ""
    plugin_json: dict[str, Any] = field(default_factory=dict)
    source: GitSource | OCISource | ZipSource | str | None = None  # plain str for mlflow content
    source_type: SkillSourceType | None = None

    status: SkillStatus = SkillStatus.ACTIVE
    tags: dict[str, str] = field(default_factory=dict)
    skills: list[str] = field(default_factory=list)
    aliases: list[str] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

### AgentPluginVersion field details

**Version uniqueness.** The combination of
`(workspace, organization, name, version)` is unique. `name` and `version` are
extracted from the canonical payload, and an explicitly supplied path name must
match `plugin_json["name"]`.

**Canonical manifest.** `plugin_json` is validated against the fields MLflow
requires, using `extra="allow"` to accept and preserve unknown fields. The
`$schema` identifier is not gated on a specific version. A non-object
`extensions` field generates a nonfatal warning, remains preserved in the
stored payload, and receives no MLflow semantics. The payload is immutable
after creation.

**Version validation.** When `plugin_json["version"]` is present, MLflow
validates it as SemVer. Semverish inputs (e.g., `1.0`) are normalized to full
SemVer (e.g., `1.0.0`). Non-SemVer version strings are rejected. When the
manifest does not include a version, the user must supply one explicitly.
MLflow inserts the supplied version into the stored payload. In every stored
entity, `version == plugin_json["version"]`.

**Manifest synthesis.** For an assembled plugin, both registration and
low-level version creation may receive `plugin_json=None`. The user must
supply a `version` in this case. The server-side registry layer synthesizes
a minimal valid manifest using the parent name and supplied version before
storing the entity. Packaged import always supplies
a complete manifest after validation or adapter translation. Persistence never
stores a null or incomplete canonical payload.

**Latest resolution.** Active candidates take precedence. Semantic precedence
selects latest. Creation time breaks ties (e.g., versions differing only in
build metadata). The same rule applies to non-deleted non-active candidates
when no active version exists.

**Agent plugin-level source and kind.** An agent plugin version is either
packaged or assembled, never both. Kind is derived from the persisted
`source_type`: a version whose `source_type` is `git`, `oci`, `zip`, or
`mlflow` is **packaged**; a version whose `source_type` is `assembled`
is **assembled**. Kind is resolved per version and is not required to be
consistent across versions of the same agent plugin, so the server accepts a
new version whose derived kind differs from existing versions. Every consumer
of kind (pull, latest-resolution, member sourcing) reads the kind of the
specific version it operates on.

- **Packaged:** has its own package that contains the complete agent
  plugin, either an external typed source (`source_type` of `git`, `oci`,
  or `zip`) or a package tree uploaded to MLflow artifact storage
  (`source_type="mlflow"`). `pull` fetches the whole agent plugin as a
  unit, including `mcp.json` and any other non-skill content. On import,
  each member skill version is created with a source **derived** from the
  package: the same `source_type` and `source` as the package, with a
  `subpath` locating the skill within it (for an mlflow-stored package the
  member's `source` is set to the plugin's artifact tree and persisted on
  the skill version, so it resolves without reference to membership). Each
  member is therefore an ordinary, independently addressable skill that
  `pull` can fetch on its own without pulling the whole plugin.
- **Assembled:** has no plugin-level package; its content is defined
  entirely by individual member references (`source_type="assembled"`).
  Each skill member is pulled by its own `source_type` (an external source,
  or the member's stored artifact path for an `mlflow` member). If a
  member's content cannot be resolved, `pull` fails rather than producing a
  partial local agent plugin.

The server sets the plugin version's `source_type` from the creation flow,
not from a user-supplied value: a package source (external pointer or
MLflow upload) yields a packaged version, and a version created from member
references with no plugin-level package yields `source_type="assembled"`.
A packaged plugin has a plugin-level source and members whose sources are
derived from (and therefore subpaths of) that same package; an assembled
plugin has no plugin-level source and members that each carry their own
independent source. What a plugin version cannot do is combine a
plugin-level source with members whose sources are independent of that
package. In an assembled agent plugin, every member is an independently
resolvable skill carrying its own `source_type`, whether an external pointer
(`git`, `oci`, `zip`) or MLflow-stored content (`mlflow`, resolved by
identity); the API rejects only the disallowed combination above, a
plugin-level source together with members whose content is independent of
that package.

**Immutability contract.** `plugin_json`, the member list, and source fields of
an agent plugin version are immutable after creation. To change the canonical
manifest, members, or source pointer, register a new agent plugin version.
Mutable fields (`status`, `tags`) can be updated independently.

The low-level registry API validates the manifest but does not fetch a remote
artifact to validate its layout. Client-side import performs format-specific
filesystem discovery and containment validation before registration.

A member can appear in multiple agent plugins and multiple agent plugin versions.
Membership is at the version level, so an agent plugin version is a
reproducible record of "these specific skill versions work together." The
membership record is preserved even if a member is later withdrawn; a
soft-deleted member withdraws the containing plugin version from
resolution, discovery, and pull (see Deletion semantics), while a
`deprecated` member still resolves and pulls.

### Skill and Agent Plugin URI formats

Skill URIs are used for CLI target identification and agent plugin
member lists, following the `models:/name/version` convention
established by MLflow's Model Registry. The Python SDK and REST
API continue to use separate `name`, `organization`, `version`,
and `alias` parameters for primary resource identification.

| Pattern | Meaning | Example |
|---------|---------|---------|
| `skills:/name` | Skill with no organization | `skills:/code-review` |
| `skills:/name/version` | Pin a specific version | `skills:/code-review/1` |
| `skills:/name@alias` | Resolve through an alias | `skills:/code-review@production` |
| `skills:/@org/name` | Skill with organization | `skills:/@acme/code-review` |
| `skills:/@org/name/version` | Pin version with organization | `skills:/@acme/code-review/1` |
| `skills:/@org/name@alias` | Alias with organization | `skills:/@acme/code-review@production` |

**URI disambiguation.** An organization is always marked by a leading
`@` on the first segment (e.g., `skills:/@acme/...`), and names cannot
begin with `@`, so the organization is unambiguous whether or not it is
present. After the optional `@organization` segment, the segments are the
name followed by an optional version, and a trailing `@alias` selects an
alias instead of a version. For example, `skills:/code-review/1` is name
`code-review` at version `1`, and `skills:/@acme/code-review/1` is that
version within organization `acme`.

Agent plugins use a separate URI scheme because their versions are
SemVer strings rather than integers. Organization uses the same leading
`@` marker and is omitted when empty, so the parser identifies the
organization by the `@` marker rather than by inspecting the version:

| Pattern | Meaning | Example |
|---------|---------|---------|
| `agent-plugins:/name` | Agent plugin parent (no org) | `agent-plugins:/pr-workflow` |
| `agent-plugins:/@org/name` | Agent plugin parent (with org) | `agent-plugins:/@acme/pr-workflow` |
| `agent-plugins:/name/version` | Exact version (no org) | `agent-plugins:/pr-workflow/1.0.0-beta.11` |
| `agent-plugins:/@org/name/version` | Exact version (with org) | `agent-plugins:/@acme/pr-workflow/1.0.0` |
| `agent-plugins:/name@alias` | Resolve through an alias | `agent-plugins:/pr-workflow@production` |
| `agent-plugins:/@org/name@alias` | Resolve through an alias (with org) | `agent-plugins:/@acme/pr-workflow@production` |

Agent plugin versions in URIs must be nonempty, valid SemVer, fit within the
database field, and cannot contain `/`, `@`, `#`, or `?`.

In the CLI, commands that operate on an already-registered skill or agent
plugin accept its URI as an optional positional argument, with the equivalent
`--skill-uri` (skills) or `--plugin-uri` (agent plugins) flag also accepted.
For example, `mlflow skills get skills:/code-review` and
`mlflow skills get --skill-uri skills:/code-review` are equivalent.
Identity-establishing commands (`register`, and `agent-plugins create`) use
`--name` (with an optional `--organization`) rather than a URI, since the entity
does not exist yet.
In agent plugin member lists, URIs appear as plain strings in `list[str]`.
The server parses the URI into its constituent fields
(`member_organization`, `member_name`, `member_version`)
for storage and validation. Every reference is resolved to a concrete
`member_version` at create time and that integer is frozen into the member row:
a pinned `skills:/name/version` reference stores the given version, a name-only
`skills:/name` reference resolves to the skill's latest `active` version, and a
`skills:/name@alias` reference resolves to the version the alias points to at
that moment. Name-only member resolution deliberately differs from the dynamic
latest-resolution used by individual `get`/`pull`: because a member reference is
frozen at create time, it resolves only to the latest `active` version and never
freezes a `draft` (unpublished) or `deprecated` (discouraged) version into a
permanent member pin. If the skill has no `active` version, the create request
fails with an error directing the author to publish the skill or pin an explicit
version, rather than silently pinning a draft when the plugin is registered
prematurely. Individual `get`/`pull` keep the standard latest-resolution rule,
which may select a `draft`, precisely because they re-evaluate on each call. An
explicit `skills:/name/version` pin still stores exactly the version given, and
`skills:/name@alias` stores whatever the alias points to, since both are
deliberate author choices. Because the concrete version is captured on creation,
`member_version` is `NOT NULL` and a later change to the skill's latest version
or alias target does not alter existing member rows.

## Store interface

The store interface follows the mixin pattern established by the MCP
Server Registry (RFC-0004). Methods raise `NotImplementedError` rather
than using `@abstractmethod`, allowing stores that do not support
skills (e.g., `FileStore`) to work without stubbing every method.

In the store interface, `delete_*` methods on top-level entities are
hard deletes, while `delete_*_version` methods are soft deletes that
transition the version to `deleted`.

Version-creation methods (`create_skill_version`,
`create_agent_plugin_version`) receive the server-resolved `source_type`
as a flat field. The server resolves it per the field-inference rules
(validating a submitted type, inferring when none is submitted) before
calling the store, and the store persists it without re-deriving.
At this layer `source_type` is the resolved value, not a raw client
claim; the client-facing `MlflowClient` and `mlflow.genai` methods take
only `source`, whose typed class carries the explicit type when there is
one.

```python
from mlflow.store.tracking import SEARCH_MAX_RESULTS_DEFAULT


NOT_SET = object()


class SkillRegistryMixin:
    # Methods raise NotImplementedError rather than using @abstractmethod,
    # following the GatewayStoreMixin pattern. This allows stores that don't
    # support skills (e.g., FileStore) to work without stubbing every method.

    # --- Skill operations ---

    def create_skill(
        self, name: str,
        organization: str = "",
        description: str | None = None,
        icons: list[RegistryIcon] | None = None,
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill(
        self, name: str, organization: str = "",
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def search_skills(
        self,
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill(
        self,
        name: str,
        organization: str = "",
        description: str | None = NOT_SET,
        icons: list[RegistryIcon] | None = NOT_SET,
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill(
        self, name: str, organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillVersion operations ---

    def create_skill_version(
        self,
        name: str,
        organization: str = "",
        source_type: str | None = None,
        source: str | None = None,
        ref: str | None = None,
        subpath: str | None = None,
        digest: str | None = None,
        status: str = "active",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version(
        self, name: str, version: int,
        organization: str = "",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version_by_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_latest_skill_version(
        self, name: str, organization: str = "",
    ) -> SkillVersion:
        """Raises RESOURCE_DOES_NOT_EXIST when the skill has no
        non-deleted version that resolves as latest."""
        raise NotImplementedError(self.__class__.__name__)

    def search_skill_versions(
        self,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill_version(
        self,
        name: str,
        version: int,
        organization: str = "",
        status: SkillStatus | None = NOT_SET,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version(
        self, name: str, version: int,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill tag operations ---

    def set_skill_tag(
        self, name: str, key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_tag(
        self, name: str, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def set_skill_version_tag(
        self, name: str, version: int,
        key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version_tag(
        self, name: str, version: int, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill alias operations ---

    def set_skill_alias(
        self, name: str, alias: str, version: int,
        organization: str = "",
    ) -> None:
        """Raises RESOURCE_DOES_NOT_EXIST when the target version does
        not exist or is deleted, so an alias can never dangle. Any other
        status (draft, active, deprecated) is a valid target."""
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin operations ---

    def create_agent_plugin(
        self, name: str,
        organization: str = "",
        description: str | None = None,
        icons: list[RegistryIcon] | None = None,
    ) -> AgentPlugin:
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin(
        self, name: str, organization: str = "",
    ) -> AgentPlugin:
        raise NotImplementedError(self.__class__.__name__)

    def search_agent_plugins(
        self,
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPlugin]:
        raise NotImplementedError(self.__class__.__name__)

    def update_agent_plugin(
        self,
        name: str,
        organization: str = "",
        description: str | None = NOT_SET,
        icons: list[RegistryIcon] | None = NOT_SET,
    ) -> AgentPlugin:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin(
        self, name: str, organization: str = "", cascade: bool = False,
    ) -> None:
        """Hard-delete the agent plugin and its versions, tags, aliases, and
        membership rows. With cascade=False (default) member skills are left in
        place as ordinary standalone skills. With cascade=True the plugin's
        member skills are also hard-deleted, subject to the same
        referential-integrity checks as any skill hard delete; the checks run
        before anything is removed, and if any member is still referenced by a
        live (non-`deleted`) plugin version other than the one being deleted the
        entire cascade delete fails and nothing is removed. Membership rows held
        only by soft-deleted plugin versions do not block the cascade and are
        purged along with the member."""
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPluginVersion operations ---

    def create_agent_plugin_version(
        self,
        name: str,
        organization: str = "",
        version: str | None = None,
        plugin_json: dict[str, Any] | None = None,
        skills: list[str] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        ref: str | None = None,
        subpath: str | None = None,
        status: str = "active",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def import_agent_plugin(
        self,
        name: str,
        organization: str = "",
        version: str | None = None,
        plugin_json: dict[str, Any] | None = None,
        member_skills: list[dict[str, Any]] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        ref: str | None = None,
        subpath: str | None = None,
        status: str = "active",
    ) -> tuple[AgentPluginVersion, list[SkillVersion]]:
        """Register a packaged agent plugin as a single unit of work. For each
        entry in member_skills (a name, a subpath locating the skill within the
        package, a client-computed content digest, and optional description and
        keywords), create the Skill when the name is free or add a version when
        the name has previously been a member of this same plugin (derived from
        this plugin's member rows), set search_text, and create a member
        SkillVersion whose source is derived from the package (the package's
        source_type and source, with the entry's subpath) and whose digest is
        the entry's digest. Import never reuses an existing skill version on the
        strength of a matching digest: every discovered member gets a new
        version so that each version keeps exactly one immutable, package-derived
        source (see Content digest). Then create the packaged AgentPluginVersion
        referencing every member skill version. If a member name is already taken
        in the organization by a skill that is not part of this plugin's history,
        the whole import fails. All of it commits in one database transaction or
        none of it does. A store that cannot provide a single-transaction unit
        of work raises NotImplementedError rather than applying the import
        partially. Returns the packaged AgentPluginVersion and the member
        SkillVersions it created."""
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin_version(
        self, name: str, version: str,
        organization: str = "",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin_version_by_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_latest_agent_plugin_version(
        self, name: str, organization: str = "",
    ) -> AgentPluginVersion:
        """Raises RESOURCE_DOES_NOT_EXIST when the plugin has no
        non-deleted version (nothing resolves as latest)."""
        raise NotImplementedError(self.__class__.__name__)

    def search_agent_plugin_versions(
        self,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPluginVersion]:
        raise NotImplementedError(self.__class__.__name__)

    def update_agent_plugin_version(
        self,
        name: str,
        version: str,
        organization: str = "",
        status: SkillStatus | None = NOT_SET,
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_version(
        self, name: str, version: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin tag operations ---

    def set_agent_plugin_tag(
        self, name: str, key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_tag(
        self, name: str, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def set_agent_plugin_version_tag(
        self, name: str, version: str,
        key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_version_tag(
        self, name: str, version: str, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin alias operations ---

    def set_agent_plugin_alias(
        self, name: str, alias: str, version: str,
        organization: str = "",
    ) -> None:
        """Raises RESOURCE_DOES_NOT_EXIST when the target version does
        not exist, is deleted, or is withdrawn because it contains a
        deleted member, so an alias can never dangle or bypass the kill
        switch. Any other status (draft, active, deprecated) is a valid
        target."""
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)
```

The alias name `latest` is reserved for both skills and agent plugins.
The corresponding `set_*_alias()` methods reject it. Alias lookup with
`latest` delegates to `get_latest_skill_version()` or
`get_latest_agent_plugin_version()` rather than reading a stored alias
row.

For update fields, omitting a parameter leaves the stored value
unchanged, while passing `None` to a nullable field explicitly sets
the field to `null`.

All store methods that identify a specific entity use `name` and
`organization` (with `organization` defaulting to `""` when
omitted). Workspace scoping is handled implicitly by the store
implementation based on the caller's context.

## SDK convenience functions

The SDK is split into two layers, following the MLflow model and
prompt registries. The `mlflow.genai` namespace exposes a small set
of high-level, workflow-oriented functions for the common cases
(register, import, introspect, pull, search). The full low-level
create/get/update/delete surface, including version, tag, and alias
operations, lives on `MlflowClient`; the `mlflow.genai` helpers call
these methods internally. This keeps the top-level namespace concise
while still making every operation reachable. The two search
functions appear in both layers, mirroring
`mlflow.search_registered_models` and
`MlflowClient.search_registered_models` in the model registry.
`import_agent_plugin` likewise appears in both layers, but as a division of
labor rather than a passthrough: the `mlflow.genai` function fetches the source,
detects the format, and translates it, then calls the
`MlflowClient.import_agent_plugin` primitive that posts the prepared payload to
`POST /import`. (`introspect_agent_plugin` is top-level only, since it writes
nothing to the registry.)

### High-level workflow functions (`mlflow.genai`)

```python
from dataclasses import dataclass

import mlflow


def register_skill(
    *,
    name: str | None = None,
    organization: str = "",
    source: GitSource | OCISource | ZipSource | str | None = None,
    status: str = "active",
) -> SkillVersion:
    """Register a skill version. The server assigns the next
    monotonic integer version. Auto-creates the parent Skill if
    it does not exist (with null description) and otherwise reuses
    the existing parent; fails if the name is already taken by a
    member of a packaged plugin. To set parent-level metadata, use
    MlflowClient.create_skill() before registering versions or
    MlflowClient.update_skill() afterward. This matches the MCP
    Server Registry behavior
    (register_mcp_server). If name is omitted, the client
    extracts it from the skill's SKILL.md entry point during local
    inspection and submits it; the server never fetches the source
    to infer content-derived fields. The client also computes the
    content digest from SKILL.md and the files under subpath during
    that same inspection and submits it whether or not name was
    supplied. The server
    resolves source_type: a typed source class (GitSource, OCISource,
    ZipSource) is converted to flat REST fields including an explicit
    source_type, which the server validates against the source value;
    a plain string is also accepted for convenience and passed as the
    source field with type inferred by the server, and a plain string
    whose type cannot be inferred is rejected with an error directing
    the caller to a typed source class. The creation flow sets mlflow
    when the client uploads a local path.
    If source is a local path (no :// scheme), the client submits the
    packaged content with the registration request and the server stores
    it and creates the version in one atomic operation (no separate
    upload or finalize step; see Client-side upload flow)."""


def search_skills(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[Skill]: ...


def register_agent_plugin(
    *,
    plugin_json: dict[str, Any] | None = None,
    name: str | None = None,
    version: str | None = None,
    organization: str = "",
    skills: list[str] | None = None,
    source: GitSource | OCISource | ZipSource | str | None = None,
    status: str = "active",
) -> AgentPluginVersion:
    """Validate or synthesize a canonical manifest, create or reuse the
    parent AgentPlugin, and register one immutable version. version is
    required when plugin_json is None or does not contain a version."""


def search_agent_plugins(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[AgentPlugin]: ...


@dataclass(frozen=True)
class PluginImportWarning:
    category: str
    path: str
    message: str


@dataclass(frozen=True)
class IntrospectedSkill:
    name: str
    path: str


@dataclass
class PluginIntrospectionResult:
    detected_format: str
    plugin_name: str | None
    plugin_json: dict[str, Any]
    skills: list[IntrospectedSkill]
    recognized_unregistered_content: list[str]
    warnings: list[PluginImportWarning]


@dataclass
class PluginImportResult:
    detected_format: str
    plugin_version: AgentPluginVersion
    skill_versions: list[SkillVersion]
    recognized_unregistered_content: list[str]
    warnings: list[PluginImportWarning]


def introspect_agent_plugin(
    *,
    source: GitSource | OCISource | ZipSource | str,
) -> PluginIntrospectionResult:
    """Inspect a local or remote plugin without modifying the registry."""


def import_agent_plugin(
    *,
    source: GitSource | OCISource | ZipSource | str,
    plugin_name: str | None = None,
    version: str | None = None,
    organization: str = "",
) -> PluginImportResult:
    """Import a plugin as a packaged agent plugin.

    Auto-detects Agent Plugins, Claude Code, or generic skill-directory input;
    validates or constructs canonical plugin_json; discovers and registers
    skills, each with a source derived from the package (the package's
    source_type and source, plus a subpath locating the skill); and preserves
    the package source. Recognized mcp.json content is reported but not
    registered. plugin_name is used only when the selected adapter cannot
    derive a name. version is required when the detected manifest does not
    contain a version field. A typed source class (GitSource, OCISource,
    ZipSource) is converted to flat REST fields; a plain string is also
    accepted for convenience. Fetching and inspection are client-side; the
    member skills and the plugin version are then created atomically by the
    server via the POST /import endpoint. The `ImportRegisterResponse` carries
    the packaged plugin version and the member skill versions (created or
    created), which populate `PluginImportResult.plugin_version` and
    `skill_versions` without additional reads.
    """


def pull(
    *,
    name: str | None = None,
    organization: str = "",
    entity_type: str = "skill",
    version: int | str | None = None,
    alias: str | None = None,
    destination: str = ".",
) -> str:
    """Pull skill or agent plugin content from registered sources to a
    local directory. Set entity_type to 'skill' or 'agent_plugin'."""


# Example usage:
version = mlflow.genai.register_skill(
    name="code-review",
    source=GitSource(url="https://github.com/acme/skills.git", ref="v1.0.0"),
)
# version.name and version.organization identify the skill for subsequent operations
skills = mlflow.genai.search_skills(filter_string="status = 'active'")
```

### Low-level CRUD (`MlflowClient` methods)

The full create/get/update/delete surface, including version, tag,
and alias operations, is exposed as `MlflowClient` methods. The
`mlflow.genai` helpers above call these internally. `search_skills`
and `search_agent_plugins` appear in both layers, mirroring
`search_registered_models` in the model registry.

```python
class MlflowClient:
    # --- Skills ---
    def create_skill(
        self, *, name: str, organization: str = "", description: str | None = None,
        icons: list[RegistryIcon] | None = None,
    ) -> Skill: ...

    def get_skill(self, *, name: str, organization: str = "") -> Skill: ...

    def search_skills(
        self,
        *,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]: ...

    def update_skill(
        self, *, name: str, organization: str = "", description: str | None = NOT_SET,
        icons: list[RegistryIcon] | None = NOT_SET,
    ) -> Skill: ...

    def delete_skill(self, *, name: str, organization: str = "") -> None: ...

    def create_skill_version(
        self,
        *,
        name: str,
        organization: str = "",
        source: GitSource | OCISource | ZipSource | str | None = None,
        digest: str | None = None,
        status: str = "active",
    ) -> SkillVersion: ...

    def get_skill_version(
        self, *, name: str, version: int, organization: str = ""
    ) -> SkillVersion: ...

    def get_skill_version_by_alias(
        self, *, name: str, alias: str, organization: str = ""
    ) -> SkillVersion: ...

    def get_latest_skill_version(
        self, *, name: str, organization: str = ""
    ) -> SkillVersion: ...

    def search_skill_versions(
        self,
        *,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]: ...

    def update_skill_version(
        self, *, name: str, version: int, organization: str = "", status: str | None = NOT_SET
    ) -> SkillVersion: ...

    def delete_skill_version(
        self, *, name: str, version: int, organization: str = ""
    ) -> None: ...

    # --- Agent plugins ---
    def create_agent_plugin(
        self, *, name: str, organization: str = "", description: str | None = None,
        icons: list[RegistryIcon] | None = None,
    ) -> AgentPlugin: ...

    def create_agent_plugin_version(
        self,
        *,
        name: str,
        organization: str = "",
        version: str | None = None,
        plugin_json: dict[str, Any] | None = None,
        skills: list[str] | None = None,
        source: GitSource | OCISource | ZipSource | str | None = None,
        status: str = "active",
    ) -> AgentPluginVersion:
        """version is required when plugin_json is None or when
        plugin_json does not contain a version field."""

    def import_agent_plugin(
        self,
        *,
        name: str,
        organization: str = "",
        version: str | None = None,
        plugin_json: dict[str, Any] | None = None,
        member_skills: list[dict[str, Any]] | None = None,
        source: GitSource | OCISource | ZipSource | str | None = None,
        status: str = "active",
    ) -> tuple[AgentPluginVersion, list[SkillVersion]]:
        """Post a client-prepared import payload to the transactional
        import-registration endpoint. The server creates the member skills
        and the packaged plugin version atomically; source_type is server-set
        and each member skill's source is derived from the package. Each
        member_skills entry carries a name, a subpath locating the skill within
        the package, a client-computed content digest, and optional description
        and keywords, read from its SKILL.md during local inspection. Returns
        the packaged AgentPluginVersion and the member SkillVersions (all newly
        created) parsed from the ImportRegisterResponse."""

    def get_agent_plugin(self, *, name: str, organization: str = "") -> AgentPlugin: ...

    def search_agent_plugins(
        self,
        *,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPlugin]: ...

    def update_agent_plugin(
        self, *, name: str, organization: str = "", description: str | None = NOT_SET,
        icons: list[RegistryIcon] | None = NOT_SET,
    ) -> AgentPlugin: ...

    def delete_agent_plugin(
        self, *, name: str, organization: str = "", cascade: bool = False
    ) -> None: ...

    def get_agent_plugin_version(
        self, *, name: str, version: str, organization: str = ""
    ) -> AgentPluginVersion: ...

    def get_agent_plugin_version_by_alias(
        self, *, name: str, alias: str, organization: str = ""
    ) -> AgentPluginVersion: ...

    def get_latest_agent_plugin_version(
        self, *, name: str, organization: str = ""
    ) -> AgentPluginVersion: ...

    def search_agent_plugin_versions(
        self,
        *,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPluginVersion]: ...

    def update_agent_plugin_version(
        self, *, name: str, version: str, organization: str = "", status: str | None = NOT_SET
    ) -> AgentPluginVersion: ...

    def delete_agent_plugin_version(
        self, *, name: str, version: str, organization: str = ""
    ) -> None: ...

    # --- Tags and aliases ---
    def set_skill_tag(self, *, name: str, key: str, value: str, organization: str = "") -> None: ...

    def delete_skill_tag(self, *, name: str, key: str, organization: str = "") -> None: ...

    def set_skill_version_tag(self, *, name: str, version: int, key: str, value: str, organization: str = "") -> None: ...

    def delete_skill_version_tag(self, *, name: str, version: int, key: str, organization: str = "") -> None: ...

    def set_skill_alias(self, *, name: str, alias: str, version: int, organization: str = "") -> None: ...

    def delete_skill_alias(self, *, name: str, alias: str, organization: str = "") -> None: ...

    def set_agent_plugin_tag(self, *, name: str, key: str, value: str, organization: str = "") -> None: ...

    def delete_agent_plugin_tag(self, *, name: str, key: str, organization: str = "") -> None: ...

    def set_agent_plugin_version_tag(self, *, name: str, version: str, key: str, value: str, organization: str = "") -> None: ...

    def delete_agent_plugin_version_tag(self, *, name: str, version: str, key: str, organization: str = "") -> None: ...

    def set_agent_plugin_alias(self, *, name: str, alias: str, version: str, organization: str = "") -> None: ...

    def delete_agent_plugin_alias(self, *, name: str, alias: str, organization: str = "") -> None: ...
```

For SDK update methods, `NOT_SET` means "leave unchanged" while `None`
means "clear this nullable field". This mirrors the store-layer update
contract so callers can distinguish partial updates from explicit
nulling.

`pull` is implemented in the SDK/CLI layer, not the store mixin. The
client calls `get_skill_version` (or resolves an alias) to obtain the
version's `source_type` and source pointer, then routes on `source_type`:
`git` clone, `oci` pull, `zip` download, or `mlflow` artifact-tree
download. For `mlflow`, the artifact base is always the version's `source`: the
server-stored upload path for a standalone skill, or a package tree pointer
(with `subpath`) for an imported member. A skill
created by importing a packaged plugin pulls through this same routing: it
carries a derived `source` (and `subpath`) persisted on the version, so the
client fetches just that skill's content without pulling the whole plugin and
without consulting any membership row.
For agent plugin pulls, the stored `plugin.json` manifest is always
written to the destination root. Assembled plugin members are placed
under `skills/<member-name>/` to match the Agent Plugins `skills/*/SKILL.md`
discovery layout, making the result a conformant Agent Plugins package.
This keeps the store as a pure data-access layer.

## REST API

The REST API is implemented as a FastAPI router using RESTful nested
resource paths. It is exposed under both `/api/3.0/mlflow/skills` and
`/ajax-api/3.0/mlflow/skills`, plus the corresponding static-prefix
variants, following the MCP Server Registry (RFC-0004) pattern.

**Field inference.** Deriving a value from source *content* requires reading
the bytes, and the registry server never fetches a user-supplied source URL.
Every content-derived field is therefore produced client-side during local
inspection and submitted with the request; the server derives only from data it
already holds (the submitted payload, the source pointer string, and bytes
uploaded directly to it). This is the same rule the plugin import flow follows,
and it keeps fetching of untrusted URLs off the server.

A skill's `name` (and, for a package member, its `description` and `keywords`)
is read from SKILL.md by the client (SDK or CLI) during inspection and
submitted. The simplest
invocation still omits the name because the client fills it in, whether the
source is a local path it is about to upload or a remote git/oci/zip pointer it
resolves with the user's own credentials. Because the server never reads
content to derive it, `name` is always client-supplied; a raw REST caller that
omits it is rejected with an error indicating that `name` must be provided
explicitly. `digest` is computed by the
client from the resolved content during the same inspection and submitted (see
Content digest); the server stores the client-asserted value and does not
recompute or verify it, for uploaded content or external pointers alike.

The server sets the stored `source_type`, and setting it needs no access to
the content. For external pointers the request may carry an explicit
`source_type` (`git`, `oci`, or `zip`): the CLI's `register
git`/`oci`/`zip` subcommands and the SDK's typed source classes always
know the type and submit it, so information the caller has already
expressed is not discarded and then guessed at. The server validates an
explicit type against the source value and rejects a contradiction with
an unambiguous scheme (`source_type="git"` with an `oci://` source, for
example). When no explicit type is submitted (a plain-string source, or
a raw REST caller), the server infers it from the `source` value: `.git`
suffix or `git://` scheme = git, `oci://` scheme = oci, `.zip` = zip.
An external pointer that matches none of these and carries no explicit
type is rejected with an error directing the caller to supply
`source_type` (or use a typed source class or CLI subcommand), rather
than misclassified; this matters because common source URLs carry no
distinguishing shape (an HTTPS Git clone URL without the `.git` suffix,
an archive URL without a `.zip` path ending). The
`oci://` scheme is a hint only: the server records
`source_type="oci"` and persists the bare image reference without the
scheme, so `pull` and `introspect` operate on the native OCI reference.
For content
without an external pointer the server sets it from the endpoint used, not from
the submitted `source` value: a standalone skill version uploaded through the
ordinary creation APIs with a null submitted `source` becomes `mlflow` (content
stored in MLflow artifact storage). The import-registration endpoint derives each
member skill's source from the package: the member skills take the package's
`source_type` and `source` with a `subpath` locating the skill, so an import
from a git/oci/zip package yields members of that same source type, and an
import of a package stored in MLflow artifact storage yields `mlflow` members
whose `source` is set to the plugin's artifact tree and persisted on each
member. In every case `source_type` is fixed by the creation flow, not by
whether the submitted `source` was null: a standalone upload submits a null
`source`, and the server then records the artifact path it wrote the content to,
so a committed `mlflow` version always carries a non-null stored `source`. An agent plugin version created
through the ordinary creation APIs from member references with no plugin-level
package becomes `assembled`. Because an ordinary agent plugin request
that carries no plugin-level package is `assembled` by definition, a null-`source`
request to the ordinary creation APIs is unambiguous (a skill version is
`mlflow`, an agent plugin version is `assembled`). The `mlflow` and
`assembled` types are therefore never client-supplied: an explicit
`source_type` naming either of them is rejected, since those types are
fixed by the creation flow rather than declared. Keeping the stored
`source_type` server-set (validated when submitted, inferred otherwise)
keeps that discriminator consistent across languages,
and it needs no content, so it does not require the server to fetch anything.
Content-derived fields are produced client-side instead, so the server never
fetches user-supplied URLs; checks that operate on bytes uploaded directly to
MLflow (e.g., signature verification) can still run server-side on that stored
content.

For agent plugins, `POST /register` and version creation validate the submitted
canonical `plugin_json`. The server extracts and checks `name` and validates
the version as SemVer (normalizing semverish values). When `plugin_json` is
absent or does not contain a version, the request must include a `version`
field. When both `plugin_json["version"]` and the request-level `version` are
present, they must agree; a mismatch is rejected. The server does not fetch remote package content. Client-side
`import_agent_plugin()` performs source fetching, format detection, filesystem
validation, and adapter translation, then submits the canonical payload and
member references to the transactional `POST /import` endpoint
(`ImportRegisterRequest`), which creates the member skills and the packaged
plugin version atomically and returns both in an `ImportRegisterResponse` (the
packaged plugin version and the newly created member skill versions), so the
client populates its result without additional reads.

When `source` is a local path, the registration request carries the
packaged content, so `POST /register` and `POST .../versions` accept a
`multipart/form-data` body with two parts:

- a `metadata` part: `application/json` carrying the ordinary
  `CreateSkillVersionRequest` fields (`name`, `organization`, `digest`,
  `status`, and so on). The request model carries metadata only, never raw
  bytes; `source` is omitted or null, which is how the server routes the
  request to the upload flow rather than treating `source` as a remote
  pointer.
- a `content` part: `application/gzip`, a gzip-compressed tar archive of the
  skill directory (the same tree the client hashed for the `digest`).

The server validates the archive before storing it: it rejects an archive
whose entries contain absolute paths or `..` segments. It accepts only regular
files and directories and rejects any other entry type (symbolic links, hard
links, devices, FIFOs, and the like), matching the digest's canonicalization
rules so the stored tree is exactly what was hashed. It also rejects a
decompressed size over
a configurable limit (default 25 MiB, sufficient for text-based skills) to
bound resource use. A malformed or oversize archive fails the registration
with no version created. On success the server unpacks it to the version's
controlled artifact prefix and commits the row (see Client-side upload flow).
Remote-pointer registrations (`git`/`oci`/`zip`) carry no content part and use
an ordinary `application/json` body.

The body and the `source` field must agree, and the server rejects any
mismatched combination rather than guessing intent:

- a null or omitted `source` selects the upload flow and therefore *requires*
  the `multipart/form-data` body with a `content` part; a plain
  `application/json` registration with a null `source` is rejected, so a raw
  REST caller cannot create a content-less version.
- a non-null remote `source` *requires* the `application/json` body; a
  `multipart/form-data` request that supplies both a `content` part and a
  remote `source` is rejected rather than silently ignoring one of them.

There is no separate content-upload
endpoint and no finalize step: the version is created only once its content is
in place.

### Skill endpoints

All paths relative to the logical skills router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create a skill |
| `GET` | `/` | Search skills |
| `POST` | `/register` | Register a skill version (name required at this endpoint; the SDK/CLI fills it from SKILL.md during local inspection so human callers omit it, but a raw REST caller that omits it is rejected; auto-creates parent) |
| `GET` | `/@{organization}/{name}` | Get skill by organization and name |
| `PATCH` | `/@{organization}/{name}` | Update skill fields |
| `DELETE` | `/@{organization}/{name}` | Hard-delete skill (cascades, subject to references) |
| `POST` | `/@{organization}/{name}/versions` | Create a skill version |
| `GET` | `/@{organization}/{name}/versions` | Search versions |
| `GET` | `/@{organization}/{name}/versions/{version}` | Get a specific version |
| `PATCH` | `/@{organization}/{name}/versions/{version}` | Update version status |
| `DELETE` | `/@{organization}/{name}/versions/{version}` | Soft-delete a version (`status='deleted'`) |
| `POST` | `/@{organization}/{name}/tags` | Set a skill-level tag |
| `DELETE` | `/@{organization}/{name}/tags/{key}` | Delete a skill-level tag |
| `POST` | `/@{organization}/{name}/versions/{version}/tags` | Set a version-level tag |
| `DELETE` | `/@{organization}/{name}/versions/{version}/tags/{key}` | Delete a version tag |
| `POST` | `/@{organization}/{name}/aliases` | Set an alias |
| `GET` | `/@{organization}/{name}/aliases/{alias}` | Resolve alias to `SkillVersion` |
| `DELETE` | `/@{organization}/{name}/aliases/{alias}` | Delete an alias |

Each entity-specific row above (those whose path begins
`/@{organization}/{name}`) shows the with-organization route. Every such
operation is exposed as two concrete route templates: the with-organization
form listed and a no-organization form that is the same path with the leading
`@{organization}/` segment removed. For
example, "Get a specific version" is both
`GET /@{organization}/{name}/versions/{version}` (organization present) and
`GET /{name}/versions/{version}` (no organization); "Get skill by
organization and name" is both `GET /@{organization}/{name}` and
`GET /{name}`. There is no placeholder for the empty organization: the
`@{organization}` segment and its slash are omitted entirely. So
`GET /code-review/versions/1` addresses a skill with no organization, and
`GET /@acme/code-review/versions/1` addresses the same-named skill in
organization `acme`.

This mirrors the URI format, which uses the same `@organization` marker and
likewise omits it when empty. The `@` marker is what keeps the paths
unambiguous even though fixed keyword subresource segments (`versions`,
`tags`, `aliases`) follow the name: an organization segment always begins
with `@` and names cannot begin with `@`. So `/acme/versions` can only mean
`(no organization, name=acme, the versions collection)`, never
`(organization=acme, name=versions)`; and conversely `/@acme/versions` can
only mean the skill literally named `versions` in organization `acme`, never
a versions collection.

Because both the with-organization template `/@{organization}/{name}` and the
no-organization template `/{name}/versions` structurally match a two-segment
path, the routing layer must not rely on match order alone: the `{name}`
segment (and the leading no-organization segment) is constrained to reject a
leading `@`, so `/@acme/versions` binds only the organization template and
`/acme/versions` binds only the no-organization template. Splitting each
operation into a with-organization and a no-organization route template is
the cost of omitting the empty-organization segment, but it needs no
placeholder value and no reserved subresource keywords. The artifact storage
paths use the same `@organization` convention.

Similarly, agent plugin endpoints are exposed under both
`/api/3.0/mlflow/agent-plugins` and
`/ajax-api/3.0/mlflow/agent-plugins`.

### Agent plugin endpoints

All paths relative to the logical agent-plugins router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create an agent plugin |
| `GET` | `/` | Search agent plugins |
| `POST` | `/register` | Validate or synthesize `plugin_json`, create or reuse the parent, and create a version |
| `POST` | `/import` | Import a packaged plugin: atomically create the member skills (each with a derived source) and the plugin version from a client-prepared payload (`ImportRegisterRequest`); returns the packaged plugin version and the newly created member skill versions (`ImportRegisterResponse`) |
| `GET` | `/@{organization}/{name}` | Get agent plugin by organization and name |
| `PATCH` | `/@{organization}/{name}` | Update agent plugin fields |
| `DELETE` | `/@{organization}/{name}` | Hard-delete agent plugin (cascades versions, memberships, and aliases). With `cascade=true`, also hard-deletes member skills; the whole delete fails atomically if any member is still referenced by a live (non-`deleted`) plugin version other than the one being deleted (membership rows held only by soft-deleted plugin versions do not block and are purged with the member); without it, member skills are left in place |
| `POST` | `/@{organization}/{name}/versions` | Create an agent plugin version with members |
| `GET` | `/@{organization}/{name}/versions` | Search agent plugin versions |
| `GET` | `/@{organization}/{name}/versions/{version}` | Get a specific agent plugin version |
| `PATCH` | `/@{organization}/{name}/versions/{version}` | Update agent plugin version status |
| `DELETE` | `/@{organization}/{name}/versions/{version}` | Soft-delete an agent plugin version (`status='deleted'`) |
| `POST` | `/@{organization}/{name}/tags` | Set an agent plugin-level tag |
| `DELETE` | `/@{organization}/{name}/tags/{key}` | Delete an agent plugin-level tag |
| `POST` | `/@{organization}/{name}/versions/{version}/tags` | Set an agent plugin version tag |
| `DELETE` | `/@{organization}/{name}/versions/{version}/tags/{key}` | Delete an agent plugin version tag |
| `POST` | `/@{organization}/{name}/aliases` | Set an agent plugin alias |
| `GET` | `/@{organization}/{name}/aliases/{alias}` | Resolve agent plugin alias to version |
| `DELETE` | `/@{organization}/{name}/aliases/{alias}` | Delete an agent plugin alias |

Same `@organization` segment convention as skill endpoints: each
entity-specific row (those beginning `/@{organization}/{name}`) shows the
with-organization route, and every such operation also has a no-organization
route with the leading `@{organization}/` segment removed. Agent plugin
version path values follow the same length and reserved character validation
as Agent Plugin URIs.

### Pagination and filtering

Search endpoints use page-token-based pagination and `filter_string`
expressions following existing MLflow conventions. Free-text discovery is also
expressed through `filter_string`: parent search endpoints expose a derived
`search_text` field that is matched with `LIKE`/`ILIKE` (e.g.,
`search_text LIKE '%review%'`). This field concatenates several
discovery fields into one column so that a single `LIKE` matches across
all of them without requiring `OR`, which MLflow `filter_string` does not
support. For skills the projection is derived from name and description; for
agent plugins it is derived from the fields listed below.

The two entity kinds store `search_text` in different places. A skill's
discovery fields (name, description) all live on the parent row, so `search_text`
is a column on the `skills` parent table. An agent plugin's discovery fields
come mostly from the manifest, which is version-scoped, so there is no
`search_text` column on the `agent_plugins` parent; the column lives on
`agent_plugin_versions`, and parent search joins to the latest-resolved version
and matches that version's `search_text`. This is why a lifecycle transition
that changes which version resolves as latest can change the parent's free-text
matches. Because a version's `search_text` also folds in the mutable parent
description, updating that description recomputes `search_text` across the
plugin's version rows, so parent search stays consistent regardless of which
version later resolves as latest.

**Skills:** the `search_text` field covers name and description. Structured
examples include `name LIKE '%review%'`, `description LIKE '%security%'`,
`organization = 'acme'`, `status = 'active'`, and `tags.team = 'platform'`.

**Agent plugins:** the `search_text` field covers name, mutable parent
description, organization, and the latest-resolved manifest's description,
keywords, and author name. Structured filters are the same as Skills, plus
`member_name = 'code-review'` to find agent plugins that include a given skill.
Unlike the other parent filters, which resolve against the latest version,
`member_name` matches a plugin when any of its versions lists that member, not
only the latest-resolved one. This makes it the discovery path for "which
plugins depend on this skill," including plugins pinned to an older version, now
that memberships are not a stored field on the skill.

On parent search, a `status` filter matches the parent's derived status,
resolved from the same latest-resolved version that drives `latest_version`
and the entity's read-only `status`, since parent tables have no status
column of their own. A parent with no non-`deleted` version has a `None`
derived status and is excluded by any `status` equality filter. To filter on
the status of a specific version, use version search.

**Versions (all entity types):** `status = 'active'`,
`organization = 'acme'`, `source_type = 'git'`,
`tags.approved = 'true'`, and, for skill versions, `digest = '<hex>'`
to find all versions of a skill that share a given content digest.

Manifest keywords are not copied into MLflow tags. A derived version-level
search projection supports portable free-text matching while `plugin_json`
remains canonical.

### Request and response models

Version-creation requests include immutable creation payloads and mutable
initial status; later update requests contain only mutable fields. Resource
identifiers normally come from path parameters. The Skill and Agent Plugin
`POST /register` endpoints accept identity inputs in the body so they can create
or reuse the parent and create a version in one operation. Agent Plugin identity
is extracted from or checked against `plugin_json`.

```python
from typing import Any

from pydantic import BaseModel, ConfigDict, Field


class CreateSkillRequest(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None
    icons: list[RegistryIcon] | None = None


class UpdateSkillRequest(BaseModel):
    description: str | None = None
    icons: list[RegistryIcon] | None = None


class CreateSkillVersionRequest(BaseModel):
    # Metadata only; carries no content bytes. For a local-path upload this is
    # the JSON `metadata` part of a multipart request whose `content` part holds
    # the gzip-tar archive (see the REST section); the client leaves `source`
    # null to route into the upload flow, and the server sets the stored
    # version's `source` to the artifact path where it wrote the content.
    name: str | None = None  # structurally nullable only because this model is shared with POST /@{organization}/{name}/versions (where name comes from the parent path and any body name is ignored); POST /register validates it as required (the SDK/CLI fills it from SKILL.md during local inspection, so a raw REST caller that omits it is rejected)
    organization: str = ""  # for POST /register only; ignored for versioned paths
    source_type: str | None = None  # optional explicit type for an external source (git/oci/zip only; mlflow is flow-derived and rejected here); validated against `source`, with inference as the fallback when omitted (see Field inference)
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    digest: str | None = None  # client-computed content hash; stored and indexed, not server-verified
    status: str = "active"


class UpdateSkillVersionRequest(BaseModel):
    status: str | None = None


class CreateAgentPluginRequest(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None
    icons: list[RegistryIcon] | None = None


class UpdateAgentPluginRequest(BaseModel):
    description: str | None = None
    icons: list[RegistryIcon] | None = None


class PluginJSONPayload(BaseModel):
    model_config = ConfigDict(extra="allow", populate_by_name=True)

    schema_: str = Field(alias="$schema")
    name: str
    version: str | None = None
    description: str | None = None
    author: dict[str, str] | None = None
    homepage: str | None = None
    repository: str | None = None
    license: str | None = None
    keywords: list[str] | None = None
    extensions: Any = None


class CreateAgentPluginVersionRequest(BaseModel):
    version: str | None = None
    plugin_json: PluginJSONPayload | None = None
    skills: list[str] | None = None
    source_type: str | None = None  # optional explicit type for an external package source (git/oci/zip only; mlflow and assembled are flow-derived and rejected here); validated against `source`, with inference as the fallback when omitted (see Field inference)
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"


class RegisterAgentPluginRequest(CreateAgentPluginVersionRequest):
    name: str | None = None
    organization: str = ""


class MemberSkillDefinition(BaseModel):
    # One discovered member skill, read from its SKILL.md during local
    # inspection. Carries the discovery metadata the server stores on the
    # Skill so it is searchable like any standalone skill, plus the subpath
    # locating the skill within the package. The version and source_type are
    # server-set, and the member's source is derived from the package source.
    name: str
    subpath: str  # path of this skill within the package
    description: str | None = None
    keywords: list[str] | None = None
    digest: str | None = None  # client-computed content hash of this member's content


class ImportRegisterRequest(BaseModel):
    # Submitted after the client fetches and inspects the source locally. The
    # server creates the member skills and the packaged plugin version in one
    # transaction; the stored source_type is server-set and each member skill's
    # source is derived from the package (its source_type and source, plus the
    # member's subpath).
    plugin_json: PluginJSONPayload
    version: str | None = None  # required when plugin_json omits a version
    organization: str = ""
    member_skills: list[MemberSkillDefinition] = Field(default_factory=list)  # discovered members with metadata
    source_type: str | None = None  # optional explicit type for the external package pointer (git/oci/zip only); the import client fetched the source, so it always knows the type and submits it; validated against `source`, with inference as the fallback (see Field inference)
    source: str | None = None  # external package pointer; null for an MLflow-stored package
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"


class UpdateAgentPluginVersionRequest(BaseModel):
    status: str | None = None


class SkillAliasResponse(BaseModel):
    alias: str
    version: int


class SetSkillAliasRequest(BaseModel):
    alias: str
    version: int


class AgentPluginAliasResponse(BaseModel):
    alias: str
    version: str


class SetAgentPluginAliasRequest(BaseModel):
    alias: str
    version: str


class SetTagRequest(BaseModel):
    key: str
    value: str


class SkillVersionResponse(BaseModel):
    name: str
    version: int
    organization: str = ""
    source_type: str | None = None
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    digest: str | None = None
    status: str = "active"
    aliases: list[str] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class SkillResponse(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None
    icons: list[RegistryIcon] | None = None
    status: str | None = None
    latest_version: int | None = None
    aliases: list[SkillAliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class AgentPluginVersionResponse(BaseModel):
    name: str
    version: str
    organization: str = ""
    plugin_json: dict[str, Any]
    source_type: str | None = None
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"
    skills: list[str] = Field(default_factory=list)
    aliases: list[str] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class ImportRegisterResponse(BaseModel):
    # The /import transaction returns both the packaged plugin version and the
    # member skill versions it references, so the client need not re-fetch them;
    # every member skill version is newly created by the import (import does not
    # reuse existing versions).
    plugin_version: AgentPluginVersionResponse
    member_skill_versions: list[SkillVersionResponse] = Field(default_factory=list)


class AgentPluginResponse(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None
    icons: list[RegistryIcon] | None = None
    status: str | None = None
    latest_version: str | None = None
    aliases: list[AgentPluginAliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

Aliases are modeled as dictionaries on parent entities for convenience, while
REST responses expose explicit lists. Skill alias targets are integers and
agent plugin alias targets are strings.

**Composite primary keys.** Skills and agent plugins use a
`(workspace, organization, name)` composite primary key. The
`organization` field scopes ownership (e.g., a team or publisher
name) and defaults to `""` (empty string) when not specified.
This allows the same skill name to exist under different
organizations without collision (e.g., `acme/code-review` and
`beta-corp/code-review` are distinct skills). When `organization`
is empty, the skill is identified by name alone within its
workspace, consistent with the MCP Server Registry (RFC-0004).
The Prompt Registry is considering adding `organization` for
similar reasons; this design aligns with that direction.

**Canonical manifest validation.** `PluginJSONPayload` provides typed access to
known fields using `extra="allow"` so that unknown fields at any level are
accepted and preserved. The `$schema` identifier is not gated on a specific
version; manifests with newer schema identifiers are accepted as long as the
required fields are present. This matches RFC-0004's forward-compatibility
approach for MCP `server_json`. A non-object `extensions` field is preserved
and reported as a nonfatal warning but ignored semantically. Missing or
invalid required fields are fatal. `plugin_json["name"]` must match an explicit path or request name. When the
manifest does not include a version, the request must supply one. When both
are present, they must agree.

**Name and organization validation.** Skill and agent plugin identifiers
become segments of URIs, REST paths, and artifact storage paths, so the
server validates them on create and rejects any value that could introduce
path ambiguity or escape its controlled storage prefix:

- **Skill `name`:** 1 to 64 characters; lowercase ASCII letters (`a-z`),
  digits (`0-9`), and hyphens only; must not begin or end with a hyphen;
  and no consecutive hyphens. This matches the
  [Agent Skills naming rules](https://agentskills.io/specification), so no
  otherwise-valid skill name is rejected.
- **Agent plugin `name`:** the canonical Agent Plugins constraints: 1 to 64
  characters; lowercase ASCII letters, digits, hyphens, and periods;
  alphanumeric first and last characters; and no consecutive hyphens or
  periods. Agent plugin names that parse as valid SemVer are permitted,
  since the `@` marker distinguishes an organization from a `name/version`
  sequence without relying on SemVer recognition.
- **`organization`** (both entity types): empty (the default), or the same
  rule as an agent plugin name, which also allows periods so domain-style
  publisher names such as `acme.io` are valid: 1 to 64 characters; lowercase
  ASCII letters, digits, hyphens, and periods; alphanumeric first and last
  characters; and no consecutive hyphens or periods.

These rules exclude path-significant values by construction: `/`, `\`, `@`,
`#`, `?`, whitespace, control characters, percent-encoded separators, and
dot-segments such as `.` and `..` cannot appear in any identifier, and the
leading `@` that marks an organization segment is never part of a value.
Because an organization is always marked by a leading `@` in URIs and REST
paths, the parser never has to guess whether a segment is an organization or
a name; numeric skill names (e.g., `123`) are therefore permitted.

As defense in depth, artifact operations build storage paths only from
validated segments (organization, name, and version) and verify that the
resolved path stays within the controlled prefix (`skills/...` or
`agent-plugins/...`), rejecting any operation that would resolve outside it.

## Python SDK and CLI

The `mlflow.genai` module exposes the high-level workflow functions
(register, search, import, introspect, pull); the full create/get/
update/delete surface, including version, tag, and alias operations,
is available as `MlflowClient` methods, which the `mlflow.genai`
helpers call internally. Skill-specific entity and request types are
also re-exported from `mlflow.genai`. The `mlflow skills` and
`mlflow agent-plugins` CLI command groups provide the same operations
from the command line, mapping to whichever layer defines each
function (the SDK function column below). Identity-establishing commands
(`register`, and `agent-plugins create`) accept `--name` with an optional
`--organization`. Commands that operate on
an already-registered entity accept its URI as an optional positional argument,
with the equivalent `--skill-uri` (skills) or `--plugin-uri` (agent plugins)
flag also accepted; for example, `mlflow skills get skills:/code-review` and
`mlflow skills get --skill-uri skills:/code-review` are equivalent:

| CLI subcommand | SDK function | Description |
|---|---|---|
| `mlflow skills register git` | `register_skill(source=GitSource(...))` | Register from a Git repository |
| `mlflow skills register oci` | `register_skill(source=OCISource(...))` | Register from an OCI image |
| `mlflow skills register zip` | `register_skill(source=ZipSource(...))` | Register from a ZIP archive |
| `mlflow skills register` | `register_skill(source="./local-path")` | Register from a local directory (uploaded to MLflow artifact storage) |
| `mlflow skills get` | `get_skill()` | Get skill metadata |
| `mlflow skills update` | `update_skill()` | Update skill presentation metadata (description, icons) |
| `mlflow skills search` | `search_skills()` | Search skills |
| `mlflow skills get-version` | `get_skill_version()` | Get a specific version |
| `mlflow skills search-versions` | `search_skill_versions()` | Search versions of a skill |
| `mlflow skills update-version` | `update_skill_version()` | Update version status |
| `mlflow skills delete-version` | `delete_skill_version()` | Soft-delete a version (withdrawal kill switch) |
| `mlflow skills set-alias` | `set_skill_alias()` | Set a version alias |
| `mlflow skills delete-alias` | `delete_skill_alias()` | Delete a version alias |
| `mlflow skills set-tag` | `set_skill_tag()` | Set a skill-level tag |
| `mlflow skills delete-tag` | `delete_skill_tag()` | Delete a skill-level tag |
| `mlflow skills set-version-tag` | `set_skill_version_tag()` | Set a version tag |
| `mlflow skills delete-version-tag` | `delete_skill_version_tag()` | Delete a version tag |
| `mlflow skills pull` | `pull()` | Pull content to local filesystem |
| `mlflow agent-plugins create` | `create_agent_plugin()` | Create an agent plugin |
| `mlflow agent-plugins create-version` | `create_agent_plugin_version()` | Create a version on an existing parent |
| `mlflow agent-plugins register` | `register_agent_plugin()` | Create or reuse the parent and register a canonical version |
| `mlflow agent-plugins get` | `get_agent_plugin()` | Get agent plugin metadata |
| `mlflow agent-plugins get-version` | `get_agent_plugin_version()` | Get a specific version |
| `mlflow agent-plugins update` | `update_agent_plugin()` | Update agent plugin presentation metadata (description, icons) |
| `mlflow agent-plugins search` | `search_agent_plugins()` | Search agent plugins |
| `mlflow agent-plugins search-versions` | `search_agent_plugin_versions()` | Search agent plugin versions |
| `mlflow agent-plugins set-alias` | `set_agent_plugin_alias()` | Set an agent plugin alias |
| `mlflow agent-plugins delete-alias` | `delete_agent_plugin_alias()` | Delete an agent plugin alias |
| `mlflow agent-plugins set-tag` | `set_agent_plugin_tag()` | Set an agent plugin-level tag |
| `mlflow agent-plugins delete-tag` | `delete_agent_plugin_tag()` | Delete an agent plugin-level tag |
| `mlflow agent-plugins set-version-tag` | `set_agent_plugin_version_tag()` | Set an agent plugin version tag |
| `mlflow agent-plugins delete-version-tag` | `delete_agent_plugin_version_tag()` | Delete an agent plugin version tag |
| `mlflow agent-plugins update-version` | `update_agent_plugin_version()` | Update agent plugin version status |
| `mlflow agent-plugins delete-version` | `delete_agent_plugin_version()` | Soft-delete a version |
| `mlflow agent-plugins introspect` | `introspect_agent_plugin()` | Preview a local or remote plugin without registry writes |
| `mlflow agent-plugins import` | `import_agent_plugin()` | Import a plugin as a packaged agent plugin |
| `mlflow agent-plugins pull` | `pull()` | Pull content to local filesystem |

The `register git`/`oci`/`zip` subcommands name the source type
explicitly, and the CLI submits it as the request's `source_type` (just
as the SDK's typed source classes do), so registration through these
surfaces never depends on the server inferring the type from the URL's
shape.

`create-version` and `register` accept `--plugin-json PATH` for a full standard
manifest. When omitted for an assembled plugin, `--version` is required and
the command synthesizes a minimal manifest from the target identity (the
positional URI or `--plugin-uri` for `create-version`, which targets an
existing parent; `--name` for `register`) and the supplied version. Search
commands accept `--filter-string`, which
covers both structured filters and free-text matching against the derived
`search_text` field (e.g., `--filter-string "search_text LIKE '%review%'"`).

The CLI covers routine create, read, update, tag, alias, and version-status
operations, plus version soft delete (`delete-version`). Soft-deleting a skill
version is the withdrawal kill switch, since it also withdraws the agent plugin
versions that contain that member; soft-deleting an agent plugin version affects
only that version. The CLI does not expose top-level hard delete of a parent
entity: `delete_skill` and `delete_agent_plugin` are administrative operations
available through the SDK (`MlflowClient`) and REST only, and are intentionally
omitted from the CLI to keep destructive cascade deletes off the command line.
`delete_skill` hard-deletes any skill, including one created by importing a
packaged plugin, subject to the usual referential-integrity checks; such a
skill is an ordinary skill and is not tied to the lifetime of the plugin it
was imported from.

**Relationship to existing `mlflow skills` subcommands.** MLflow already
has `mlflow skills list` and `mlflow skills view` subcommands
(`mlflow/cli/skills.py`) that inspect the bundled Assistant skills
shipping with the MLflow installation (under `mlflow/assistant/skills/`).
The registry's subcommands use different names (e.g., `register`,
`search`, `pull`), so there is no direct conflict.

## Plugin import

Source fetching and inspection are implemented in the SDK and CLI layer: the
client fetches and inspects the source locally, and the registry server never
fetches user-supplied plugin URLs. Registration itself is a single server-side
transaction. After inspecting the source, the client submits the prepared
canonical manifest and the discovered member-skill definitions to a dedicated
import-registration endpoint, which atomically creates the member skill
versions and the packaged agent plugin version. Because the client does the
fetching and the server receives only the already-prepared payload, this
endpoint still does not require the server to reach user-supplied URLs.

### Read-only preview

`introspect_agent_plugin()` and `mlflow agent-plugins introspect` run the same plugin
discovery used by import but do not create or modify registry records.
They accept either a local path or a remote Git, OCI, ZIP, or MLflow artifact
source and return the detected format, canonical manifest preview, discovered
skill names and paths, recognized unregistered content such as `mcp.json`, and
warnings. The preview reports the `source_type` the version would receive,
determined from the source value by the same rules the server applies at
registration; because introspect writes no registry record, this is computed
client-side.

### Supported input formats

The importer:

1. Fetches the Git, OCI, ZIP, or MLflow artifact source using the same
   source-type-aware logic as `pull`.
2. Applies `subpath`, when provided, to select the plugin root.
3. Checks for a root `plugin.json` with an Agent Plugins `$schema` identifier.
   When found, it validates the required fields and discovers immediate children
   at `skills/*/SKILL.md`. A manifest missing required fields fails import.
   Unknown fields and newer schema versions are accepted and preserved.
4. Otherwise checks for `.claude-plugin/plugin.json`, translates available
   metadata into canonical `plugin_json`, and applies Claude Code discovery.
5. Otherwise applies the existing generic skill-directory discovery and
   synthesizes a minimal canonical manifest. `plugin_name` is used only when
   the adapter cannot infer a name.
6. Reports root `mcp.json` as recognized but unregistered standard content and
   reports other non-skill content without assigning membership semantics.

When multiple markers exist, the standard root manifest takes precedence. The
canonical name follows the Agent Plugins naming constraints. A supplied
manifest version is used as the registry version (normalized to full
SemVer); when the manifest does not include a version, the user must
supply one via `--version` or the `version` parameter.

### Registration behavior

Before registering anything, the importer derives each discovered skill's
name from its `SKILL.md` and rejects the package if two skills resolve to the
same name, since member names must be unique within the resulting agent plugin
version (see Member-name uniqueness). Directory paths under `skills/*/` are not
used as the uniqueness key, because a skill's name is declared in `SKILL.md`
rather than taken from its directory. Each derived name, whether read from
`SKILL.md` or synthesized by a Claude Code or generic adapter, is validated
against the skill `name` rule (see Name and organization validation); import
fails on an invalid derived name rather than silently rewriting it, so a name
that would be illegal for a standalone skill cannot enter the registry through
import.

The importer also enforces skill-name uniqueness within `(workspace,
organization)`. On a first import, each discovered name must be free unless it
belongs to the plugin's own history: if the name is already taken by a skill
that is not part of this plugin's membership history, the whole import is
rejected rather than colliding with another skill. The workaround for a genuine
collision is to import into a different organization. Standalone skill creation
(`register_skill`/`create_skill`) is symmetric: it fails if the name is already
taken by a member of a packaged plugin. This keeps a skill name bound to a
single skill, whether that skill was registered standalone or created by an
import.

The import-registration endpoint performs the following through the store's
`import_agent_plugin` unit of work, which commits it all in a single database
transaction. For each discovered skill, the server creates a `SkillVersion`
whose version is server-assigned and whose `source` is **derived from the
package**: the package's `source_type` and `source` (preserving `ref` for Git
sources) with a `subpath` locating the skill within the package. For a package
stored in MLflow artifact storage the member's `source_type` is `mlflow`, its
`source` is set to the plugin version's artifact directory, and its `subpath`
locates the skill within that stored package tree; because this pointer is
persisted on the member skill version, the member resolves independently of its
membership rows and survives a non-cascade delete of the plugin. The parent
`Skill` stores the `description` from the submitted `MemberSkillDefinition` in
its `description` column, from which `search_text` is derived, and folds the
`keywords` (which have no column of their own) into `search_text` as well, so the
skill returns a populated description and is discoverable by keyword and
description search like any other skill. Each member skill is referenced by name in the member list (e.g.,
`skills:/code-review/1`). In the same transaction the server creates one
packaged `AgentPluginVersion` whose `source_type` reflects where the
package lives: the original typed source (preserving `ref` for Git
sources) for a `git`, `oci`, or `zip` package, or `mlflow` with a null
`source` for a package stored in MLflow artifact storage. It carries the
immutable canonical `plugin_json` and member references for all
discovered skills. This preserves a pullable link to the complete original
package (including `mcp.json` and other non-skill content) while also making
each member independently addressable and pullable through its derived source.
A valid package with no skills creates an agent plugin version with an empty
member list.

Because the member skills and the plugin version are committed together, a
failure at any step rolls the whole registration back: no member skill
versions are left behind. Import
requires a source that already exists (an external `git`, `oci`, or `zip`
pointer, or a package already resident in MLflow artifact storage) and preserves
it as the plugin version's pullable pointer, so registration performs no content
upload; the transaction is the only step that can fail.

#### Re-import behavior

When the target agent plugin already has at least one version, import first
rejects an incoming canonical version that already exists. It then
matches discovered skills to existing members by name, using the
member skill names from the most recently created non-`deleted` agent plugin
version's member list (a soft-deleted latest version is skipped so matching
reflects the plugin's current effective members). Import never reuses an
existing skill version: every discovered member produces a new skill version so
that each version keeps exactly one immutable, package-derived source (see
Content digest). The name match only decides which skill the new version is
added to:

1. **Matching name:** The discovered skill's name matches a member name in the
   previous member list. Import adds a new version to that existing skill, with a
   source derived from the newly imported package and the discovered content
   `digest` recorded on it. When the content is unchanged the new version shares
   the previous version's digest, which is how a client later recognizes it as
   unchanged (clean diffs between agent plugin versions, and traces before and
   after the re-import linking to the same content); the digest drives this
   recognition on the read side rather than causing reuse at import time.
2. **New name:** The name does not match any current member. If a skill with
   that name has previously been a member of this same plugin (for example, a
   member removed in an earlier version and now reintroduced), which the server
   derives from this plugin's member rows, import adds a new version to that
   existing skill. Otherwise, if the name is free within the organization,
   import creates a new skill with its own next server-assigned integer version.
   Either way the new version's source is derived from the package and its digest
   is the discovered content digest. If the name is already taken by a skill that
   is not part of this plugin's history, the import is rejected.
3. **Removed name:** A previous member's name is not found in the
   new source. The member is omitted from the new agent plugin version.
   The skill and its existing versions remain in the registry.

A skill that is renamed between versions is treated as a removed skill
and a new one.

Whenever re-import matches a discovered skill to an existing skill (cases 1
and 2 above), adding the new version requires EDIT on that skill, evaluated
separately from plugin permissions, exactly as for any other version creation
(see Permissions). This matters once an agent plugin's kind can vary across
versions: an assembled version may reference a standalone skill the caller does
not own, so a later packaged import that discovers the same name cannot append
to it without EDIT on that skill. An import lacking EDIT on a matched skill
fails as a whole, so a reference-level relationship never becomes a write.

After processing all discovered skills, the endpoint creates a new
`AgentPluginVersion` with updated member references in the same transaction as
the member skill versions it created. Previous agent
plugin versions are immutable and unchanged.

Agent plugin and member skill version sequences are independent: a plugin
version such as `"1.2.0"` may reference skill version `7`.

### Warnings and result

Root `mcp.json` and other non-skill content remain in the package but are not
registered. Recognized standard content is returned separately from warnings;
adapter-specific or unknown skipped categories produce a `PluginImportWarning`
containing category, path, and explanation. The CLI prints the detected format,
canonical identity, recognized unregistered content, and warnings. The SDK
returns them with the created versions in `PluginImportResult`.

Import normalizes supported inputs into the canonical Agent Plugins registry
representation while preserving the complete package source. It does not
install or export an MLflow agent plugin.

## Pull semantics details

**Source availability.** The registry stores source pointers but does
not cache or proxy content. If a source is unreachable or the content
has been deleted, pull fails with an error that surfaces the
underlying failure from the source system (e.g., Git clone failure,
OCI pull 404, HTTP download error, MLflow artifact download error).
Source availability is the publisher's responsibility. For assembled
agent plugin pulls, if one member's source is unavailable, the entire pull
fails rather than producing a partial result.

**Source authentication.** The registry server stores source pointers
but does not validate source accessibility at registration time and is
not involved in content transfer at pull time. Authentication to
external sources is handled entirely by the client environment:

| Source type | Authentication mechanism |
|---|---|
| `git` | Standard Git credential resolution: SSH keys (`~/.ssh/`), Git credential helpers (`git-credential-manager`, `git-credential-store`), `.netrc`, and `GIT_SSH_COMMAND`. Private repos work if the caller's Git is configured to access them. |
| `oci` | OCI registry credential resolution: Docker config (`~/.docker/config.json`), registry-specific credential helpers, and container runtime auth. Private registries work if the caller has a valid login session. |
| `zip` | No authentication support. ZIP sources must be publicly accessible URLs. For private content, use `git` or `oci` source types instead. |
| `mlflow` | MLflow artifact storage authentication, using the same credentials as other MLflow API calls. |

The registry does not store, proxy, or manage source credentials.
Pull failures due to authentication errors are surfaced to the caller
with the underlying error from the source system.

## SDK and CLI code examples

### Register skills from an OCI artifact with subpath

```python
import mlflow

mlflow.genai.register_skill(
    name="code-review",
    source=OCISource(
        image="oci://ghcr.io/acme/agent-plugin:v1.0.0",
        subpath="skills/code-review",
    ),
)

mlflow.genai.register_skill(
    name="test-coverage",
    source=OCISource(
        image="oci://ghcr.io/acme/agent-plugin:v1.0.0",
        subpath="skills/test-coverage",
    ),
)

# Assembled agent plugin: each member has its own source. With no explicit
# plugin_json, version is required and a minimal manifest is synthesized.
plugin_version = mlflow.genai.register_agent_plugin(
    name="pr-workflow",
    version="0.1.0",
    skills=[
        "skills:/code-review/1",
        "skills:/test-coverage/1",
    ],
)
```

Packaged agent plugins are typically created through `import_agent_plugin()`, which
handles package inspection and member skill creation internally. See the
[Plugin import](#plugin-import) section for details.

### Discover and consume skills

```python
from mlflow import MlflowClient

client = MlflowClient()

# Free-text discovery across skill names and descriptions (high-level API)
skills = mlflow.genai.search_skills(filter_string="search_text LIKE '%code review%'")
skill = skills[0]

# Search for active versions of that skill (low-level CRUD on MlflowClient)
versions = client.search_skill_versions(
    name=skill.name,
    organization=skill.organization,
    filter_string="status = 'active'",
)

# Search for active agent plugins (high-level API)
plugins = mlflow.genai.search_agent_plugins(
    filter_string="search_text LIKE '%pull request review%' AND status = 'active'",
)

# Get a specific version
version = client.get_skill_version(
    name=skill.name,
    organization=skill.organization,
    version=1,
)
# isinstance(version.source, GitSource)
# version.source.url == "https://github.com/acme/agent-skills.git"
# version.source.ref == "v1.0.0"
# version.source.subpath == "code-review"

# Resolve by alias
version = client.get_skill_version_by_alias(
    name=skill.name,
    organization=skill.organization,
    alias="production",
)

# Get an agent plugin version and its pinned members
plugin = mlflow.genai.search_agent_plugins(
    filter_string="name = 'pr-workflow'",
)[0]
plugin_version = client.get_agent_plugin_version(
    name=plugin.name,
    organization=plugin.organization,
    version="0.1.0",
)
# plugin_version.skills == ["skills:/code-review/1", ...]

# Resolve an agent plugin alias
plugin_version = client.get_agent_plugin_version_by_alias(
    name=plugin.name,
    organization=plugin.organization,
    alias="production",
)
```
