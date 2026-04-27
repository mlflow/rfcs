---
start_date: 2026-04-22
mlflow_issue: https://github.com/mlflow/mlflow/issues/22833
rfc_pr: https://github.com/mlflow/rfcs/pull/10
---

# RFC: Skill Registry

| Author(s)              | Bill Murdock (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-04-27 |
| **AI Assistant(s)**    | Claude Code (Opus 4.6) |

# Summary

Add a Skill Registry to MLflow: a governed, metadata-first registry for
AI agent capabilities. The registry stores metadata and typed source
pointers (to Git repos, OCI registries, ZIP archives, etc.) rather
than artifacts directly. It provides enterprise governance on top of
existing distribution mechanisms: lifecycle management, security scan
tracking, usage analytics via traces, and federated discovery across
sources.

The registry tracks four capability kinds under the `mlflow skills`
namespace:

- **Skills** (SKILL.md) — reusable agent instructions
- **Agents** (agent .md) — sub-agent definitions
- **MCP servers** (JSON config) — tool server integrations
- **Hooks** (harness-specific) — event-triggered actions

Skill groups bundle related capabilities of any kind into versioned,
governed units that map to the "plugin" concept in agent harnesses.

`mlflow skills pull` provides a harness-agnostic way to fetch
registered content from its source. Harness-specific installation
(manifest generation, directory placement) is covered in a companion
RFC (RFC-0006).

# Basic example

## Register a skill and publish it

```python
import mlflow

# Create the logical skill asset
skill = mlflow.skills.create_skill(
    name="code-review",
    description="Reviews pull requests for correctness, style, and security",
)

# Register a version pointing to a Git source
version = mlflow.skills.create_skill_version(
    name="code-review",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/agent-skills/tree/v1.0.0/code-review",
    content_digest="sha256:a3f2b8c...",
)
# version.publish_state == "draft"

# Publish the version so downstream consumers can discover it
mlflow.skills.update_skill_version(
    name="code-review",
    version="1.0.0",
    publish_state="published",
)

# Set an alias for stable resolution
mlflow.skills.set_skill_alias(
    name="code-review",
    alias="production",
    version="1.0.0",
)

# Record a security scan result as a tag
mlflow.skills.set_skill_version_tag(
    name="code-review",
    version="1.0.0",
    key="scan.prompt-injection.status",
    value="pass",
)
mlflow.skills.set_skill_version_tag(
    name="code-review",
    version="1.0.0",
    key="scan.prompt-injection.date",
    value="2026-04-22",
)
```

## Create a skill group with a versioned membership snapshot

```python
from mlflow.entities import SkillGroupVersionMembership

# Create a group for related skills
group = mlflow.skills.create_skill_group(
    name="pr-workflow",
    description="End-to-end pull request review workflow",
)

# Create a group version that pins specific skill versions
group_version = mlflow.skills.create_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
    members=[
        SkillGroupVersionMembership(
            skill_name="code-review", skill_version="1.0.0",
        ),
        SkillGroupVersionMembership(
            skill_name="test-coverage", skill_version="2.1.0",
        ),
        SkillGroupVersionMembership(
            skill_name="security-scan", skill_version="1.0.0",
        ),
    ],
)

# Publish the group version
mlflow.skills.update_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
    publish_state="published",
)

# Set an alias for stable resolution
mlflow.skills.set_skill_group_alias(
    name="pr-workflow",
    alias="production",
    version="1.0.0",
)
```

## Register other capability kinds

```python
# Register a sub-agent
mlflow.skills.create_skill(
    name="security-auditor",
    kind="agent",
    description="Security specialist for auth and payment code",
)
mlflow.skills.create_skill_version(
    name="security-auditor",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/agent-skills/tree/v1.0.0/security-auditor",
)

# Register an MCP server
mlflow.skills.create_skill(
    name="github-mcp",
    kind="mcp-server",
    description="GitHub integration via MCP",
)
mlflow.skills.create_skill_version(
    name="github-mcp",
    version="2.0.0",
    source_type="oci",
    source="ghcr.io/acme/github-mcp:2.0.0",
    content_digest="sha256:b4e9f1d...",
)

# Register a hook
mlflow.skills.create_skill(
    name="pre-commit-scan",
    kind="hook",
    description="Runs security scan before tool commits",
)
mlflow.skills.create_skill_version(
    name="pre-commit-scan",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/agent-skills/tree/v1.0.0/pre-commit-scan",
)
```

## Create a skill group with mixed capability kinds

```python
from mlflow.entities import SkillGroupVersionMembership

group = mlflow.skills.create_skill_group(
    name="pr-workflow",
    description="End-to-end pull request review workflow",
)

# A group version can bundle skills, agents, MCP servers, and hooks
group_version = mlflow.skills.create_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
    members=[
        SkillGroupVersionMembership(
            skill_name="code-review", skill_version="1.0.0",
        ),
        SkillGroupVersionMembership(
            skill_name="security-auditor", skill_version="1.0.0",
        ),
        SkillGroupVersionMembership(
            skill_name="github-mcp", skill_version="2.0.0",
        ),
    ],
)
```

## Pull skills to a local directory

```python
# Pull a single skill version
mlflow.skills.pull_skill(
    name="code-review",
    alias="production",
    destination="./skills/code-review",
)

# Pull an entire skill group (all members)
mlflow.skills.pull_skill_group(
    name="pr-workflow",
    alias="production",
    destination="./plugins/pr-workflow",
)
```

```bash
# CLI equivalents
mlflow skills pull --name code-review --alias production \
    --destination ./skills/code-review

mlflow skills pull-group --name pr-workflow --alias production \
    --destination ./plugins/pr-workflow
```

## Discover and consume skills

```python
# Search for published skill versions
versions = mlflow.skills.search_skill_versions(
    name="code-review",
    filter_string="publish_state = 'published'",
)

# Search for active skill groups
groups = mlflow.skills.search_skill_groups(
    filter_string="status = 'active'",
)

# Get a specific version
version = mlflow.skills.get_skill_version(
    name="code-review",
    version="1.0.0",
)
# version.source_type == "git"
# version.source == "https://github.com/acme/agent-skills/tree/v1.0.0/code-review"

# Resolve by alias
version = mlflow.skills.get_skill_version_by_alias(
    name="code-review",
    alias="production",
)

# Get a group version and its pinned skill versions
group_version = mlflow.skills.get_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
)
# group_version.members == [SkillGroupVersionMembership(...), ...]

# Resolve a group alias
group_version = mlflow.skills.get_skill_group_version_by_alias(
    name="pr-workflow",
    alias="production",
)
```

## CLI usage

```bash
# Register a skill pointing to a Git source
mlflow skills create --name code-review \
    --description "Reviews pull requests"
mlflow skills create-version --name code-review --version 1.0.0 \
    --source-type git \
    --source https://github.com/acme/agent-skills/tree/v1.0.0/code-review \
    --content-digest sha256:a3f2b8c...

# Publish and alias
mlflow skills update-version --name code-review --version 1.0.0 \
    --publish-state published
mlflow skills set-alias --name code-review --alias production \
    --version 1.0.0

# Create a group and a versioned membership snapshot
mlflow skill-groups create --name pr-workflow \
    --description "End-to-end PR review workflow"
mlflow skill-groups create-version --name pr-workflow --version 1.0.0 \
    --member code-review:1.0.0 \
    --member test-coverage:2.1.0 \
    --member security-scan:1.0.0
mlflow skill-groups update-version --name pr-workflow --version 1.0.0 \
    --publish-state published
mlflow skill-groups set-alias --name pr-workflow --alias production \
    --version 1.0.0

# Search published skill versions
mlflow skills search-versions --name code-review \
    --filter "publish_state = 'published'"

# Search active groups
mlflow skill-groups search --filter "status = 'active'"
```

## Motivation

### The problem

AI agent capabilities — skills, sub-agents, MCP server configurations,
and hooks — are becoming a critical asset class in enterprise AI
platforms. As organizations adopt agentic AI, they accumulate these
capabilities across teams, repositories, and agent harnesses.

A cross-harness portable format is emerging around SKILL.md files (for
skills and agents), MCP server configs (for tool integrations), and
hooks (for event-triggered actions). Agent harnesses including Claude
Code, Codex CLI, Cursor, GitHub Copilot, OpenClaw, Kilo Code, and
Antigravity support overlapping subsets of these formats, with SKILL.md
and MCP being the most broadly adopted.

Today, these capabilities are managed as ad-hoc files in Git
repositories. This works well for individual developers and small
teams. GitHub provides versioning, collaboration, and access control.

However, enterprises face governance challenges that Git alone does not
address:

1. **No publish-state lifecycle.** Git has no concept of "this version
   is approved for production use" vs. "this is a draft." Teams resort
   to branch naming conventions or external tracking to manage
   promotion.

2. **No security scan tracking.** Skills may contain executable code or
   be vulnerable to prompt injection. Hooks execute arbitrary commands.
   There is no standard place to record whether a capability version
   has been scanned and what the results were.

3. **Fragmented discovery.** Capabilities may live in multiple Git
   repos, OCI registries, or other distribution systems. There is no
   single discovery layer across all of these.

4. **No cross-kind grouping.** Agent harnesses like Claude Code and
   Codex CLI support plugins that bundle skills, agents, MCP servers,
   and hooks together. But there is no agent-neutral way to represent
   these bundles for governance and discovery.

5. **No usage analytics linkage.** MLflow traces can capture skill
   metadata, but without a governed registry, there is no way to link
   trace data back to a governed record to understand adoption across
   an organization.

6. **No pull mechanism.** Once a user discovers a capability in the
   registry, there is no standard way to fetch its content from the
   source system. Users must manually copy source pointers and run
   harness-specific install steps.

### Use cases

1. **Governed registration**: Platform administrators register
   capability metadata with typed source pointers to where the content
   lives (Git, OCI, ZIP). The registry governs; the source system
   stores. All four capability kinds (skill, agent, mcp-server, hook)
   use the same registration model.

2. **Lifecycle management**: Capability versions move through publish
   states (draft, published, deprecated, retired) to control downstream
   surfacing. This is the governance layer that Git lacks.

3. **Security scan tracking**: Scan results (prompt injection, code
   vulnerabilities, etc.) are recorded as version-level tags. The
   registry does not perform scans; it provides the metadata layer for
   recording and querying results.

4. **Cross-kind grouping**: Related capabilities of any kind are
   organized into skill groups for discovery and governance. A skill
   group maps to the "plugin" concept in agent harnesses — for example,
   a "pr-workflow" group might bundle a code-review skill, a
   security-auditor agent, and a GitHub MCP server.

5. **Federated discovery**: Users discover published capabilities and
   groups across all source types from a single search interface,
   filtered by kind, without requiring content to be centralized.

6. **Pull**: `mlflow skills pull` fetches capability content from its
   registered source to a local directory. This is source-type-aware
   (git clone, OCI pull, ZIP extract) and harness-agnostic.

7. **Usage analytics**: Agent traces record which capability versions
   were used. Combined with registry metadata, this enables
   organizations to understand adoption and make data-driven promotion
   decisions.

### Out of scope

- **Artifact storage.** The registry stores metadata and source
  pointers. Content remains in Git, OCI, or other distribution systems.
  `pull` fetches from the source; the registry itself does not store
  artifacts.
- **Authoring or development tools.** The registry manages published
  capabilities, not the process of writing them.
- **Format specification.** The registry is format-agnostic. It does
  not define or enforce what a skill, agent, MCP config, or hook looks
  like.
- **Security scanning execution.** The registry records scan results;
  it does not perform scans.
- **Harness-specific installation.** How a specific agent harness
  (Claude Code, Codex CLI, Cursor, etc.) installs capabilities from
  the registry — including manifest generation and directory placement
  — is covered in a companion RFC (RFC-0006). This RFC provides the
  registry, governance, and `pull`; RFC-0006 provides `install`.
- **Approval workflows or review gates.** Publish state transitions
  are sufficient for initial governance.
- **Detailed UI/UX design.** This RFC describes the UI surface and
  placement but does not specify interaction patterns.

## Detailed design

### Entities and data model

```mermaid
erDiagram
Skill ||--o{ SkillVersion : "has versions"
Skill ||--o{ SkillTag : "has tags"
Skill ||--o{ SkillAlias : "has aliases"
SkillVersion ||--o{ SkillVersionTag : "has tags"
SkillGroup ||--o{ SkillGroupVersion : "has versions"
SkillGroup ||--o{ SkillGroupTag : "has tags"
SkillGroup ||--o{ SkillGroupAlias : "has aliases"
SkillGroupVersion ||--o{ SkillGroupVersionMembership : "contains skills"
SkillGroupVersion ||--o{ SkillGroupVersionTag : "has tags"
SkillGroupVersionMembership }o--|| SkillVersion : "references"
```

#### Skill

The logical governed asset, scoped to a workspace.

```python
from dataclasses import dataclass, field
from enum import StrEnum


class SkillKind(StrEnum):
    SKILL = "skill"
    AGENT = "agent"
    MCP_SERVER = "mcp-server"
    HOOK = "hook"


class SkillStatus(StrEnum):
    ACTIVE = "active"
    DEPRECATED = "deprecated"
    RETIRED = "retired"


@dataclass
class Skill:
    name: str
    kind: SkillKind = SkillKind.SKILL
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus = SkillStatus.ACTIVE
    tags: dict[str, str] = field(default_factory=dict)
    aliases: list[SkillAlias] = field(default_factory=list)
    last_registered_version: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Stable logical asset name, unique within a workspace |
| `kind` | `SkillKind` | Capability type: `skill`, `agent`, `mcp-server`, `hook` |
| `status` | `SkillStatus` | Skill-level lifecycle: `active`, `deprecated`, `retired` |
| `aliases` | `list[SkillAlias]` | Stable version pointers (e.g., `production` → `1.2.0`) |
| `last_registered_version` | `str` | Most recently registered version string |
| `workspace` | `str` | Visibility boundary |

**Kind extensibility.** The `kind` enum covers the four capability
types with broad cross-harness support. New kinds can be added without
schema changes since the column stores a string value. `kind` is
immutable after creation.

#### SkillVersion

A versioned record containing a typed source pointer, publish state,
and tags.

```python
class SkillPublishState(StrEnum):
    DRAFT = "draft"
    PUBLISHED = "published"
    DEPRECATED = "deprecated"
    RETIRED = "retired"


class SkillSourceType(StrEnum):
    GIT = "git"
    OCI = "oci"
    ZIP = "zip"


@dataclass
class SkillVersion:
    name: str
    version: str
    source_type: SkillSourceType | None = None
    source: str | None = None
    publish_state: SkillPublishState = SkillPublishState.DRAFT
    content_digest: str | None = None
    tags: dict[str, str] = field(default_factory=dict)
    run_id: str | None = None
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `version` | `str` | Publisher-supplied version string. Semver recommended but not enforced |
| `source_type` | `SkillSourceType` | Optional distribution mechanism: `git`, `oci`, `zip` |
| `source` | `str` | Optional pointer to the content in the source system. Required for standalone pull; omit when content is only available via a group-level source |
| `content_digest` | `str` | Optional digest for integrity verification (e.g., `sha256:abc123...`). Aligns with OCI digest terminology |
| `publish_state` | `SkillPublishState` | Per-version surfacing lifecycle |
| `run_id` | `str` | Optional MLflow run association for trace linkage |

**Source type extensibility.** The `source_type` enum is intentionally
small for the initial implementation. New source types (e.g., `s3`,
`azure-blob`) can be added without schema changes since the column
stores a string value.

**Version uniqueness.** The combination of `(name, version)` is unique
within a workspace. A skill version represents a single logical
version of a capability; `source_type` and `source` describe where to
find it but are not part of its identity. If the same content is
available from multiple distribution mechanisms (e.g., Git and OCI),
register separate versions or use a group-level source.

**Content integrity.** The optional `content_digest` field stores a
digest of the skill content at registration time (e.g.,
`sha256:abc123...`). Consumers can use this to verify that the content
at `source` has not changed since registration. For OCI sources,
this is the native image digest. For Git sources, this is a digest of
the skill file contents at the pinned commit. For ZIP sources, this is
a digest of the archive. The registry stores the digest but does not
verify it on read; verification is the consumer's responsibility.

**Immutability contract.** `source_type`, `source`, `content_digest`,
and `version` are immutable after creation. To point to different content,
register a new version. Mutable fields (`publish_state`, `tags`) can be
updated independently.

#### SkillGroup

The logical group asset, scoped to a workspace. A skill group bundles
capabilities of any kind (skills, agents, MCP servers, hooks) into a
governed unit that maps to the "plugin" concept in agent harnesses.
Follows the same pattern as Skill: a top-level entity with versions,
tags, and aliases.

```python
class SkillGroupStatus(StrEnum):
    ACTIVE = "active"
    DEPRECATED = "deprecated"
    RETIRED = "retired"


@dataclass
class SkillGroup:
    name: str
    description: str | None = None
    workspace: str | None = None
    status: SkillGroupStatus = SkillGroupStatus.ACTIVE
    tags: dict[str, str] = field(default_factory=dict)
    aliases: list["SkillGroupAlias"] = field(default_factory=list)
    last_registered_version: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

#### SkillGroupVersion

A versioned snapshot of a skill group's membership. Each version
captures a specific set of skill versions that work together.

```python
@dataclass
class SkillGroupVersion:
    name: str
    version: str
    source_type: SkillSourceType | None = None
    source: str | None = None
    content_digest: str | None = None
    publish_state: SkillPublishState = SkillPublishState.DRAFT
    tags: dict[str, str] = field(default_factory=dict)
    members: list["SkillGroupVersionMembership"] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

**Version uniqueness.** The combination of `(name, version)` is unique
within a workspace.

**Group-level source.** A group version can optionally have its own
`source_type`, `source`, and `content_digest`, pointing to a single
artifact (e.g., an OCI image or Git repo) that contains the complete
plugin. When present, `pull` fetches the group artifact as a unit
rather than pulling members individually. This supports distribution
patterns where a plugin is packaged as a single image or repo.

**Source resolution for pull.** When pulling a group, if the group
version has a source, that source is used. Otherwise, each member is
pulled individually from its own source. Members without a source are
skipped with a warning. When pulling a standalone skill, the skill
version's source is required.

**Immutability contract.** The membership list and source fields of a
group version are immutable after creation. To change the set of
skills or source pointer, register a new group version. Mutable fields
(`publish_state`, `tags`) can be updated independently.

#### SkillGroupVersionMembership

Each membership entry pins a specific skill version (including source
type). The parent group identity is provided by the enclosing
`SkillGroupVersion`; the storage layer adds those columns as FKs.

```python
@dataclass(frozen=True)
class SkillGroupVersionMembership:
    skill_name: str
    skill_version: str
```

A skill can appear in multiple groups and multiple group versions.
Membership is at the skill version level, so a group version is a
reproducible snapshot of "these specific skill versions work together."

#### SkillGroupAlias

```python
@dataclass(frozen=True)
class SkillGroupAlias:
    name: str      # parent SkillGroup name
    alias: str     # e.g., "production", "staging"
    version: str   # group version string this alias points to
```

#### SkillAlias and SkillTag

```python
@dataclass(frozen=True)
class SkillAlias:
    name: str       # parent Skill name
    alias: str      # e.g., "production", "staging"
    version: str    # version string this alias points to

@dataclass(frozen=True)
class SkillTag:
    key: str
    value: str
```

Tags use the same structure for skill-level, version-level, and
group-level tags. The distinction is maintained at the storage and API
layer (separate tables, separate endpoints).

### Publish state and lifecycle

#### Per-version publish state

Each `SkillVersion` has an independent publish state:

| State | Meaning | Downstream surfacing |
|---|---|---|
| `draft` | Registered but not ready for consumption | Not surfaced |
| `published` | Ready for downstream use | Surfaced to discovery, traces, consumers |
| `deprecated` | Still functional but no longer recommended | Surfaced with deprecation signal |
| `retired` | Preserved for history, no longer active | Not surfaced |

Allowed transitions:

| From | To |
|---|---|
| `draft` | `published`, `retired` |
| `published` | `deprecated` |
| `deprecated` | `published`, `retired` |

`published` cannot return to `draft`. `deprecated` can return to
`published` (re-publish) for cases where a deprecation was premature.

#### Skill group version publish state

Each `SkillGroupVersion` has its own publish state lifecycle, following
the same transitions as `SkillVersion`. A group version's publish state
is independent of its member skills' publish states. Publishing a group
version does not require its member skill versions to be published,
though consumers will typically want to verify this.

#### Skill-level status

`Skill.status` is a separate lifecycle for the logical asset as a whole
(`active`, `deprecated`, `retired`). Setting a skill to `deprecated`
does not automatically change individual version publish states.

### Database schema

Twelve tables, created via a single Alembic migration. All tables are
workspace-scoped.

#### `skills`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `name` | `String(256)` | PK |
| `kind` | `String(20)` | default `'skill'`; `skill`, `agent`, `mcp-server`, `hook` |
| `description` | `String(5000)` | |
| `status` | `String(20)` | default `'active'` |
| `last_registered_version` | `String(256)` | |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

#### `skill_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, publisher-supplied |
| `source_type` | `String(20)` | nullable; `git`, `oci`, `zip`, etc. |
| `source` | `String(2048)` | nullable pointer to skill content |
| `content_digest` | `String(512)` | optional integrity digest |
| `publish_state` | `String(20)` | default `'draft'` |
| `run_id` | `String(32)` | optional MLflow run linkage |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, name)` references `skills`, CASCADE delete.

#### `skill_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

