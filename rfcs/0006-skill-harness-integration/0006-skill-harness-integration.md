---
start_date: 2026-04-27
mlflow_issue: https://github.com/mlflow/mlflow/issues/22833
rfc_pr: https://github.com/mlflow/rfcs/pull/10
---

# RFC: Skill Registry Harness Integration

| Author(s)              | Bill Murdock (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-04-27 |
| **AI Assistant(s)**    | Claude Code (Opus 4.6) |

# Summary

Add harness-specific installation to the MLflow Skill Registry
(RFC-0005). Where RFC-0005 provides `mlflow skills pull` to fetch
content from registered sources to a local directory, this RFC adds
`mlflow skills install` to generate harness-specific manifests, place
files in the correct directories, and configure the agent harness to
use the installed capabilities.

This bridges the gap between "I found a skill group in the registry"
and "my agent harness can use it."

# Basic example

## Install a skill group for Claude Code

```bash
mlflow skills install --group pr-workflow --alias production \
    --harness claude-code
```

This resolves the `pr-workflow` skill group, pulls all member
capabilities from their registered sources, and generates:

```
.claude/plugins/pr-workflow/
  .claude-plugin/plugin.json      # Generated manifest
  skills/
    code-review/SKILL.md          # Pulled from Git source
  agents/
    security-auditor.md           # Pulled from Git source
  .mcp.json                       # Generated from mcp-server members
```

## Install for other harnesses

```bash
# Codex CLI (nearly identical to Claude Code)
mlflow skills install --group pr-workflow --alias production \
    --harness codex-cli

# Cursor
mlflow skills install --group pr-workflow --alias production \
    --harness cursor

# Antigravity
mlflow skills install --group pr-workflow --alias production \
    --harness antigravity
```

## Python SDK

```python
import mlflow

mlflow.skills.install_skill_group(
    name="pr-workflow",
    alias="production",
    harness="claude-code",
    destination=".",  # project root
)
```

## Motivation

### The problem

RFC-0005 provides a governed registry with `pull` for fetching content
to a local directory. But each agent harness has its own conventions
for where files go, what manifests are needed, and how capabilities
are discovered:

- **Claude Code / Codex CLI** expect a `plugin.json` manifest, skills
  in `skills/`, agents in `agents/`, and MCP configs in `.mcp.json`.
- **Cursor** discovers skills from `.cursor/skills/`, agents from
  `.cursor/agents/`, and MCP servers from `.cursor/mcp.json`.
- **Antigravity** discovers skills from `.agent/skills/`.
- **OpenClaw** expects skills in `skills/` directories and uses
  `openclaw.plugin.json`.
- **GitHub Copilot** uses `plugin.json` with skills, agents, hooks,
  and MCP configs.

Without harness-specific installation, users must manually:
1. Run `mlflow skills pull` to get the content
2. Create the appropriate manifest files
3. Place files in harness-specific directories
4. Configure the harness to discover the new capabilities

This is error-prone and discourages adoption.

### The cross-harness landscape

The following table summarizes the capability types and installation
conventions across major agent harnesses:

| Harness | Skills | Agents | MCP | Hooks | Manifest | Install dir |
|---|---|---|---|---|---|---|
| Claude Code | SKILL.md | agent .md | .mcp.json | settings.json | plugin.json | `.claude/plugins/` |
| Codex CLI | SKILL.md | agent .md | .mcp.json | hooks | plugin.json | `.codex/plugins/` |
| Cursor | SKILL.md | agent .md | mcp.json | -- | -- | `.cursor/skills/`, `.cursor/agents/` |
| GitHub Copilot | skills/ | agents/ | .mcp.json | hooks/*.json | plugin.json | project |
| OpenClaw | SKILL.md | -- | -- | plugin hooks | openclaw.plugin.json | `skills/` |
| Kilo Code | SKILL.md | custom modes | mcp.json | -- | -- | project |
| Antigravity | SKILL.md | -- | -- | -- | -- | `.agent/skills/` |
| OpenCode | .md/.ts | agent configs | config | JS events | -- | `.opencode/` |
| Continue | -- | config.yaml | mcpServers/ | -- | -- | `.continue/` |
| Windsurf | -- | -- | mcp_config.json | -- | -- | project |
| Amazon Q | -- | -- | mcp.json | -- | -- | `.amazonq/` |
| Goose | -- | -- | MCP only | -- | -- | config |
| Zed | -- | profiles | settings.json | -- | -- | config |

Key insight: the SKILL.md file format is portable across harnesses —
only the directory placement and manifest format differ.

### Out of scope

- **Registry operations.** Registration, versioning, lifecycle,
  search, and governance are covered in RFC-0005.
- **Harness-specific features beyond installation.** This RFC does not
  extend harness functionality (e.g., adding hook support to harnesses
  that lack it).
- **Automatic harness detection.** The user specifies `--harness`
  explicitly. Auto-detection could be a follow-up.

## Detailed design

### Harness adapters

Each supported harness has an adapter that knows how to:

1. **Map capability kinds to harness paths.** Given the registry's
   `kind` field (skill, agent, mcp-server, hook), determine where
   each capability's content should be placed.
2. **Generate manifests.** Create harness-specific manifest files
   (e.g., `plugin.json`, `.mcp.json`) from registry metadata.
3. **Handle unsupported kinds.** Skip capability kinds the harness
   does not support, with a warning.

```python
from abc import abstractmethod


class HarnessAdapter:
    @abstractmethod
    def install_skill_group(
        self,
        group_version: SkillGroupVersion,
        members: list[tuple[Skill, SkillVersion]],
        destination: str,
    ) -> str: ...

    @abstractmethod
    def supported_kinds(self) -> set[str]: ...
```

### Claude Code / Codex CLI adapter

These two harnesses share nearly identical plugin formats. The adapter
generates:

**`plugin.json`:**
```json
{
  "name": "pr-workflow",
  "version": "1.0.0",
  "description": "End-to-end pull request review workflow",
  "author": { "name": "Generated by MLflow Skill Registry" }
}
```

**Directory layout:**
```
{destination}/.claude/plugins/{group-name}/
  .claude-plugin/plugin.json
  skills/{skill-name}/SKILL.md      # kind=skill members
  agents/{agent-name}.md            # kind=agent members
  .mcp.json                         # kind=mcp-server members, merged
```

For Codex CLI, the path uses `.codex/plugins/` instead.

**MCP server merging.** If the group contains multiple `mcp-server`
members, their configs are merged into a single `.mcp.json` file
using server name as the key:

```json
{
  "mcpServers": {
    "github-mcp": { ... },
    "jira-mcp": { ... }
  }
}
```

**Hook handling.** `hook` members are placed in the plugin directory.
The adapter generates appropriate entries but does not modify the
user's `settings.json` — the user must explicitly enable hooks for
security reasons.

### Cursor adapter

Cursor does not have a plugin bundle format. The adapter places
capabilities directly into Cursor's discovery directories:

```
{destination}/.cursor/skills/{skill-name}/SKILL.md   # kind=skill
{destination}/.cursor/agents/{agent-name}.md          # kind=agent
```

For MCP servers, the adapter merges entries into the project's
`.cursor/mcp.json`, adding new servers without overwriting existing
ones.

Hooks are skipped with a warning (Cursor does not support hooks).

### Antigravity adapter

```
{destination}/.agent/skills/{skill-name}/SKILL.md   # kind=skill
```

Agents, MCP servers, and hooks are skipped with a warning.

### Other harness adapters

Additional adapters (OpenClaw, GitHub Copilot, Kilo Code, OpenCode,
Continue, etc.) follow the same pattern: map kinds to paths, generate
manifests, skip unsupported kinds with warnings.

New adapters can be contributed without changes to the registry or
the adapter interface.

### Marketplace integration

Some harnesses (Claude Code, Codex CLI) support marketplace catalogs:
a JSON endpoint that lists available plugins so users can browse and
install them natively from within the harness. The registry serves a
`marketplace.json` endpoint that exposes published skill groups as
installable plugins, enabling native harness-driven installation
without requiring the MLflow CLI.

#### Endpoint

```
GET /ajax-api/3.0/mlflow/skill-groups/marketplace.json?harness=claude-code
```

Query parameters:

| Parameter | Required | Description |
|---|---|---|
| `harness` | yes | Target harness (`claude-code`, `codex-cli`) |
| `filter_string` | no | Filter expression (e.g., `tags.team = 'platform'`) |

The endpoint returns only skill groups whose latest published version
contains at least one member with a kind supported by the target
harness.

#### Response format

The response follows the harness's native marketplace schema. For
Claude Code / Codex CLI:

```json
{
  "plugins": [
    {
      "name": "pr-workflow",
      "version": "1.0.0",
      "description": "End-to-end pull request review workflow",
      "author": { "name": "Generated by MLflow Skill Registry" },
      "source": "https://mlflow.example.com/ajax-api/3.0/mlflow/skill-groups/pr-workflow/install?harness=claude-code",
      "skills": ["code-review", "test-coverage"],
      "agents": ["security-auditor"],
      "mcpServers": ["github-mcp"]
    }
  ]
}
```

Each entry is derived from a published skill group version and its
members. The `source` field points to a registry endpoint that serves
the installable plugin bundle.

#### Configuration

Users add the registry as a marketplace source in their harness
settings:

```bash
# Claude Code settings.json
{
  "extraKnownMarketplaces": [
    "https://mlflow.example.com/ajax-api/3.0/mlflow/skill-groups/marketplace.json?harness=claude-code"
  ]
}
```

Once configured, users can browse and install registry plugins
natively:

```
/plugin install pr-workflow@mlflow
```

This is the recommended installation path for Claude Code and Codex
CLI users. It provides the most seamless experience and keeps the
harness as the single point of plugin management.

#### Limitations

Marketplace integration is only available for harnesses with
marketplace infrastructure (currently Claude Code and Codex CLI).
Harnesses without marketplace support (Cursor, Antigravity, OpenClaw)
use the adapter-based `mlflow skills install` command instead.

### Store interface

```python
class AbstractSkillRegistryStore:
    # ... (existing methods from RFC-0005) ...

    @abstractmethod
    def install_skill(
        self,
        name: str,
        harness: str,
        destination: str,
        version: str | None = None,
        alias: str | None = None,
        source_type: str | None = None,
    ) -> str: ...

    @abstractmethod
    def install_skill_group(
        self,
        name: str,
        harness: str,
        destination: str,
        version: str | None = None,
        alias: str | None = None,
    ) -> str: ...

    @abstractmethod
    def generate_marketplace(
        self,
        harness: str,
        filter_string: str | None = None,
    ) -> dict: ...
```

### REST API

Additional endpoints on the skill and skill group resources:

| Method | Path | Description |
|---|---|---|
| `POST` | `/ajax-api/3.0/mlflow/skills/{name}/install` | Install a single capability for a harness |
| `POST` | `/ajax-api/3.0/mlflow/skill-groups/{name}/install` | Install a skill group for a harness |
| `GET` | `/ajax-api/3.0/mlflow/skill-groups/marketplace.json` | Generate marketplace catalog for a harness |

### CLI

```bash
# Install a single capability
mlflow skills install --name code-review --alias production \
    --harness claude-code --destination .

# Install a skill group
mlflow skills install --group pr-workflow --alias production \
    --harness claude-code --destination .

# List supported harnesses
mlflow skills harnesses
```

## Drawbacks

- **Adapter maintenance.** Each harness adapter must be maintained as
  harness plugin formats evolve. This is ongoing work.
- **Incomplete coverage.** Not all harnesses support all capability
  kinds. Users may be surprised when hooks are silently skipped for
  Cursor, or agents are skipped for Antigravity.
- **Manifest format drift.** Generated manifests may not cover all
  features of a harness's native plugin format (e.g., Codex CLI's
  `interface` block with branding, or OpenClaw's `requires` field).

# Alternatives

## Let users write their own install scripts

Provide only `pull` (RFC-0005) and let users or third parties build
harness-specific tooling.

Rejected because the gap between "pull" and "working in my harness"
is the main adoption barrier. A first-party install experience is
critical for driving adoption.

# Adoption strategy

**Initial release:**
- Claude Code and Codex CLI adapters (highest impact, nearly identical
  format).
- Cursor adapter (second-highest priority for MLflow's user base).
- `marketplace.json` generation for Claude Code / Codex CLI.

**Follow-up:**
- Additional harness adapters based on demand.
- Automatic harness detection from project structure.
- Bi-directional sync: detect locally installed plugins and register
  them in the registry.
