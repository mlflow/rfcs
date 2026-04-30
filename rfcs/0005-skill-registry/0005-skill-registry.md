---
start_date: 2026-04-22
mlflow_issue: https://github.com/mlflow/mlflow/issues/22833
rfc_pr: https://github.com/mlflow/rfcs/pull/10
---

# RFC: Skill Registry

| Author(s)              | Bill Murdock (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-04-29 |
| **AI Assistant(s)**    | Claude Code (Opus 4.6) |

# Summary

Add a Skill Registry to MLflow: a governed, metadata-first registry for
AI agent capabilities. The registry stores metadata and typed source
pointers (to Git repos, OCI registries, ZIP archives, etc.) rather
than artifacts directly. It provides enterprise governance on top of
existing distribution mechanisms: lifecycle management, security scan
tracking, usage analytics via traces, and federated discovery across
sources.

The registry tracks four capability kinds under the `mlflow.genai.skills`
SDK namespace (CLI: `mlflow skills`):

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

## Register a skill

```python
import mlflow

# Create the logical skill asset
skill = mlflow.genai.skills.create_skill(
    name="code-review",
    description="Reviews pull requests for correctness, style, and security",
)

# Register a version pointing to a Git source
version = mlflow.genai.skills.create_skill_version(
    name="code-review",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/agent-skills/tree/v1.0.0/code-review",
    content_digest="sha256:a3f2b8c...",
)
# version.status == "active"

# Set an alias for stable resolution
mlflow.genai.skills.set_skill_alias(
    name="code-review",
    alias="production",
    version="1.0.0",
)

# Record security scan results (see "Security scan tracking" for convention)
mlflow.genai.skills.set_skill_version_tag(
    name="code-review",
    version="1.0.0",
    key="scan.prompt-injection.status",
    value="pass",
)
mlflow.genai.skills.set_skill_version_tag(
    name="code-review",
    version="1.0.0",
    key="scan.prompt-injection.date",
    value="2026-04-29",
)
mlflow.genai.skills.set_skill_version_tag(
    name="code-review",
    version="1.0.0",
    key="scan.prompt-injection.tool",
    value="promptfoo/1.2.0",
)
```

## Create a skill group with a versioned membership snapshot

```python
from mlflow.entities import SkillGroupVersionMembership

# Create a group for related skills
group = mlflow.genai.skills.create_skill_group(
    name="pr-workflow",
    description="End-to-end pull request review workflow",
)

# Create a group version that pins specific skill versions
group_version = mlflow.genai.skills.create_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
    members=[
        SkillGroupVersionMembership(
            member_name="code-review", member_version="1.0.0",
        ),
        SkillGroupVersionMembership(
            member_name="test-coverage", member_version="2.1.0",
        ),
        SkillGroupVersionMembership(
            member_name="security-scan", member_version="1.0.0",
        ),
    ],
)

# Set an alias for stable resolution
mlflow.genai.skills.set_skill_group_alias(
    name="pr-workflow",
    alias="production",
    version="1.0.0",
)
```

## Register other capability kinds

```python
# Register a sub-agent
mlflow.genai.skills.create_skill(
    name="security-auditor",
    kind="agent",
    description="Security specialist for auth and payment code",
)
mlflow.genai.skills.create_skill_version(
    name="security-auditor",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/agent-skills/tree/v1.0.0/security-auditor",
)

# Register an MCP server
mlflow.genai.skills.create_skill(
    name="github-mcp",
    kind="mcp-server",
    description="GitHub integration via MCP",
)
mlflow.genai.skills.create_skill_version(
    name="github-mcp",
    version="2.0.0",
    source_type="oci",
    source="ghcr.io/acme/github-mcp:2.0.0",
    content_digest="sha256:b4e9f1d...",
)

# Register a hook
mlflow.genai.skills.create_skill(
    name="pre-commit-scan",
    kind="hook",
    description="Runs security scan before tool commits",
)
mlflow.genai.skills.create_skill_version(
    name="pre-commit-scan",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/agent-skills/tree/v1.0.0/pre-commit-scan",
)
```

## Create a skill group with mixed capability kinds

```python
from mlflow.entities import SkillGroupVersionMembership

group = mlflow.genai.skills.create_skill_group(
    name="pr-workflow",
    description="End-to-end pull request review workflow",
)

# A group version can bundle skills, agents, and MCP server references
group_version = mlflow.genai.skills.create_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
    members=[
        SkillGroupVersionMembership(
            member_name="code-review", member_version="1.0.0",
        ),
        SkillGroupVersionMembership(
            member_name="security-auditor", member_version="1.0.0",
        ),
        # Reference an MCP server from the MCP registry (RFC-0004)
        SkillGroupVersionMembership(
            member_name="github-mcp", member_version="2.0.0",
            registry="mcp",
        ),
    ],
)
```

## Pull skills to a local directory

```python
# Pull a single skill version
mlflow.genai.skills.pull_skill(
    name="code-review",
    alias="production",
    destination="./skills/code-review",
)

# Pull an entire skill group (all members)
mlflow.genai.skills.pull_skill_group(
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
# Search for active skill versions
versions = mlflow.genai.skills.search_skill_versions(
    name="code-review",
    filter_string="status = 'active'",
)

# Search for active skill groups
groups = mlflow.genai.skills.search_skill_groups(
    filter_string="status = 'active'",
)

# Get a specific version
version = mlflow.genai.skills.get_skill_version(
    name="code-review",
    version="1.0.0",
)
# version.source_type == "git"
# version.source == "https://github.com/acme/agent-skills/tree/v1.0.0/code-review"

# Resolve by alias
version = mlflow.genai.skills.get_skill_version_by_alias(
    name="code-review",
    alias="production",
)

# Get a group version and its pinned skill versions
group_version = mlflow.genai.skills.get_skill_group_version(
    name="pr-workflow",
    version="1.0.0",
)
# group_version.members == [SkillGroupVersionMembership(...), ...]

# Resolve a group alias
group_version = mlflow.genai.skills.get_skill_group_version_by_alias(
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

# Alias
mlflow skills set-alias --name code-review --alias production \
    --version 1.0.0

# Create a group and a versioned membership snapshot
mlflow skill-groups create --name pr-workflow \
    --description "End-to-end PR review workflow"
mlflow skill-groups create-version --name pr-workflow --version 1.0.0 \
    --member code-review:1.0.0 \
    --member test-coverage:2.1.0 \
    --member security-scan:1.0.0 \
    --member mcp:github-mcp:2.0.0
mlflow skill-groups set-alias --name pr-workflow --alias production \
    --version 1.0.0

# Search active skill versions
mlflow skills search-versions --name code-review \
    --filter "status = 'active'"

# Search active groups
mlflow skill-groups search --filter "status = 'active'"
```

## Motivation

### The problem

AI agent capabilities — skills, sub-agents, MCP server configurations,
and hooks — are becoming a critical asset class in enterprise AI
platforms. As organizations adopt agentic AI, they accumulate these
capabilities across teams, repositories, and agent harnesses.

A cross-harness portable format is emerging around these capabilities.
The registry is format-agnostic but is designed to interoperate with
the conventions gaining adoption across agent harnesses:

- **SKILL.md** — a markdown file with structured instructions for the
  agent. Supported by Claude Code, Codex CLI, Cursor, GitHub Copilot,
  OpenClaw, Kilo Code, and Antigravity. This is the most broadly
  portable format for skills and agents.
- **MCP server configs** — JSON configuration for Model Context
  Protocol servers. MCP is a universal tool extension protocol
  supported by nearly all major harnesses.
- **Hooks** — event-triggered shell commands or scripts. Less
  standardized; Claude Code and Codex CLI have the most mature hook
  support.
- **Plugin bundles** — harness-specific packaging of skills, agents,
  MCP configs, and hooks into a single installable unit. Claude Code
  and Codex CLI use `plugin.json` manifests; other harnesses use
  directory conventions.

Today, these capabilities are managed as ad-hoc files in Git
repositories. This works well for individual developers and small
teams. GitHub provides versioning, collaboration, and access control.

However, enterprises face governance challenges that Git alone does not
address:

1. **No status lifecycle.** Git has no concept of "this version is
   approved for production use" vs. "this is deprecated." Teams resort
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

**Platform administrator** — A platform admin at Acme Corp registers
their team's code-review skill, pointing to its Git source. They
create a version, record a prompt-injection scan result as a tag, and
group it with a security-auditor agent and a GitHub MCP server into a
"pr-workflow" skill group. They set the group's `production` alias to
the tested version. When a newer version introduces a vulnerability,
they deprecate it — downstream consumers resolving `production` are
unaffected because the alias still points to the safe version.

**Developer** — A developer starting a new project searches the
registry for active skills filtered by `kind = 'skill'`. They find
the `pr-workflow` group, resolve its `production` alias, and run
`mlflow skills pull --group pr-workflow --alias production` to fetch
all member content locally. They can also browse and install directly
from their agent harness if marketplace integration is configured
([RFC-0006](../0006-skill-harness-integration/0006-skill-harness-integration.md)).

**Security engineer** — A security engineer queries scan tags across
all skill versions to find capabilities that haven't been scanned
recently (`tags.scan.prompt-injection.date < '2026-01-01'`). They
deprecate versions that fail re-scanning and track compliance posture
across the organization's registered capabilities.

### Out of scope

- **Artifact storage as the only path.** The registry supports both
  external source pointers (Git, OCI, ZIP) and direct artifact storage
  (`source_type="mlflow"`). However, it is not an artifact-only store;
  the metadata-first, source-pointer model remains the primary design.
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
- **Approval workflows or review gates.** Status transitions are
  sufficient for initial governance.
- **Detailed UI/UX design.** This RFC describes the UI surface and
  placement but does not specify interaction patterns.

## Detailed design

### Entities and data model

```mermaid
erDiagram
Skill ||--o{ SkillVersion : "has versions"
Skill ||--o{ SkillTag : "has tags"
Skill ||--o{ SkillAlias : "has aliases"
SkillAlias ||--o{ SkillAliasHistory : "has history"
SkillVersion ||--o{ SkillVersionTag : "has tags"
SkillGroup ||--o{ SkillGroupVersion : "has versions"
SkillGroup ||--o{ SkillGroupTag : "has tags"
SkillGroup ||--o{ SkillGroupAlias : "has aliases"
SkillGroupAlias ||--o{ SkillGroupAliasHistory : "has history"
SkillGroupVersion ||--o{ SkillGroupVersionMembership : "contains members"
SkillGroupVersion ||--o{ SkillGroupVersionTag : "has tags"
SkillGroupVersionMembership }o--o| SkillVersion : "references (registry=skill)"
SkillGroupVersionMembership }o--o| MCPServerVersion : "references (registry=mcp)"
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
    DELETED = "deleted"


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
    latest_version_alias: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Stable logical asset name, unique within a workspace |
| `kind` | `SkillKind` | Capability type: `skill`, `agent`, `mcp-server`, `hook` |
| `status` | `SkillStatus` | Read-only, derived from the latest version's status |
| `aliases` | `list[SkillAlias]` | Stable version pointers (e.g., `production` → `1.2.0`) |
| `last_registered_version` | `str` | Most recently registered version string (read-only, auto-updated) |
| `latest_version_alias` | `str` | Optional alias name to resolve as "latest" (e.g., `"production"`). If unset, `get_latest_skill_version` falls back to `creation_timestamp` |
| `workspace` | `str` | Visibility boundary |

**Kind extensibility.** The `kind` enum covers the four capability
types with broad cross-harness support. New kinds can be added without
schema changes since the column stores a string value. `kind` is
immutable after creation.

**MCP servers: two registration paths.** The MCP server registry
(RFC-0004) is the default and recommended path for registering MCP
servers. It provides deployment tracking via hosted bindings,
deduplication across skill groups, and the full MCP governance model.
Skill groups reference MCP registry entries via `registry="mcp"` in
their membership.

`kind=mcp-server` in this registry is reserved for MCP configs that
are embedded in a group-level artifact (e.g., an OCI image containing
a complete plugin with an `.mcp.json` file). These are not
independently managed and exist only as part of their containing
artifact. Standalone MCP servers should always be registered in the
MCP registry, not as skills.

#### SkillVersion

A versioned record containing a typed source pointer, status, and
tags.

```python
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
    status: SkillStatus = SkillStatus.ACTIVE
    content_digest: str | None = None
    tags: dict[str, str] = field(default_factory=dict)
    aliases: list[str] = field(default_factory=list)
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
| `status` | `SkillStatus` | Per-version lifecycle: `active`, `deprecated`, `deleted` |
| `aliases` | `list[str]` | Alias names currently pointing at this version (read-only, projected from alias table) |
| `run_id` | `str` | Optional MLflow run association for trace linkage |

**Source type extensibility.** The `source_type` enum is intentionally
small for the initial implementation. New source types (e.g., `s3`,
`azure-blob`) can be added without schema changes since the column
stores a string value.

**MLflow artifact storage (`source_type="mlflow"`).** In addition to
external source pointers, the registry supports storing skill content
directly in MLflow's artifact storage. This serves users who do not
have external Git/OCI infrastructure, who want agent capabilities
stored alongside their models, or who operate in airgapped
environments where external sources are not reachable.

Content is stored as a directory tree of individual files under an
artifact path, consistent with how MLflow stores model artifacts. For
example, a skill with a SKILL.md, scripts, and reference material is
stored as separate artifacts under a version-specific prefix:

```
skills/code-review/1.0.0/
  SKILL.md
  scripts/analyze.sh
  scripts/lint-config.json
  reference/style-guide.md
```

The `source` field contains the MLflow artifact URI (e.g.,
`mlflow-artifacts:/skills/code-review/1.0.0/`). Pull downloads the
directory tree from the artifact store. The MLflow UI can browse
individual files within a stored skill version.

The upload API accepts a local directory path and stores each file as
a separate artifact. The `content_digest` is computed over the full
directory contents at upload time.

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
register a new version. Mutable fields (`status`, `tags`) can be
updated independently.

#### SkillGroup

The logical group asset, scoped to a workspace. A skill group bundles
capabilities of any kind (skills, agents, MCP servers, hooks) into a
governed unit that maps to the "plugin" concept in agent harnesses.
Follows the same pattern as Skill: a top-level entity with versions,
tags, and aliases.

```python
@dataclass
class SkillGroup:
    name: str
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus = SkillStatus.ACTIVE
    tags: dict[str, str] = field(default_factory=dict)
    aliases: list["SkillGroupAlias"] = field(default_factory=list)
    last_registered_version: str | None = None
    latest_version_alias: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`SkillGroup.status` is read-only, derived from the latest group
version's status. `latest_version_alias` works the same as on `Skill`.

**Why groups instead of tags?** Tags on individual skills could
express "these skills are related" but cannot provide:

- **Versioned membership snapshots.** A group version pins specific
  member versions, so "pr-workflow v1.0.0" always means the same set
  of capabilities. Tags are mutable and cannot capture a reproducible
  point-in-time combination.
- **Cross-registry references.** A group version can reference both
  skill registry members and MCP server registry members (RFC-0004).
  Tags on individual skills cannot express this cross-registry
  relationship.
- **Group-level source.** A group version can have its own source
  pointer (e.g., a single OCI image containing a complete plugin).
  Tags cannot carry source metadata.
- **Independent lifecycle.** A group version has its own status,
  aliases, and tags. The group can be deprecated independently of its
  members. With tags, lifecycle management would have to be inferred
  from individual skill states.
- **Plugin mapping.** Agent harnesses (Claude Code, Codex CLI) model
  plugins as bundles of capabilities with a manifest. Skill groups
  map directly to this concept; tags do not.

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
    status: SkillStatus = SkillStatus.ACTIVE
    tags: dict[str, str] = field(default_factory=dict)
    members: list["SkillGroupVersionMembership"] = field(default_factory=list)
    aliases: list[str] = field(default_factory=list)
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
(`status`, `tags`) can be updated independently.

#### SkillGroupVersionMembership

Each membership entry pins a specific versioned asset from either the
skill registry or the MCP server registry (RFC-0004). The `registry`
field indicates which registry the member comes from. The parent group
identity is provided by the enclosing `SkillGroupVersion`; the storage
layer adds those columns as FKs.

```python
@dataclass(frozen=True)
class SkillGroupVersionMembership:
    member_name: str
    member_version: str
    registry: str = "skill"  # "skill" or "mcp"
```

| Field | Type | Description |
|---|---|---|
| `member_name` | `str` | Name of the member asset in the target registry |
| `member_version` | `str` | Version of the member asset |
| `registry` | `str` | Which registry the member comes from: `skill` (this registry) or `mcp` (MCP server registry, RFC-0004) |

When `registry="skill"`, the member references a `SkillVersion` in
this registry. When `registry="mcp"`, the member references an
`MCPServerVersion` in the MCP server registry (RFC-0004). This
cross-registry reference enables:

- **Deduplication.** Two skill groups that both need `github-mcp`
  reference the same MCP registry entry. No duplicate configs.
- **Runtime status.** The MCP registry tracks deployment state via
  hosted bindings (`is_deployed`, `endpoint_url`). Install-time
  tooling can check whether a referenced MCP server is already
  running rather than starting a duplicate.
- **Single source of truth.** MCP server definitions are governed in
  the MCP registry; skill groups reference them rather than carrying
  standalone copies.

A member can appear in multiple groups and multiple group versions.
Membership is at the version level, so a group version is a
reproducible snapshot of "these specific asset versions work together."

**Group-level source and embedded MCP configs.** When a group version
has a group-level source (e.g., a single OCI image containing a
complete plugin), the artifact may include MCP configs alongside
skills and agents. In this case, MCP servers do not need separate
membership entries or MCP registry references — they are part of the
artifact. Cross-registry MCP references are for the case where MCP
servers are independently registered and managed.

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

#### Alias audit trail

Alias changes are auditable. Every call to `set_skill_alias`,
`delete_skill_alias`, `set_skill_group_alias`, or
`delete_skill_group_alias` appends a record to an append-only history
table. This supports governance questions like "who promoted this to
production and when?" or "what was production pointing to before the
incident?"

```python
@dataclass(frozen=True)
class SkillAliasHistory:
    name: str              # parent Skill name
    alias: str             # e.g., "production"
    old_version: str | None  # previous target (None if alias was created)
    new_version: str | None  # new target (None if alias was deleted)
    changed_by: str | None
    timestamp: int | None    # millis since epoch
```

History is recorded automatically by the store on every alias
mutation. The same structure applies to `SkillGroupAliasHistory`.

History records are read-only and append-only. They cannot be modified
or deleted through the API.

### Status and lifecycle

This lifecycle aligns with the MCP Server Registry (RFC-0004).

#### Per-version status

Each `SkillVersion` and `SkillGroupVersion` has an independent status:

| State | Meaning | Downstream surfacing |
|---|---|---|
| `active` | Ready for downstream use | Surfaced to discovery, traces, consumers |
| `deprecated` | Still functional but no longer recommended | Surfaced with deprecation signal |
| `deleted` | Soft-deleted; preserved for history, no longer active | Not surfaced |

New versions default to `active` upon creation.

Allowed transitions:

| From | To |
|---|---|
| `active` | `deprecated` |
| `deprecated` | `active`, `deleted` |

`deprecated` can return to `active` (re-activate) for cases where a
deprecation was premature.

#### Skill-level and group-level status

`Skill.status` and `SkillGroup.status` are read-only, derived from the
latest version's status. This follows the MCP Server Registry pattern
where the parent entity's status reflects its latest version.

#### `latest_version_alias` resolution

`get_latest_skill_version(name)` resolves the "latest" version:

1. If `Skill.latest_version_alias` is set, resolve that alias to a
   version.
2. If unset, fall back to the version with the most recent
   `creation_timestamp`.

`latest_version_alias` is mutable via `update_skill()`. It stores an
alias name (e.g., `"production"`), providing a level of indirection:
the user says "latest means whatever `production` points to." The same
pattern applies to `SkillGroup` and `get_latest_skill_group_version`.

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
| `last_registered_version` | `String(256)` | |
| `latest_version_alias` | `String(256)` | optional; alias name to resolve as "latest" |
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
| `status` | `String(20)` | default `'active'` |
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

#### `skill_alias_history`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | FK |
| `name` | `String(256)` | FK |
| `alias` | `String(256)` | |
| `old_version` | `String(256)` | nullable; null on alias creation |
| `new_version` | `String(256)` | nullable; null on alias deletion |
| `changed_by` | `String(256)` | |
| `timestamp` | `BigInteger` | millis since epoch; PK with workspace, name, alias |

Append-only. No updates or deletes through the API.

#### `skill_groups`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `name` | `String(256)` | PK |
| `description` | `String(5000)` | |
| `last_registered_version` | `String(256)` | |
| `latest_version_alias` | `String(256)` | optional; alias name to resolve as "latest" |
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
| `status` | `String(20)` | default `'active'` |
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
| `member_name` | `String(256)` | PK |
| `member_version` | `String(256)` | PK |
| `registry` | `String(20)` | PK, default `'skill'`; `skill` or `mcp` |

FK: `(workspace, group_name, group_version)` references `skill_group_versions`, CASCADE delete.
FK: `(workspace, member_name, member_version)` references `skill_versions`, RESTRICT delete. Only applies when `registry='skill'`.

**Cross-registry references (`registry='mcp'`).** There is no
database-level FK for MCP registry references. Referential integrity
is enforced at the application layer: the store validates that the
referenced `MCPServerVersion` exists when creating a group version
and returns `RESOURCE_DOES_NOT_EXIST` if it does not. This avoids
deployment-ordering dependencies between RFC-0004 and RFC-0005
migrations and allows either registry to be deployed independently.

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

#### `skill_group_alias_history`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | FK |
| `name` | `String(256)` | FK |
| `alias` | `String(256)` | |
| `old_version` | `String(256)` | nullable; null on alias creation |
| `new_version` | `String(256)` | nullable; null on alias deletion |
| `changed_by` | `String(256)` | |
| `timestamp` | `BigInteger` | millis since epoch; PK with workspace, name, alias |

Append-only. No updates or deletes through the API.

**Workspace handling.** All tables use `(workspace, ...)` as the leading
primary key components. Single-tenant deployments use `'default'`.

**Timestamps.** Set at the application layer via
`get_current_time_millis()`, not via DDL defaults.

### Store interface

The store interface follows the mixin pattern established by the MCP
Server Registry (RFC-0004). Methods raise `NotImplementedError` rather
than using `@abstractmethod`, allowing stores that don't support skills
(e.g., `FileStore`) to work without stubbing every method.

```python
class SkillRegistryMixin:
    # --- Skill operations ---

    def create_skill(
        self, name: str, kind: str = "skill",
        description: str | None = None,
    ) -> Skill:
        raise NotImplementedError

    def get_skill(self, name: str) -> Skill:
        raise NotImplementedError

    def search_skills(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]:
        raise NotImplementedError

    def update_skill(
        self,
        name: str,
        description: str | None = None,
        latest_version_alias: str | None = None,
    ) -> Skill:
        raise NotImplementedError

    def delete_skill(self, name: str) -> None:
        raise NotImplementedError

    # --- SkillVersion operations ---

    def create_skill_version(
        self,
        name: str,
        version: str,
        source_type: str | None = None,
        source: str | None = None,
        content_digest: str | None = None,
        run_id: str | None = None,
    ) -> SkillVersion:
        raise NotImplementedError

    def get_skill_version(
        self, name: str, version: str,
    ) -> SkillVersion:
        raise NotImplementedError

    def get_skill_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillVersion:
        raise NotImplementedError

    def get_latest_skill_version(self, name: str) -> SkillVersion:
        raise NotImplementedError

    def search_skill_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]:
        raise NotImplementedError

    def update_skill_version(
        self,
        name: str,
        version: str,
        status: SkillStatus | None = None,
    ) -> SkillVersion:
        raise NotImplementedError

    def delete_skill_version(
        self, name: str, version: str,
    ) -> None:
        raise NotImplementedError

    # --- Tag operations ---

    def set_skill_tag(
        self, name: str, key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_tag(self, name: str, key: str) -> None:
        raise NotImplementedError

    def set_skill_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_version_tag(
        self, name: str, version: str, key: str,
    ) -> None:
        raise NotImplementedError

    # --- Alias operations ---

    def set_skill_alias(
        self, name: str, alias: str, version: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_alias(
        self, name: str, alias: str,
    ) -> None:
        raise NotImplementedError

    def get_skill_alias_history(
        self,
        name: str,
        alias: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
    ) -> PagedList[SkillAliasHistory]:
        raise NotImplementedError

    # --- SkillGroup operations ---

    def create_skill_group(
        self, name: str, description: str | None = None,
    ) -> SkillGroup:
        raise NotImplementedError

    def get_skill_group(self, name: str) -> SkillGroup:
        raise NotImplementedError

    def search_skill_groups(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillGroup]:
        raise NotImplementedError

    def update_skill_group(
        self,
        name: str,
        description: str | None = None,
        latest_version_alias: str | None = None,
    ) -> SkillGroup:
        raise NotImplementedError

    def delete_skill_group(self, name: str) -> None:
        raise NotImplementedError

    # --- SkillGroupVersion operations ---

    def create_skill_group_version(
        self,
        name: str,
        version: str,
        members: list[SkillGroupVersionMembership],
        source_type: str | None = None,
        source: str | None = None,
        content_digest: str | None = None,
    ) -> SkillGroupVersion:
        raise NotImplementedError

    def get_skill_group_version(
        self, name: str, version: str,
    ) -> SkillGroupVersion:
        raise NotImplementedError

    def get_skill_group_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillGroupVersion:
        raise NotImplementedError

    def get_latest_skill_group_version(
        self, name: str,
    ) -> SkillGroupVersion:
        raise NotImplementedError

    def search_skill_group_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillGroupVersion]:
        raise NotImplementedError

    def update_skill_group_version(
        self,
        name: str,
        version: str,
        status: SkillStatus | None = None,
    ) -> SkillGroupVersion:
        raise NotImplementedError

    def delete_skill_group_version(
        self, name: str, version: str,
    ) -> None:
        raise NotImplementedError

    # --- SkillGroup tag operations ---

    def set_skill_group_tag(
        self, name: str, key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_group_tag(
        self, name: str, key: str,
    ) -> None:
        raise NotImplementedError

    def set_skill_group_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_group_version_tag(
        self, name: str, version: str, key: str,
    ) -> None:
        raise NotImplementedError

    # --- SkillGroup alias operations ---

    def set_skill_group_alias(
        self, name: str, alias: str, version: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_group_alias(
        self, name: str, alias: str,
    ) -> None:
        raise NotImplementedError

    def get_skill_group_alias_history(
        self,
        name: str,
        alias: str | None = None,
        max_results: int = 100,
        page_token: str | None = None,
    ) -> PagedList[SkillGroupAliasHistory]:
        raise NotImplementedError
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
| `GET` | `/{name}/aliases/history` | Get alias change history (all aliases) |
| `GET` | `/{name}/aliases/{alias}/history` | Get alias change history (specific alias) |

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
| `PATCH` | `/{name}/versions/{version}` | Update group version status |
| `DELETE` | `/{name}/versions/{version}` | Delete a group version |
| `POST` | `/{name}/tags` | Set a group-level tag |
| `DELETE` | `/{name}/tags/{key}` | Delete a group-level tag |
| `POST` | `/{name}/versions/{version}/tags` | Set a group version tag |
| `DELETE` | `/{name}/versions/{version}/tags/{key}` | Delete a group version tag |
| `POST` | `/{name}/aliases` | Set a group alias |
| `GET` | `/{name}/aliases/{alias}` | Resolve group alias to version |
| `DELETE` | `/{name}/aliases/{alias}` | Delete a group alias |
| `GET` | `/{name}/aliases/history` | Get alias change history (all aliases) |
| `GET` | `/{name}/aliases/{alias}/history` | Get alias change history (specific alias) |

#### Pagination and filtering

Search endpoints use page-token-based pagination and `filter_string`
expressions following existing MLflow conventions.

**Skills and skill groups:** `name LIKE '%review%'`, `status = 'active'`,
`kind = 'agent'`, `tags.team = 'platform'`

**Skill versions:** `status = 'active'`,
`source_type = 'git'`, `tags.scan.prompt-injection.status = 'pass'`

**Skill group versions:** `status = 'active'`,
`tags.approved = 'true'`

### Python SDK and CLI

The `mlflow.genai.skills` module exposes top-level functions delegating to
`MlflowClient`, with a 1:1 mapping to the store mixin methods above.
Two CLI command groups (`mlflow skills` and `mlflow skill-groups`)
provide the same operations from the command line. See the basic
examples at the top of this RFC for usage.

`pull` is implemented in the SDK/CLI layer, not the store mixin. The
client calls `get_skill_version` (or resolves an alias) to obtain the
source pointer, then fetches content locally using source-type-specific
logic (git clone, OCI pull, ZIP download). This keeps the store as a
pure data-access layer.

### Pull semantics

`pull` is a client-side operation. The SDK reads the source pointer
from the registry via the REST API, then fetches content directly
from the source system to the caller's local filesystem. The registry
server is not involved in content transfer. `pull` is
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
| Invalid status transition | `INVALID_PARAMETER_VALUE` | 400 |
| Unknown source type | `INVALID_PARAMETER_VALUE` | 400 |
| Alias references non-existent version | `RESOURCE_DOES_NOT_EXIST` | 404 |
| Group version member references non-existent version (skill or MCP) | `RESOURCE_DOES_NOT_EXIST` | 404 |
| Delete skill version referenced by a group version | `INVALID_PARAMETER_VALUE` | 400 |
| Delete skill with versions referenced by a group | `INVALID_PARAMETER_VALUE` | 400 |
| Delete MCP server version referenced by a group version | `INVALID_PARAMETER_VALUE` | 400 |
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

### Permissions

The skill registry integrates with MLflow's existing permission
framework (READ / EDIT / MANAGE), applied at the `Skill` and
`SkillGroup` level. Versions, tags, aliases, and memberships inherit
permissions from their parent entity.

| Permission | Operations |
|---|---|
| `READ` | Search skills and groups, get versions, resolve aliases, list tags and memberships |
| `EDIT` | Create skills and groups, create versions, set and delete tags, update description |
| `MANAGE` | Status transitions (deprecate, delete), set and delete aliases, delete versions, delete skills and groups, manage permissions |

Key design choices:

- **Status transitions require MANAGE.** Deprecating or deleting a
  capability version affects all downstream consumers. This is a
  governance action, not a routine edit, and should require elevated
  permissions.
- **Alias management requires MANAGE.** Aliases like `production`
  control which version downstream consumers resolve to. Changing an
  alias has the same blast radius as a status transition.
- **Tag edits require EDIT.** Tags (including scan result tags) are
  operational metadata. Requiring MANAGE for scan tags would create
  friction for CI/CD scan integrations that need to record results
  automatically.
- **Creator gets MANAGE.** When a user creates a skill or group, they
  automatically receive MANAGE permission, following the MLflow model
  registry pattern.

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
list, aliases, and tags. Each group version shows its status and the
pinned member versions it contains.

### Security scan tracking

The registry does not perform security scans. It provides a metadata
layer for recording and querying scan results using version-level tags
with a reserved `scan.*` namespace.

**Tag namespace convention.** All security scan tags use the pattern
`scan.{scan-type}.{field}`, where `{scan-type}` identifies the scan
(e.g., `prompt-injection`, `code-vulnerability`, `secrets-detection`)
and `{field}` is one of the following defined keys:

| Field | Expected values | Description |
|---|---|---|
| `status` | `pass`, `fail`, `error` | Scan outcome |
| `date` | ISO 8601 date (e.g., `2026-04-29`) | When the scan was run |
| `tool` | Tool name/version (e.g., `promptfoo/1.2.0`) | Which tool performed the scan |
| `details` | URL or free text | Link to full results or summary |

**Example tags on a skill version:**

| Tag key | Value |
|---|---|
| `scan.prompt-injection.status` | `pass` |
| `scan.prompt-injection.date` | `2026-04-29` |
| `scan.prompt-injection.tool` | `promptfoo/1.2.0` |
| `scan.code-vulnerability.status` | `fail` |
| `scan.code-vulnerability.date` | `2026-04-28` |
| `scan.code-vulnerability.tool` | `semgrep/1.67.0` |
| `scan.code-vulnerability.details` | `https://scans.acme.com/results/abc123` |

**Convention, not schema.** These are documented conventions, not
server-enforced schema. The registry does not validate that `status`
is one of the expected values or that `date` is a valid ISO 8601
string. This is a deliberate tradeoff: the scan tool landscape is
evolving rapidly, and a flexible convention allows organizations to
adopt new scan types without schema changes. Organizations can define
additional `scan.{type}` prefixes for their own scanning tools.

**UI rendering.** The convention gives the UI enough structure to
detect `scan.*.status` tags and render a scan summary (e.g., a green
check or red X per scan type) without requiring a dedicated entity.

**Querying.** Scan results are queryable using the existing filter
syntax: `tags.scan.prompt-injection.status = 'pass'` or
`tags.scan.code-vulnerability.date < '2026-01-01'`.

**Scan-gated workflows.** The status lifecycle supports scan-gated
deprecation: organizations can deprecate versions that fail scans and
use scan result tags to filter for safe versions. The registry does
not enforce this workflow, but the combination of status and scan tags
makes it straightforward to implement.

**Future evolution.** If scan patterns stabilize and the convention
proves insufficient (e.g., organizations need server-side validation,
separate permissions for scan results, or richer scan metadata),
structured scan metadata can be added as a first-class entity in a
follow-up without breaking the tag-based approach.

## Drawbacks

- **Source pointer validity.** The registry stores source pointers but
  cannot guarantee they remain valid. The optional `content_digest`
  field mitigates content tampering but does not prevent link rot. This
  is inherent to a metadata-first design.
- **No artifact storage.** This design does not provide a self-contained
  backup of skill content. If the source system goes away, the metadata
  remains but the content is lost.

# Alternatives

## Store skill artifacts only in MLflow (no source pointers)

Make MLflow artifact storage the sole storage mechanism, with no
support for external source pointers.

Rejected because most organizations already manage skills in Git or
OCI. Source pointers federate across existing distribution mechanisms
without requiring migration. The current design supports both:
`source_type="mlflow"` for direct artifact storage alongside
`source_type="git"`, `"oci"`, and `"zip"` for external sources.

## Use Git alone (no registry)

Continue using Git repositories as the sole mechanism for skill
management.

This is sufficient for individual developers and small teams. This RFC
proposes a governance layer on top of Git for enterprises that need
status lifecycle, security scan tracking, and federated discovery.
The two approaches are complementary.

# Adoption strategy

This is a new feature, not a breaking change. Adoption is incremental:

**This RFC (RFC-0005):**
- Entities, database schema, store implementation, REST API, Python SDK,
  CLI, and basic UI.
- Users can register capabilities of any kind (skill, agent, mcp-server,
  hook), manage status lifecycle, record scan results as tags, organize
  capabilities into skill groups, and discover active capabilities.
- Source types include `git`, `oci`, `zip`, and `mlflow` (direct
  artifact storage).
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
- Cross-workspace export/import for promoting assets between
  workspaces or instances. This should follow whatever pattern the
  other MLflow registries adopt rather than designing a serialization
  format in isolation.