#### `skill_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

#### `skill_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `alias` | `String(256)` | PK |
| `version` | `String(256)` | target version string |

#### `skill_groups`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `name` | `String(256)` | PK |
| `description` | `String(5000)` | |
| `status` | `String(20)` | default `'active'` |
| `last_registered_version` | `String(256)` | |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

#### `skill_group_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, publisher-supplied |
| `source_type` | `String(20)` | optional; `git`, `oci`, `zip`, etc. |
| `source` | `String(2048)` | optional pointer to group artifact |
| `content_digest` | `String(512)` | optional integrity digest |
| `publish_state` | `String(20)` | default `'draft'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, name)` references `skill_groups`, CASCADE delete.

#### `skill_group_version_memberships`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK |
| `group_name` | `String(256)` | PK, FK to `skill_group_versions` |
| `group_version` | `String(256)` | PK, FK to `skill_group_versions` |
| `skill_name` | `String(256)` | PK, FK to `skill_versions` |
| `skill_version` | `String(256)` | PK, FK to `skill_versions` |

FK: `(workspace, group_name, group_version)` references `skill_group_versions`, CASCADE delete.
FK: `(workspace, skill_name, skill_version)` references `skill_versions`, RESTRICT delete.

#### `skill_group_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

#### `skill_group_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

