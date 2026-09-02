# RFC 0010: Extended Agent Plugins

| start_date   | 2026-07-20 |
| :----------- | :--------- |
| mlflow_issue | https://github.com/mlflow/mlflow/issues/22833 |
| rfc_pr       | https://github.com/mlflow/rfcs/pull/27 |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-08-24 |
| **AI Assistant(s)**    | Claude Code |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [Open questions](#open-questions)

# Summary

[RFC-0008](https://github.com/mlflow/rfcs/blob/main/rfcs/0008-mvp-skill-registry/0008-mvp-skill-registry.md)
registers agent plugins as governed versions of the open
[Agent Plugins](https://agent-plugins.org/) package format. Each
version preserves its complete canonical `plugin.json` manifest, and
skill members receive individual registry entries and membership rows
that connect the plugin to registered skill versions. Everything else
a plugin contains is preserved in the stored package but is invisible
to governance: it cannot be discovered through search, tracked in
composition views, or queried for impact analysis.

This RFC extends agent plugin membership to the rest of the plugin,
in two tiers that mirror the Agent Plugins standard itself.

**MCP server references become first-class members.** MCP is part of
the Agent Plugins standard, and MLflow already has a registry for MCP
servers
([RFC-0004](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md)).
This RFC connects the two: an agent plugin version can reference a
registered MCP server version the same way it references registered
skill versions, instead of carrying opaque configuration. That single
connection is the main value of this RFC. It makes questions like
"which plugins depend on this MCP server?" answerable from the
registry, lets deprecation of an MCP server version surface every
plugin it would affect, and gives plugin consumers a governed path
from a plugin to the servers it needs.

**Other component types get generic membership.** Today the Agent
Plugins standard defines skills and MCP; real plugins carry more.
Claude Code plugins include agents, hooks, commands, LSP servers, and
other component types, and other harnesses have their own additions
(Cursor rules, Copilot prompts). Rather than a registry entity per
type, membership rows carry a free-form `member_type` string that
MLflow treats as an opaque label. This keeps MLflow compatible with
Claude Code plugins now, absorbs whatever the standard adds next
without schema changes, and supports harness-specific extensions
indefinitely. If the standard never grows beyond skills and MCP, it
becomes the base layer that each harness builds on, and the generic
mechanism is how those extensions stay visible to governance.

**Membership is derived on import and supplied on assembly**,
extending the split RFC-0008 established for skills. When a plugin is
imported, the import adapter derives member rows for every component
type it recognizes from the manifest and package content; the
manifest remains the source of truth and the rows are a queryable
index of it. When a plugin is assembled from registered parts,
supplying the references is what assembly means: skill references and
MCP server references are passed explicitly, and both point into
their own registries.

The plugin remains the governance unit. Skills and MCP servers keep
their independent lifecycles in their own registries, and membership
rows reference specific versions of them. Other member types have no
independent lifecycle: they are governed through the plugin's
versioning, status, aliases, and tags.

# Basic example

```python
import mlflow

mlflow.genai.register_agent_plugin(
    name="pr-workflow",
    version="2.0.0",
    skills=[
        "skills:/code-review/1",
        "skills:/style-check/2",
    ],
    mcp_servers=[
        "mcp-servers:/database-connector/2.0.0",
    ],
)
```

The skill references resolve in the skill registry (RFC-0008) and the
MCP server reference resolves in the MCP server registry (RFC-0004).
Importing an existing plugin needs no member arguments at all; the
import adapter derives membership from the package content.

## Motivation

### The problem

1. **Plugins are more than skills.** The Agent Plugins standard
   itself defines two component types, skills and MCP servers, and
   RFC-0008 registers only the skills; a plugin's MCP content is
   preserved as opaque package configuration. Beyond the standard,
   harnesses extend the format with their own component types: a
   Claude Code plugin can contain agents, hooks, commands, and LSP
   servers, a Copilot plugin carries instructions and prompts, and a
   Cursor plugin carries `.mdc` rules. All of that is likewise
   preserved but unregistered. A developer browsing a plugin in the
   registry sees its skills and must inspect the raw artifact to
   learn what else installing it would bring.

2. **MCP dependencies are invisible to the registries that govern
   both sides.** The Agent Plugins standard includes MCP, and MLflow
   governs MCP servers in their own registry, yet a registered plugin
   that uses an MCP server carries that fact only as configuration
   inside its package. An administrator deprecating an MCP server
   version cannot ask "which plugins reference this?" A platform team
   evaluating a plugin cannot see, from the registry, what server
   infrastructure it requires. The two registries exist; the edge
   between them does not.

3. **Composition questions have no query.** "Which plugins reference
   an MCP server?" "Which plugins install hooks?" "What component
   types are in use across the organization?" None of these are
   answerable today without pulling and inspecting every plugin
   artifact.

The component type landscape is large and growing: Claude Code alone
defines ten or more component types, package managers such as APM and
Lola define their own taxonomies across many harness targets, and
each harness adds types no other harness supports. An entity type per
component, each with its own tables, endpoints, and CLI commands,
would not scale and would put MLflow in the business of tracking
every harness's taxonomy. A single opaque type label on membership
rows scales with the ecosystem instead.

### User journeys

These journeys illustrate the workflows that extended membership
enables. Each builds on RFC-0008's agent plugin infrastructure and
RFC-0004's MCP server registry.

#### Assemble an agent plugin with skills and an MCP server

A platform team packages a data pipeline workflow whose skills need a
database MCP server that is already registered and governed in the
MCP server registry.

1. Register an assembled agent plugin version that pins skill members
   and references the MCP server:
   ```bash
   mlflow agent-plugins register --name data-pipeline \
       --version 1.0.0 \
       --skill skills:/sql-query/1 \
       --skill skills:/data-transform/1 \
       --mcp-server mcp-servers:/database-connector/2.0.0
   ```
   The MCP server reference names a version registered through
   RFC-0004. MLflow stores a cross-registry reference; it does not
   copy or embed the server's configuration into the plugin.
   **UI path:** on the agent plugin creation page, add members by
   searching registered skills and registered MCP servers.
2. View the composition:
   ```bash
   mlflow agent-plugins get-version agent-plugins:/data-pipeline/1.0.0
   ```
   The output lists members grouped by type: two skills and one MCP
   server, each with its registry coordinates.
   **UI path:** the plugin version detail page shows members grouped
   by type. Skill members link to skill detail pages; the MCP server
   member links to its detail page in the MCP server registry.
3. When the plugin is installed into a harness, the package manager
   plugin resolves the MCP server reference from the MCP server
   registry and produces the harness-specific configuration for it,
   alongside the installed skills.

#### Assess the impact of deprecating an MCP server version

An administrator is about to deprecate an MCP server version and
needs to know which plugins would be affected. This is the journey
that motivates cross-registry references: it cannot be answered at
all when server configuration is embedded in plugin packages.

1. Find the plugins that reference the version:
   ```bash
   mlflow agent-plugins search-versions \
       --filter "member.type = 'mcp-server' \
                 AND member.name = 'database-connector'"
   ```
   The results list every plugin version whose membership references
   the server, with the referenced server version on each row.
2. Notify the owners of the affected plugins, then deprecate the
   server version in the MCP server registry.
3. Plugin owners assemble new plugin versions referencing the
   replacement server version and move their aliases forward.
   **UI path:** the MCP server version detail page (RFC-0004) shows a
   "Referenced by" section listing the agent plugin versions that
   reference it, with a deprecation banner once the version is
   deprecated. The plugin detail page shows the deprecation signal on
   the affected member.

#### Import a plugin with non-skill content

A team imports an existing Claude Code plugin. RFC-0008's import
already detects the format, registers the skills, preserves the rest
of the content, and warns about what it could not register. With this
RFC, the warning becomes membership.

1. Import the plugin exactly as in RFC-0008; there is no new flag:
   ```bash
   mlflow agent-plugins import \
       --source https://github.com/acme/plugins.git \
       --ref v2.0.0 --subpath pr-workflow
   ```
2. The import adapter derives members for every component type it
   recognizes, using the format's own conventions:
   ```
   Discovered 5 members:
     skill: code-review
     skill: style-check
     mcp-server: database-connector
     agent: security-auditor
     hook:  pre-commit-scan

   Created agent plugin: pr-workflow v2.0.0 (5 members)
   ```
   Skills are registered as in RFC-0008. Non-skill members get
   membership rows with the `member_type` the adapter assigned.
   MLflow does not interpret the types; the adapter that understands
   the format assigns them.
3. Content the adapter does not recognize is preserved in the stored
   package without a membership row, as in RFC-0008, and the import
   output notes it.

The MCP configuration discovered during import produces an
`mcp-server` member row, so the plugin's server dependency is visible
and searchable even though the configuration itself stays in the
package. Whether import should go further and connect that member to
a matching registered MCP server in the MCP server registry is an
open question (see below).

#### Register a plugin from a harness with its own component types

A team maintains a Cursor plugin that includes skills and `.mdc`
instruction rules, a component type no other harness uses. Format
adapters are pluggable, so support for a harness's conventions does
not require changes to MLflow itself.

1. Import the plugin. The Cursor format adapter discovers skills by
   the standard layout and instruction rules by Cursor's conventions:
   ```
   Discovered 3 members:
     skill: api-design
     skill: error-handling
     instruction: coding-standards

   Created agent plugin: cursor-coding-standards v1.0.0 (3 members)
   ```
   The `instruction` member type is assigned by the adapter. MLflow
   does not need to know what a `.mdc` file is.
2. Install for Cursor. The Cursor package manager plugin places each
   member according to its type; MLflow passes all members to the
   plugin regardless of type.

#### Discover plugins by component type

An administrator wants to understand the governance surface across
all registered plugins.

1. Search plugins by member type:
   ```bash
   mlflow agent-plugins search-versions \
       --filter "member.type = 'hook'"
   ```
   The results list plugin versions that install hooks, regardless of
   which harness format they came from.
2. Summarize the member types in use across the registry to see the
   full component surface: how many plugins carry each type, and
   which types exist at all. New types appear in this view
   automatically as plugins that use them are imported; no MLflow
   change is involved.
   **UI path:** the plugin gallery shows member type badges on each
   card, and a filter panel narrows the gallery by member type.

#### Trace non-skill member execution

A team running the `pr-workflow` plugin wants execution visibility
across all of its members, not only the skills.

[RFC-0009](https://github.com/mlflow/rfcs/pull/37) links traces to
registered skills through spans that carry registry coordinates. The
same linkage generalizes to other member types: when a bundled agent
is invoked or a bundled hook fires, the corresponding span can carry
the plugin membership coordinates (plugin name and version, member
name and type), so "which plugin version was active?" is answerable
for every member type the same way it is for skills. The
instrumentation mechanics, and which member types each harness can
attribute, follow RFC-0009's design and are not restated here.

### Out of scope

- **Independent lifecycle for generic member types.** Skills and MCP
  servers have their own registries and lifecycles, and membership
  references specific versions of them. Other member types have no
  standalone registry entities, no independent versioning, and no
  member-level status, aliases, or tags. The plugin is their
  governance unit. If a component type later needs a full lifecycle,
  that is a separate proposal.
- **Format validation of non-skill content.** MLflow records member
  metadata but does not parse or validate hook schemas, agent
  frontmatter, or instruction rule syntax. Validation belongs to the
  package manager plugin and the harness.
- **MCP server hosting and runtime.** Deployment and runtime
  management of referenced MCP servers remain with the MCP server
  registry (RFC-0004) and the MCP Gateway. This RFC only adds the
  reference edge between the registries.
- **A new package format.** Plugins remain canonical Agent Plugins
  packages. Membership rows index the package; they do not extend or
  alter the `plugin.json` payload.
- **Security scan metadata.** As in RFC-0008, structured scan results
  are a cross-registry capability, not a member-type feature.

# Detailed design

TBD.

# Drawbacks

TBD.

# Alternatives

TBD.

# Adoption strategy

TBD.

# Open questions

- **MCP reference validation timing.** Should a cross-registry MCP
  server reference be validated when the plugin version is created
  (fail if the referenced server version does not exist) or resolved
  only at pull and install time?
- **MCP references discovered during import.** Import records an
  `mcp-server` member for configuration it discovers. When that
  configuration matches a registered MCP server, should import
  connect the member to the registry entry automatically, require the
  user to confirm the match, or leave the member unconnected?
- **Reference syntax for MCP servers.** RFC-0008 defines the
  `skills:/` and `agent-plugins:/` URI schemes; RFC-0004 does not
  define one for MCP servers. Does this RFC introduce an
  `mcp-servers:/` scheme, and does RFC-0004 adopt it?
- **Member type vocabulary.** Types assigned by import adapters are
  free-form strings. Should MLflow document a recommended vocabulary
  for common types (agent, hook, command, instruction) so equivalent
  components from different formats group together in discovery, or
  is convergence left to adapters?
