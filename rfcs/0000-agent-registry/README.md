---
start_date: 2026-10-08
mlflow_issue: # TBD
rfc_pr:
---

# Agent Registry

## Summary

Make agents the primary GenAI container in MLflow. Each agent has a persistent identity and
immutable snapshots of its declared composition that connect source code and AI asset dependencies
with traces and evaluations. Teams can compare snapshots, choose what to deploy based on evidence,
and investigate production issues. Other agents can discover available agents through the registry.
Eligible experiments can be converted to agents, while experiments remain supported for existing
workflows and model training.

OSS MLflow provides the default registry backend. Managed deployments can instead resolve agent
records from an authoritative catalog or gateway, such as Unity AI Gateway. This draft seeks
agreement on product direction; storage and API details remain subject to follow-up design.

## Motivation

An agent should have one persistent identity in MLflow from development through production. Its
registry page would bring snapshots, source code, AI asset dependencies, evaluations, traces, and
endpoints together. Each snapshot would show its prompts, models, tools, skills, and plugins
alongside evaluation results and production quality, latency, and cost, with links to the underlying
traces. Teams could compare candidates, choose what to deploy, and investigate production
regressions using the same record.

MLflow's built-in AI Assistant or an external AI agent using MLflow's APIs could use this context as
well. If latency rises after an upgrade, it could compare the snapshots' source and dependencies
with slow calls in traces, then cite the evidence supporting its diagnosis.

Today, MLflow captures traces and evaluations but organizes GenAI work primarily under experiments,
even for production applications. "Experiment" suggests a temporary development activity. An
experiment's agent versions are currently logged models which provide version tracking, but there is
no persistent agent registry identity connecting discovery, governance, and registered dependencies
to that evidence. The proposal keeps the development loop—tracing, evaluating, and revising prompts
or tools—under the agent identity.

### Why MLflow

MLflow holds the behavioral evidence used to decide whether an agent snapshot is ready and to
explain how it behaves after deployment. Linking that evidence to an authoritative agent record puts
snapshot and dependency context where teams and assistants inspect it. OSS MLflow supplies the
default registry. A managed deployment can keep an existing catalog or gateway (e.g. Unity AI
Gateway) authoritative while MLflow resolves its agent records.

### Out of scope

- Hosting agents, proxying invocations, or managing runtime health and scaling.

## User journeys

A **snapshot** is MLflow's immutable record of an agent's declared composition, resolved assets, and
source metadata when available. Traces and evaluations refer to the agent container or a snapshot
ID. A source **revision** is a Git commit when the code is committed, or an author-supplied marker
for uncommitted code; the marker does not save that code.

Tracing and evaluation still work as they do today. Snapshots connect those results to recorded
source and declared AI assets; trace links show which assets a request actually used when available.

### Start tracing an agent

1. Create an agent in the UI. It gets a persistent identity before any snapshot or endpoint is
   registered.
2. Select it by ID with `mlflow.set_agent(...)` and run the instrumented application.
3. Open the agent's **Traces** page. When available, traces with a snapshot also link to recorded
   source (such as a Git commit) and declared assets.

### Trace an agent running in a harness

1. Put the agent ID and declared assets in `mlflow-agent.yaml` and enable the MLflow Claude Code
   tracing plugin.
2. Start a session. The plugin detects the harness version and, when skill tracing is available,
   registered skills installed in Claude Code's effective skill directories. MLflow creates or
   reuses a snapshot from the resolved inputs.
3. Open the agent's **Traces** page. The session links to its snapshot; skill links show which
   available skills were activated. With only an agent ID and no manifest, the trace remains under
   the agent without a snapshot.

### Test an agent update

1. Declare the agent in a manifest and run `mlflow.genai.evaluate`. MLflow infers the current Git
   commit as the source revision from a clean checkout when none is supplied; the run records the
   snapshot ID.