#### `skill_group_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `alias` | `String(256)` | PK |
| `version` | `String(256)` | target group version string |

**Workspace handling.** All tables use `(workspace, ...)` as the leading
primary key components. Single-tenant deployments use `'default'`.

**Timestamps.** Set at the application layer via
`get_current_time_millis()`, not via DDL defaults.

### Abstract store interface

The store interface follows MLflow's abstract store pattern.

```python
from abc import abstractmethod


class AbstractSkillRegistryStore:
    # --- Skill operations ---

    @abstractmethod
    def create_skill(
        self, name: str, kind: str = "skill",
        description: str | None = None,
    ) -> Skill: ...

    @abstractmethod
    def get_skill(self, name: str) -> Skill: ...

    @abstractmethod
    def search_skills(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
    ) -> PagedList[Skill]: ...

    @abstractmethod
    def update_skill(
        self,
        name: str,
        description: str | None = None,
        status: SkillStatus | None = None,
    ) -> Skill: ...

    @abstractmethod
    def delete_skill(self, name: str) -> None: ...

    # --- SkillVersion operations ---

    @abstractmethod
    def create_skill_version(
        self,
        name: str,
        version: str,
        source_type: str | None = None,
        source: str | None = None,
        publish_state: SkillPublishState = SkillPublishState.DRAFT,
        content_digest: str | None = None,
        run_id: str | None = None,
    ) -> SkillVersion: ...

    @abstractmethod
    def get_skill_version(
        self, name: str, version: str,
    ) -> SkillVersion: ...

    @abstractmethod
    def get_skill_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillVersion: ...

    @abstractmethod
    def get_latest_skill_version(self, name: str) -> SkillVersion: ...

    @abstractmethod
    def search_skill_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]: ...

    @abstractmethod
    def update_skill_version(
        self,
        name: str,
        version: str,
        publish_state: SkillPublishState | None = None,
    ) -> SkillVersion: ...

    @abstractmethod
    def delete_skill_version(
        self, name: str, version: str,
    ) -> None: ...

    # --- Tag operations ---

    @abstractmethod
    def set_skill_tag(
        self, name: str, key: str, value: str,
    ) -> None: ...

    @abstractmethod
    def delete_skill_tag(self, name: str, key: str) -> None: ...

    @abstractmethod
    def set_skill_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None: ...

    @abstractmethod
    def delete_skill_version_tag(
        self, name: str, version: str, key: str,
    ) -> None: ...

    # --- Alias operations ---

    @abstractmethod
    def set_skill_alias(
        self, name: str, alias: str, version: str,
    ) -> None: ...

    @abstractmethod
    def delete_skill_alias(
        self, name: str, alias: str,
    ) -> None: ...

    # --- Pull operations ---

    @abstractmethod
    def pull_skill(
        self, name: str, destination: str,
        version: str | None = None,
        alias: str | None = None,
        source_type: str | None = None,
    ) -> str: ...

    @abstractmethod
    def pull_skill_group(
        self, name: str, destination: str,
        version: str | None = None,
        alias: str | None = None,
    ) -> str: ...

    # --- SkillGroup operations ---

    @abstractmethod
    def create_skill_group(
        self, name: str, description: str | None = None,
    ) -> SkillGroup: ...

    @abstractmethod
    def get_skill_group(self, name: str) -> SkillGroup: ...

    @abstractmethod
    def search_skill_groups(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
    ) -> PagedList[SkillGroup]: ...

    @abstractmethod
    def update_skill_group(
        self,
        name: str,
        description: str | None = None,
        status: SkillGroupStatus | None = None,
    ) -> SkillGroup: ...

    @abstractmethod
    def delete_skill_group(self, name: str) -> None: ...

    # --- SkillGroupVersion operations ---

    @abstractmethod
    def create_skill_group_version(
        self,
        name: str,
        version: str,
        members: list[SkillGroupVersionMembership],
        publish_state: SkillPublishState = SkillPublishState.DRAFT,
        source_type: str | None = None,
        source: str | None = None,
        content_digest: str | None = None,
    ) -> SkillGroupVersion: ...

    @abstractmethod
    def get_skill_group_version(
        self, name: str, version: str,
    ) -> SkillGroupVersion: ...

    @abstractmethod
    def get_skill_group_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillGroupVersion: ...

    @abstractmethod
    def get_latest_skill_group_version(
        self, name: str,
    ) -> SkillGroupVersion: ...

    @abstractmethod
    def search_skill_group_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
    ) -> PagedList[SkillGroupVersion]: ...

    @abstractmethod
    def update_skill_group_version(
        self,
        name: str,
        version: str,
        publish_state: SkillPublishState | None = None,
    ) -> SkillGroupVersion: ...

    @abstractmethod
    def delete_skill_group_version(
        self, name: str, version: str,
    ) -> None: ...

    # --- SkillGroup tag operations ---

    @abstractmethod
    def set_skill_group_tag(
        self, name: str, key: str, value: str,
    ) -> None: ...

    @abstractmethod
    def delete_skill_group_tag(
        self, name: str, key: str,
    ) -> None: ...

    @abstractmethod
    def set_skill_group_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None: ...

    @abstractmethod
    def delete_skill_group_version_tag(
        self, name: str, version: str, key: str,
    ) -> None: ...

    # --- SkillGroup alias operations ---

    @abstractmethod
    def set_skill_group_alias(
        self, name: str, alias: str, version: str,
    ) -> None: ...

    @abstractmethod
    def delete_skill_group_alias(
        self, name: str, alias: str,
    ) -> None: ...
```

### REST API

The REST API uses RESTful nested resource paths, following the pattern
from the MCP Server Registry proposal.

#### Skill endpoints

All paths relative to `/ajax-api/3.0/mlflow/skills`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create a skill |
| `GET` | `/` | Search skills |
| `GET` | `/{name}` | Get skill by name |
| `PATCH` | `/{name}` | Update skill fields |
| `DELETE` | `/{name}` | Delete skill (cascades) |
| `POST` | `/{name}/versions` | Create a skill version |
| `GET` | `/{name}/versions` | Search versions |
| `GET` | `/{name}/versions/{version}` | Get a specific version |
| `PATCH` | `/{name}/versions/{version}` | Update version |
| `DELETE` | `/{name}/versions/{version}` | Delete a version |
| `POST` | `/{name}/tags` | Set a skill-level tag |
| `DELETE` | `/{name}/tags/{key}` | Delete a skill-level tag |
| `POST` | `/{name}/versions/{version}/tags` | Set a version-level tag |
| `DELETE` | `/{name}/versions/{version}/tags/{key}` | Delete a version tag |
| `POST` | `/{name}/aliases` | Set an alias |
| `GET` | `/{name}/aliases/{alias}` | Resolve alias to `SkillVersion` |
| `DELETE` | `/{name}/aliases/{alias}` | Delete an alias |
| `POST` | `/{name}/pull` | Pull skill content from source to a local destination |