2. Change the code or its declared assets and evaluate again. A new commit or resolved asset creates
   a snapshot. If the Git checkout is dirty, the author supplies a `revision` to distinguish the
   local code state.
3. Compare the runs by snapshot, source, and observed assets.

For a production prompt A/B test, register the candidate as a new snapshot with its prompt
dependency pinned. The application splits traffic between the baseline and candidate snapshots;
MLflow groups evaluations and traces by snapshot so users can compare quality, latency, and cost,
with links to the prompt version actually used when available.

### Update a production prompt alias

1. A production agent declares its system prompt through the prompt's `production` alias. Its traces
   link agent snapshot 7 to prompt version 3.
2. The prompt alias moves to version 4. When the client next resolves it, MLflow creates agent
   snapshot 8; traces already in flight retain the snapshots and assets they used.
3. New traces link snapshot 8 to prompt version 4. The owner compares their quality and cost with
   snapshot 7; moving the prompt alias back reuses snapshot 7.

### Investigate a production cost increase

1. Notice a cost spike on the agent's **Usage** page and ask MLflow's AI Assistant or a coding agent
   to investigate.
2. It notices a recent agent snapshot and compares the snapshots' recorded source and declared
   assets. It checks affected traces to see which assets were used and which calls drove costs.
3. It suggests a likely cause, such as a newly added skill increasing token use, and links to
   supporting traces so the owner can verify it.

### Find an agent to reuse

1. A user searches the UI for a capability, or a running agent calls `search_agents` through the
   MLflow MCP server, SDK, or REST API when it needs another agent.
2. MLflow returns accessible endpoints with their advertised capabilities and agent snapshots. The
   user can inspect ownership, source, assets, and evaluation evidence; the running agent selects a
   match from the returned metadata.
3. The user integrates the chosen endpoint, or the running agent retrieves details with `get_agent`
   and invokes the endpoint during the request.

### Review a fleet of agents

1. An operator creates a fleet and adds several cooperating agents.
2. A developer with read access opens the fleet to compare quality and usage, then filters its
   traces and sessions to selected agents.
3. From the fleet's **Agents** tab, they open one agent to investigate its activity.

## Detailed design

### Overview of competitors

These products establish useful requirements and opportunities for MLflow:

| Platform   | Relevant capability                                                                                                                                                                                                                                                                                                                                                       | Requirement for MLflow                                                                             | MLflow opportunity                                                                                                              |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| AWS        | [Agent Registry](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry.html) supports discovery and approval, while [AgentCore Evaluations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations.html) scores traces.                                                                                                               | Discover approved agents and sync external descriptors.                                            | Bring agent snapshots, dependencies, traces, and evaluations into the same registry review workflow.                            |
| GCP        | [Agent Registry](https://docs.cloud.google.com/agent-registry/register-agents) indexes A2A Agent Cards for discovery; [bindings](https://docs.cloud.google.com/agent-registry/manage-bindings) connect agents to resources; [agent details](https://docs.cloud.google.com/agent-registry/manage-agents) show evaluation and observability for supported managed runtimes. | Import A2A capabilities and represent agent dependencies explicitly.                               | Provide a consistent version and evidence view for managed and externally hosted agents.                                        |
| Databricks | [Agent Services](https://docs.databricks.com/aws/en/ai-gateway/agent-services) register external agents as Unity Catalog assets with connections and permissions.                                                                                                                                                                                                         | Register external agents with ownership, permissions, and endpoints; no MLflow packaging required. | Add version, dependency, trace, and evaluation lineage while Unity Catalog can remain authoritative in the Databricks offering. |
| Microsoft  | [Agent Registry](https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-registry) provides organization-wide inventory; [Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/development-lifecycle) supports versioning, tracing, and evaluation.                                                                                        | Support immutable snapshots and comparisons alongside ownership and discovery controls.            | Link each snapshot to registered assets and trace/evaluation evidence across runtimes in OSS MLflow.                            |

### Agents as the new GenAI container

#### What that means

The GenAI landing page becomes **Agents**. Users can create an agent in the UI or SDK, or convert an
eligible experiment. Agents can be registered before deployment, so an endpoint is optional.

An agent author can declare its composition in `mlflow-agent.yaml`:

```yaml
agent_id: { fromEnv: MLFLOW_AGENT_ID }
version: { fromEnv: APP_VERSION }
sources:
  - repository: https://github.com/example/support-agent
    revision: { fromEnv: GIT_COMMIT }
assets:
  prompts:
    system: prompts:/support@production
```

MLflow resolves the manifest's assets and source into an immutable snapshot with a server-assigned,
per-agent monotonic number. The publisher version is optional metadata; multiple snapshots can share
it as assets pointed to aliases change during the life of the agent.

The **Agents** page keeps the existing navigation and workflows from experiments, including traces,
evaluations, and usage, with these changes:

- Add **Agent snapshots**, **Tools**, **Skills**, **Agent Plugins**, **MCP Servers**, **Models**,
  **Related agents**, and **Endpoints**. "Related agents" distinguishes declared subagents and agent
  dependencies from outbound agent calls observed in traces, including agents found through registry
  discovery.
- Skills and agent plugins link to registered versions in MLflow's
  [Skill Registry](../0008-mvp-skill-registry/0008-mvp-skill-registry.md). MCP servers and prompts
  similarly link to their MLflow registries; declared agent dependencies resolve through the
  configured agent registry backend. Each registered AI asset links back to the agents and snapshots
  that reference it.
- The **Models** page links to registered models and lists external models by provider and model
  name (e.g. `GPT 6.1 Sol`).
- The **Tools** page shows declared tools and tools observed in traces, with links to registered MCP
  servers when known.

The agent overview retains the GenAI experiment overview and adds first-class **Snapshot** and
**Version** filters. The main **Agents** page also supports filtering by fleet.

Users can compare snapshots, see their evaluation results and production traces, and identify which
prompt, tool, skill, model, or subagent changed. Harness-based agents can also identify the harness
and its declared configuration source. One logical agent can span development, staging, and
production.

Endpoint names such as `staging` and `production` identify access routes rather than snapshots. For
a specified publisher version, the computed `latest` reference selects the snapshot with the highest
per-agent snapshot number for that version. Without a version, it selects the highest-numbered
snapshot for the agent. Neither selection means that snapshot is deployed to production.

Users and other agents should be able to find agents to reuse and inspect their snapshots and
evaluation evidence.

Traces, evaluation runs, judges, and datasets can be associated with an agent instead of an
experiment. Each belongs to exactly one container. Relevant SDK and REST APIs accept the agent
association; autologging, trace ingestion, and trace search support it for traces.

Each agent has a globally unique, stable ID and a mutable name unique within its workspace. An
`AgentRef` pairs the ID with an optional snapshot ID for APIs that select an agent or an exact
registered composition. A publisher version may identify multiple snapshots. Agent-level APIs use
the ID; names support discovery and display.

Illustrative API changes, with final names TBD:

```python
from mlflow import MlflowClient
from mlflow.entities import AgentRef

client = MlflowClient()
agent_id = client.create_agent(name="support-agent")
mlflow.set_experiment(experiment_id="123")  # Existing workflow
mlflow.clear_active_destination()  # Switch from experiment to agent
agent = mlflow.set_agent(agent_id=agent_id)  # Uses mlflow-agent.yaml when present
system_prompt = agent.assets.prompts["system"]  # Resolved PromptVersion
# Explicit fields can also declare a snapshot:
candidate = mlflow.set_agent(
    agent_id=agent_id,
    version="2",
    assets={"prompts": {"system": "prompts:/support@production"}},
)
snapshot_id = candidate.snapshot_id  # None when only the agent container is selected

mlflow.search_traces(experiment_ids=["123"])
mlflow.search_traces(agents=[AgentRef(agent_id=agent_id, snapshot_id=snapshot_id)])

mlflow.genai.create_dataset(name="support-eval", agent_id=agent_id)
client.rename_agent(agent_id, "support-assistant")
# create_dataset(..., experiment_id="123") remains supported.
```

Explicit experiment and agent arguments are mutually exclusive. Setting an active agent while an
experiment is active, or an experiment while an agent is active, is an error.

Deployments can select an agent without calling `set_agent` by setting `MLFLOW_AGENT_ID` and,
optionally, `MLFLOW_AGENT_SNAPSHOT_ID`. An explicit `set_agent` or `set_experiment` call takes
precedence; without one, agent and experiment environment selectors cannot both be set.

With no manifest or other snapshot inputs, `set_agent(agent_id=...)` selects the agent container;
its traces remain at the agent level. A manifest or explicit snapshot input finds or creates a
snapshot and returns its ID.

`assets` groups named registry URIs by type; the handle returned by `set_agent` resolves them on
access, reusing registry loading and caching behavior.

`set_agent` can load `mlflow-agent.yaml` by default, or use a manifest path supplied directly or
through `MLFLOW_AGENT_MANIFEST`. Explicit arguments replace the corresponding manifest fields. The
`agent_id` argument is optional when the ID is supplied by the manifest or `MLFLOW_AGENT_ID`; users
can call `mlflow.set_agent()` without repeating it in code.

Setting `MLFLOW_AGENT_MANIFEST=autoload` opts into loading `mlflow-agent.yaml` from the process's
current working directory without calling `set_agent`; parent directories are not searched. Setting
the variable to a file path loads that manifest instead. With the variable unset, merely having a
manifest file does not enable automatic loading. Automatic loading selects the agent and finds or
creates a snapshot using the same rules as `set_agent`. An explicit `set_agent` call overrides
corresponding manifest fields; an explicit `set_experiment` call takes precedence over automatic
agent selection. A missing or invalid manifest, unresolved variable, or permission error surfaces as
an error rather than silently falling back to another destination.

The manifest may read its agent ID, publisher version, or Git revision from a named environment
variable, such as `version: {fromEnv: APP_VERSION}`. MLflow resolves the value when creating the
snapshot and stores it; a referenced variable that is unset is an error. A release pipeline can
supply these values directly or set the variables referenced by the manifest.

MLflow infers the Git commit as the source revision from a clean checkout when none is declared.
With uncommitted code, the commit alone cannot distinguish one local candidate from another. An
author-supplied source `revision` becomes part of snapshot identity, so changing it gives each
candidate its own trace and evaluation evidence for comparisons without committing the changes. A
dirty checkout requires a source revision to create a snapshot, supplied in the manifest, through
`set_agent(sources=...)`, or through an environment variable. An explicit override allows proceeding
despite the dirty Git state, with traces kept at the agent level instead of assigned to a snapshot.
With no Git checkout, a manifest-only snapshot is allowed without code provenance.

The registry atomically finds or creates a snapshot for the resolved declaration. A new snapshot
gets the next number under its agent. A prior matching snapshot is reused, including after an asset
alias rollback (e.g. a prompt alias is rolled back to a previous version). A newly resolved declared
asset from the alias moving can create a snapshot during execution; each trace retains the snapshots
and assets it actually used. A trace spanning different snapshots shows that mixture rather than
claiming one snapshot. Arbitrary assets observed in traces do not trigger registration.

#### Onboarding wizard

Similar to MLflow's agent instrumentation wizard, an onboarding wizard guides users through
identifying the AI assets their agent uses, reviewing them, and registering or reusing those assets
in MLflow. It then generates `mlflow-agent.yaml` with the agent ID and named asset references, ready
for `set_agent` or a harness integration to resolve into a snapshot. Users can instead set
`MLFLOW_AGENT_MANIFEST=autoload` and run their instrumented application without calling `set_agent`.

#### Harness integrations

An MLflow tracing plugin for a harness can read `mlflow-agent.yaml` to select the agent and supply
snapshot inputs alongside the manifest. For example, the Claude Code plugin detects the harness name
and version at session start. It resolves or creates a snapshot before exporting the session trace,
following the same source revision and permission rules as other snapshot creation. Without a
manifest or explicit snapshot inputs, it attaches the trace to the agent alone. Other harness
integrations can use the same approach when they can identify their harness version and skill paths.

After skill tracing is implemented, a plugin can scan the skill directories the harness actually
loads. For skills pulled through MLflow, it verifies the marker against the installed content digest
and adds the pinned skill URI to the snapshot's `registered_assets.skills`. Unmarked or modified
skills are not recorded as registered versions. These detected assets describe skills available to
the harness; the skill links on individual traces record which ones were activated. A changed set of
verified available skills produces a different snapshot.

OTLP request headers provide defaults; root-span attributes can override them or carry a manifest:

| Field    | Request header               | Root-span attribute        |
| -------- | ---------------------------- | -------------------------- |
| Agent    | `x-mlflow-agent-id`          | `mlflow.agent.id`          |
| Snapshot | `x-mlflow-agent-snapshot-id` | `mlflow.agent.snapshot_id` |
| Manifest | —                            | `mlflow.agent.manifest`    |

This allows one export to contain traces from different agents or snapshots. MLflow validates each
agent and snapshot pairing and agent permissions. An agent that does not use the MLflow client may
send a resolved manifest in the root span. With registration permission, MLflow can find or create a
snapshot from its resolved contents and omit the manifest from stored spans.

#### Agent fleets

An agent can belong to multiple fleets in its workspace. A fleet has a stable ID, a mutable name,
and a description of its purpose. The fleet owns its membership links to agents. Traces, sessions,
and evaluation runs do not record fleet membership.

In the UI, **Agent Fleets** opens a fleet overview with the GenAI quality and usage views, traces,
and sessions across its agents. An agent multiselect filters these views, and an **Agents** tab
links to each member's normal agent page. SDK and REST searches for agents, traces, sessions, and
evaluations accept a fleet ID filter.

Fleet views reflect current membership, including past activity; sessions show only matching
activity. Agents can appear in multiple fleets, so fleet totals are not additive. Membership does
not change agent identity or activity ownership.

Fleet ACLs govern reading and managing fleets. A fleet read succeeds with fleet read permission or
read permission for every member agent; otherwise it fails without partial results. Fleet read
permission grants read access to member agents' activity, not writes. Adding a member requires
manage permission on both the fleet and agent.

#### Agent discovery

The initial registry delivery includes agent search and retrieval through SDK and REST APIs. The
built-in MLflow MCP server exposes matching `search_agents` and `get_agent` tools. These operations
return agent identity, discoverable snapshots, and access endpoints with their advertised
capabilities and any A2A cards, plus a custom descriptor when available, subject to the configured
registry backend's permissions.

In addition, a companion skill should be published for coding agents in
[mlflow/skills](https://github.com/mlflow/skills) to find and inspect registered agents through
these interfaces.

#### Tracing and observability

An agent snapshot brings its registered composition and observed behavior into one view. Users can
filter traces and evaluations by agent and snapshot, compare quality, latency, and cost across
snapshots, and open the traces behind those results. From a trace, they can return to its snapshot
and registered prompts, models, tools, skills, and plugins. This connects production behavior to the
assets that changed between snapshots.

OTLP ingestion retains and indexes `service.instance.id` for SDK and REST trace search. The agent
**Traces** view offers an **Instance ID** filter; a trace matches when one of its spans comes from
that instance. This assumes one `service.namespace`/`service.name` pair sends traces to each agent
container.

Evaluation runs stay under the agent for comparison, including runs without a snapshot. Each run
records its snapshot ID, if any, or shows that its traces used multiple snapshots.

An agent snapshot's registered assets and external model references describe its declared
composition; they do not establish which resources a trace used. The agent view reuses
trace-to-asset links from the relevant registries when available, and separately shows tools and
agent calls observed in traces with their provenance. Traces without a known snapshot remain visible
at the agent level; missing asset-level observations do not imply non-use.

#### Registry backend

The MLflow tracking store provides the default backend for agent identity and container metadata in
OSS MLflow. A deployment can instead use another authoritative backend (e.g. Unity AI Gateway) to
resolve agent records for discovery and enforce agent permissions. MLflow still owns snapshots and
associates them, traces, and evaluations with the resolved agent; the external agent record need not
be copied into the tracking store's agent registry tables. The configured backend supplies globally
unique IDs and enforces workspace-unique names. The backend interface and UI behavior for
alternative backends remain follow-up design work.

#### Backwards compatibility and migration

Provide **Convert to agent** in the experiment UI and an equivalent API. Conversion creates or
selects a record through the configured registry backend, preserves existing traces, runs,
evaluations, and resource references, and presents them under the agent. Permissions follow the
configured backend's rules.

The GenAI experiment UI currently shows Logged Models as **Agent versions**, but these are not
registry snapshots. If an experiment has any Logged Models, the UI warns and the API rejects
conversion before changing the experiment.

Existing experiments continue to work. After conversion, existing SDK calls and OTel exporters using
the old experiment ID must keep working through a compatibility mapping to the agent. Clients can
migrate incrementally.

### Data schema

Adapt the `Agent` and mutable access endpoint concepts from the
[original draft](https://github.com/mlflow/rfcs/pull/39), replacing `AgentVersion` with immutable
agent snapshots. Logged Models continue to serve experiment-based workflows, where the GenAI UI
currently labels them **Agent versions**.

The following Python types illustrate the main read responses and relationships:

```python
from __future__ import annotations

from typing import Literal, TypeAlias, TypedDict

class ExternalModelRef(TypedDict):
    provider: str
    model: str
    version: str | None


class HarnessInfo(TypedDict):
    name: str  # e.g., "opencode"
    version: str | None  # e.g., "0.5.3"; None if unknown


class GitSource(TypedDict):
    type: Literal["git"]
    repository: str | None
    revision: str | None  # Git commit or author-supplied marker for uncommitted code
    subpath: str | None


class ImageSource(TypedDict):
    type: Literal["oci_image"]
    image: str
    digest: str


SourceRef: TypeAlias = GitSource | ImageSource


class AgentFleetMember(TypedDict):
    agent_id: str


class AgentFleet(TypedDict):
    workspace: str | None
    fleet_id: str  # Stable, server-assigned
    name: str  # Mutable
    description: str
    members: list[AgentFleetMember]


class Agent(TypedDict):
    workspace: str | None
    agent_id: str  # Globally unique and stable; assigned by the configured registry backend
    name: str  # Mutable, unique within the workspace
    description: str
    owner: str
    custom_descriptor: dict | None
    tags: dict[str, str]
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None


class AgentSnapshot(TypedDict):
    workspace: str | None
    agent_id: str
    snapshot_id: str  # Server-assigned, monotonically increasing within the agent
    version: str | None  # Optional publisher-supplied metadata
    sources: list[SourceRef]
    harness: HarnessInfo | None
    registered_assets: dict[str, dict[str, str]]  # Named, pinned registry URIs
    external_models: list[ExternalModelRef]
    tags: dict[str, str]
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None


class AgentAccessEndpoint(TypedDict):
    workspace: str | None
    agent_id: str
    name: str  # Unique within the agent; e.g., production or staging
    url: str
    protocol: Literal["a2a", "mcp", "custom"]
    custom_protocol: dict | None  # Required for custom: protocol name and interface definition
    transport_type: str  # Protocol-specific; MCP uses streamable-http or sse
    version: str | None  # Advertised publisher version, if known; not a snapshot selector
    capabilities: list[dict]  # Advertised for this access route
    a2a_card: dict | None  # Last-synced card for this access route
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None
```

Agent-owned records identify their parent by agent ID; snapshot IDs are unique within that agent.
Public APIs use the current workspace when no workspace is supplied.

Snapshot composition and source are immutable. Agent descriptors, endpoint capabilities and cards,
and tags can change. Deleted snapshots are absent from API responses; an internal tombstone reserves
each deleted snapshot ID.

Audit fields follow the MCP and Skill registries. Agent audit fields can come from the configured
backend; MLflow records snapshot audit fields. `owner` identifies who is responsible for an agent,
separately from who created or last updated its record.

An endpoint's A2A card or an agent's custom descriptor preserves the publisher's full document for
inspection, including fields MLflow does not model. This follows
[AWS Agent Registry's support for A2A and custom agent descriptors](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-supported-record-types.html).
MLflow keeps commonly queried asset links and access endpoints in structured fields. Each endpoint
has its own advertised capabilities: A2A endpoints derive them from the card, while non-A2A
endpoints can supply them directly using the same AgentSkill shape.

Composition follows these rules:

- The manifest declares named asset references, sources, harness, external models, and an optional
  publisher version. MLflow stores their resolved, canonical values in the snapshot.
- The manifest may reference asset aliases. `registered_assets` stores their resolved, pinned URIs,
  such as `prompts:/name/version` or `skills:/name/version`, under their declared names. The URI
  remains in the snapshot even if the referenced asset later becomes unavailable.
- `external_models` identifies providers and models without requiring Model Registry entries.
- `registered_assets` and `external_models` are optional when registering a snapshot.
- Declared and observed tools are distinguished by origin and may link to a registered MCP server
  using a version-pinned URI.
- `harness` is optional. Sources form a list; Git sources can point to a subpath. Sources initially
  support Git revisions and image digests. The manifest requires `type` for an image and defaults to
  `git` only when `repository` is present; it is not inferred from a URL suffix. Resolved sources
  always include `type`.
- A Git source revision is the commit for clean, committed code or an author-supplied marker for
  uncommitted code. An author-supplied marker distinguishes local states without making them
  retrievable.

Snapshots with the same agent, publisher version, resolved named assets, sources, harness, and
external models are reused. Otherwise, MLflow assigns the next snapshot ID for that agent. Deleted
snapshots are excluded from reuse. Undeclared code or dependency changes do not create a snapshot.
For a specified publisher version, `latest` selects the highest-numbered existing snapshot with that
version. Without a version, it selects the highest-numbered existing snapshot for the agent. It is
unresolved when no snapshot matches.

Endpoints follow the [MCP access endpoint pattern](../0004-mcp-registry/0004-mcp-registry.md) for
protocol and transport details. For `protocol="custom"`, `custom_protocol` describes the protocol
name, interface, and integration requirements as JSON. This describes an endpoint, separately from
the agent's `custom_descriptor`; the exact JSON structure remains TBD.

An endpoint belongs to an agent and identifies an access route. Its optional publisher version is
advertised to users and lets the UI show traces associated with the latest snapshot with that
version in the agent container.

#### Tools and relationships from traces

When declared tool metadata is unavailable, collect tool schemas exposed to the LLM in instrumented
calls, such as `ChatOpenAI` calls within LangGraph. Show their source trace and distinguish tools
offered to the model from tools actually called. Observed tools may be incomplete or
request-specific; they do not silently change a registered snapshot's composition. Trace-observed
tools retain links to the spans that supplied the evidence.

A2A autologging can later identify calls to another agent, resolve the endpoint/card to a registered
agent, and link the calling span to that entity and any available downstream trace. The **Related
agents** page lists agents called by the current agent. Selecting one filters the current agent's
traces to those containing a call to the selected agent. Within a trace, users can filter to the
corresponding outbound call spans.

### Syncing

Server-side sync jobs can run periodically or on demand from registration, the SDK, or a UI
**Refresh metadata** action. Show the last successful sync and any failure so users can judge
whether the record is current.

#### Plugin-based sync providers

A shared sync framework should support agents and eventually other AI asset registries. Agent
providers discover definitions and changes from deployment platforms or source locations, returning
metadata, source provenance, and composition for registration.

TODO: Define the shared provider interface, scheduling, change detection, authentication (including
OAuth), and handling of conflicts with user-managed fields. Asset-specific providers handle parsing
and snapshot creation. Server-side fetching should reject internal IP addresses by default,
including resolved and redirected targets, with an explicit administrator-controlled policy for
private deployments.

##### Built-in A2A scraper

A2A Agent Cards are proposed because
[A2A is an open standard](https://a2a-protocol.org/latest/specification/) for describing an agent's
advertised capabilities and connection details. The built-in provider fetches the card and retains
the most recently synced complete copy on the access endpoint for inspection.

The agent endpoint remains the source for the live card; MLflow's copy is labeled with its source
and last successful sync time. The A2A specification allows discovery through
[registries and catalogs](https://a2a-protocol.org/latest/specification/#82-discovery-mechanisms)
and [recommends HTTP caching](https://a2a-protocol.org/latest/specification/#86-caching).
[AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-supported-record-types.html)
and
[Google](https://docs.cloud.google.com/agent-registry/reference/rpc/google.cloud.agentregistry.v1)
also expose stored card payloads. The cached copy preserves the metadata MLflow indexed for
discovery and remains available when the endpoint cannot be reached.

Endpoint capabilities indexed from the card follow the
[A2A AgentSkill schema](https://a2a-protocol.org/latest/specification/#445-agentskill), populated
from the card's `skills` list. These describe advertised capabilities, such as "handle customer
refunds," rather than installed `SKILL.md` packages or internal tools such as `lookup_order`.

Create a snapshot when the card declares a previously unseen publisher version. A card refresh under
the same version updates the endpoint's cached card and advertised capabilities without changing
snapshot composition.

##### Built-in Git scraper

Detect new releases in a configured repository and register snapshots from their agent manifests.
TODO: Define which release or tag events trigger sync.

## Phasing

The work can be organized into three phases without committing to release boundaries:

1. **Agent container:** Establish agent records and access checks through the configured registry
   backend. Make agents an alternative to experiments across GenAI APIs, autologging, trace
   ingestion, search, and the UI. Add optional agent fleets with fleet views and search filters.
   Support conversion of eligible experiments and compatibility for existing clients.
2. **Snapshots, discovery, and syncing:** Add agent manifests, snapshots, linked assets, harness and
   configuration metadata, endpoints, and governance. Add the onboarding wizard, registration from
   manifests, and snapshot-level trace and evaluation comparison. Expose discovery through SDK,
   REST, and the built-in MCP server, with a companion skill in
   [mlflow/skills](https://github.com/mlflow/skills). Add A2A and Git sync, including the
   last-synced card and refresh, through the shared sync framework.
3. **Observed relationships:** Collect tool and agent relationships from instrumentation, including
   A2A calls, and show them alongside declared composition with trace provenance. Reuse
   trace-to-asset links from the corresponding asset registries.

## Drawbacks

- Moving GenAI ownership from experiments to agents affects many APIs, permissions, and UI flows;
  migration is a substantial part of the implementation.
- Imported metadata and trace observations can be incomplete or stale. Their provenance must remain
  visible when users review an agent.

## Open questions

- Should datasets eventually be decoupled from agents and experiments for reuse across containers?
- Should judges be reusable across a workspace, with optional agent association?