#### Skill group endpoints

All paths relative to `/ajax-api/3.0/mlflow/skill-groups`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create a skill group |
| `GET` | `/` | Search skill groups |
| `GET` | `/{name}` | Get group by name |
| `PATCH` | `/{name}` | Update group fields |
| `DELETE` | `/{name}` | Delete group (cascades versions) |
| `POST` | `/{name}/versions` | Create a group version with members |
| `GET` | `/{name}/versions` | Search group versions |
| `GET` | `/{name}/versions/{version}` | Get a specific group version |
| `PATCH` | `/{name}/versions/{version}` | Update group version publish state |
| `DELETE` | `/{name}/versions/{version}` | Delete a group version |
| `POST` | `/{name}/tags` | Set a group-level tag |
| `DELETE` | `/{name}/tags/{key}` | Delete a group-level tag |
| `POST` | `/{name}/versions/{version}/tags` | Set a group version tag |
| `DELETE` | `/{name}/versions/{version}/tags/{key}` | Delete a group version tag |
| `POST` | `/{name}/aliases` | Set a group alias |
| `GET` | `/{name}/aliases/{alias}` | Resolve group alias to version |
| `DELETE` | `/{name}/aliases/{alias}` | Delete a group alias |
| `POST` | `/{name}/pull` | Pull all group members from their sources |

#### Pagination and filtering

Search endpoints use page-token-based pagination and `filter_string`
expressions following existing MLflow conventions.

**Skills and skill groups:** `name LIKE '%review%'`, `status = 'active'`,
`kind = 'agent'`, `tags.team = 'platform'`

**Skill versions:** `publish_state = 'published'`,
`source_type = 'git'`, `tags.scan.prompt-injection.status = 'pass'`

**Skill group versions:** `publish_state = 'published'`,
`tags.approved = 'true'`

### Python SDK and CLI

The `mlflow.skills` module exposes top-level functions delegating to
`MlflowClient`, with a 1:1 mapping to the abstract store methods above.
Two CLI command groups (`mlflow skills` and `mlflow skill-groups`)
provide the same operations from the command line. See the basic
examples at the top of this RFC for usage.

### Pull semantics

`pull` resolves a skill or skill group to its source pointer(s) and
fetches content to a local destination directory. It is
source-type-aware:

| Source type | Pull behavior |
|---|---|
| `git` | `git clone` or `git archive` of the referenced path/ref |
| `oci` | `oci pull` of the referenced image/tag |
| `zip` | HTTP download and extract |

**Single skill pull.** Fetches the content at the skill version's
`source` to the destination directory. Returns an error if the skill
version has no `source`.

**Skill group pull.** Source resolution:
1. If the group version has a `source`, fetch the group artifact as a
   single unit to the destination directory.
2. Otherwise, pull each member individually from its own `source` to
   a subdirectory of the destination, named by the member's skill name.
   Members without a `source` are skipped with a warning.

This supports both distribution patterns: a monolithic plugin artifact
(single OCI image or Git repo) and an assembled plugin (members from
different sources).

If `content_digest` is set, `pull` verifies the fetched content
matches the digest and returns an error on mismatch.

`pull` is harness-agnostic — it downloads content but does not generate
harness-specific manifests or place files in harness-specific
directories. Harness-specific installation is covered in RFC-0006.

### Error handling

| Scenario | Error code | HTTP status |
|---|---|---|
| Skill, version, or group not found | `RESOURCE_DOES_NOT_EXIST` | 404 |
| Duplicate skill name, version, or group | `RESOURCE_ALREADY_EXISTS` | 409 |
| Invalid publish state transition | `INVALID_PARAMETER_VALUE` | 400 |
| Unknown source type | `INVALID_PARAMETER_VALUE` | 400 |
| Alias references non-existent version | `RESOURCE_DOES_NOT_EXIST` | 404 |
| Group version member references non-existent skill version | `RESOURCE_DOES_NOT_EXIST` | 404 |
| Delete skill version referenced by a group version | `INVALID_PARAMETER_VALUE` | 400 |
| Delete skill with versions referenced by a group | `INVALID_PARAMETER_VALUE` | 400 |
| Delete skill or group with no group references | Cascading delete (succeeds) | 200 |

### Workspace scoping

All skill registry operations are workspace-scoped, following the model
registry pattern:

- Workspace is resolved via `resolve_entity_workspace_name()`
- Single-tenant deployments use `"default"`
- All database queries filter by workspace
- The REST API derives workspace from the authenticated caller's context
- Version, tag, alias, and group membership operations inherit workspace
  from their parent entity

Cross-workspace sharing (e.g., a platform team publishing skills
visible to all workspaces) is not addressed by this RFC. This is a
cross-registry concern that applies equally to skills, MCP servers,
and other AI asset registries. It is expected to be solved at the
platform level across all MLflow registries rather than piecemeal in
each one.

### UI

The Skills page lives under the GenAI workflow in the MLflow sidebar,
alongside Experiments, Prompts, and other AI asset pages.

The list view shows skills and skill groups in a card-based or table
layout, with name, description, latest version, status, and tags. Users
can filter by status, source type, and search by name or description. A
toggle switches between individual skills and skill groups.

The detail view for a skill shows metadata, version list, aliases, tags
(including security scan results), and group memberships.

The detail view for a skill group shows its description, status, version
list, aliases, and tags. Each group version shows its publish state and
the pinned skill versions it contains.

### Security scan tracking

The registry does not perform security scans. It provides a metadata
layer for recording and querying scan results using version-level tags.

Recommended tag conventions:

| Tag key | Example value | Description |
|---|---|---|
| `scan.prompt-injection.status` | `pass`, `fail`, `warning` | Scan result |
| `scan.prompt-injection.date` | `2026-04-22` | When the scan was run |
| `scan.prompt-injection.tool` | `garak-0.9` | Which tool performed the scan |
| `scan.code-vuln.status` | `pass` | Code vulnerability scan result |
| `scan.code-vuln.date` | `2026-04-22` | When the scan was run |

These are conventions, not enforced schema. Organizations can define
additional scan tag prefixes for their own scanning tools and criteria.

The publish state lifecycle supports scan-gated promotion workflows:
a skill version stays in `draft` until scans pass, then is moved to
`published`. The registry does not enforce this workflow, but the
combination of publish state and scan tags makes it easy to implement.

## Drawbacks

- **Source pointer validity.** The registry stores source pointers but
  cannot guarantee they remain valid. The optional `content_digest`
  field mitigates content tampering but does not prevent link rot. This
  is inherent to a metadata-first design.
- **No artifact storage.** This design does not provide a self-contained
  backup of skill content. If the source system goes away, the metadata
  remains but the content is lost.

# Alternatives

## Store skill artifacts directly in MLflow

Store skill bundles (SKILL.md + scripts + assets) as MLflow artifacts
alongside the metadata.

Rejected because skills are already versioned and stored in Git, OCI, or
other systems. Source pointers federate across distribution mechanisms
naturally; artifact storage forces centralization. Organizations that
want artifact backup can use OCI registries, which already provide
versioned, content-addressable storage.

## Use Git alone (no registry)

Continue using Git repositories as the sole mechanism for skill
management.

This is sufficient for individual developers and small teams. This RFC
proposes a governance layer on top of Git for enterprises that need
publish-state lifecycle, security scan tracking, and federated discovery.
The two approaches are complementary.

# Adoption strategy

This is a new feature, not a breaking change. Adoption is incremental:

**This RFC (RFC-0005):**
- Entities, database schema, store implementation, REST API, Python SDK,
  CLI, and basic UI.
- Users can register capabilities of any kind (skill, agent, mcp-server,
  hook), manage publish state, record scan results as tags, organize
  capabilities into skill groups, and discover published capabilities.
- `mlflow skills pull` fetches content from registered sources.
- Existing MLflow functionality is unaffected.

**Companion RFC (RFC-0006):**
- Harness-specific installation: `mlflow skills install` generates
  manifests and places files for specific agent harnesses.
- Initial targets: Claude Code, Codex CLI, Cursor, with additional
  harnesses based on demand.

**Follow-up:**
- Agent trace integration: traces automatically record which registered
  capability version was used, linking back to the registry.
- Usage analytics dashboard based on trace metadata.
- Additional source types and capability kinds as demand emerges.
