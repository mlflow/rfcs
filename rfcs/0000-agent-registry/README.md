---
start_date: 2026-10-08
mlflow_issue: # TBD
rfc_pr:
---

# Agent Registry

## Summary

Make agents the primary GenAI container in MLflow. Each agent has a persistent identity and versions
that connect source code, AI asset dependencies, and runtime endpoints with traces and evaluations.
Teams can compare versions, promote them based on evidence, and investigate production issues. Other
agents can discover approved agents through the registry. Eligible experiments can be converted to
agents, while experiments remain supported for existing workflows and model training.

OSS MLflow provides the default registry backend. Managed deployments can instead resolve agent
records from an authoritative catalog or gateway, such as Unity AI Gateway. This draft seeks
agreement on product direction; storage and API details remain subject to follow-up design.

## Motivation

An agent should have one persistent identity in MLflow from development through production. Its
registry page would bring versions, source code, AI asset dependencies, evaluations, traces, and
endpoints together. Each version would show its prompts, models, tools, skills, and plugins
alongside evaluation results and production quality, latency, and cost, with links to the underlying
traces. Teams could compare candidates, promote a version through an alias, and investigate
production regressions using the same record.

MLflow's built-in AI Assistant or an external AI agent using MLflow's APIs could use this context as
well. If latency rises after an upgrade, it could compare the versions' source and dependencies with
slow calls in traces, then cite the evidence supporting its diagnosis.

Today, MLflow captures traces and evaluations but organizes GenAI work primarily under experiments,
even for production applications. "Experiment" suggests a temporary development activity. An
experiment's agent versions are currently logged models which provide version tracking, but there is
no persistent agent registry identity connecting discovery, governance, and registered dependencies
to that evidence. The proposal keeps the development loop—tracing, evaluating, and revising prompts
or tools—under the agent identity.

### Why MLflow

MLflow holds the behavioral evidence used to decide whether an agent version is ready and to explain
how it behaves after deployment. Linking that evidence to an authoritative agent record puts version
and dependency context where teams and assistants inspect it. OSS MLflow supplies the default
registry. A managed deployment can keep an existing catalog or gateway (e.g. Unity AI Gateway)
authoritative while MLflow resolves its agent records.

### Out of scope

- Hosting agents, proxying invocations, or managing runtime health and scaling.

## User journeys

Tracing and evaluation still work as they do today. A registered version can connect those results
to recorded source and declared AI assets; trace links show which assets a request actually used
when that information is available.

### Start tracing an agent

1. Create an agent in the UI. It gets a persistent identity before any version or endpoint is
   registered.
2. Select it with `mlflow.set_agent(...)` and run the instrumented application.
3. Open the agent's **Traces** page. When available, versioned traces also link to recorded source
   (such as a Git commit) and declared assets.

### Test an agent update

1. Select version 1 and run `mlflow.genai.evaluate`. Its run appears under the agent with version 1
   recorded.
2. Edit the agent code, select the agent without a version, and evaluate the draft. The draft run
   stays under the same agent; it does not claim to be version 2.
3. Compare the runs on the agent's **Evaluation runs** page. Open their traces to see which prompts
   and other assets were actually used, when linked, alongside version 1's source and dependencies.
4. Register version 2 with the updated source and dependencies once the draft is ready.
5. Optionally, associate the draft evaluation run with version 2 so it appears as evidence for that
   version, labeled as an evaluation performed before registration.

For a production prompt A/B test, register the candidate as a new agent version with its prompt
dependency pinned. The application splits traffic between the baseline and candidate versions;
MLflow groups evaluations and traces by agent version so users can compare quality, latency, and
cost, with links to the prompt version actually used when available.

### Investigate a production cost increase

1. Notice a cost spike on the agent's **Usage** page and ask MLflow's AI Assistant or a coding agent
   to investigate.
2. It notices a recent agent version bump and compares the versions' recorded source and declared
   assets. It checks affected traces to see which assets were used and which calls drove costs.
3. It suggests a likely cause, such as a newly added skill increasing token use, and links to
   supporting traces so the owner can verify it.

### Find an agent to reuse

1. A user searches the UI for a capability, or a running agent calls `search_agents` through the
   MLflow MCP server, SDK, or REST API when it needs another agent.
2. MLflow returns accessible versions with advertised capabilities and access endpoints. The user
   can inspect ownership, source, assets, and evaluation evidence; the running agent selects a match
   from the returned metadata.
3. The user integrates the chosen endpoint, or the running agent retrieves details with `get_agent`
   and invokes the endpoint during the request.

### Separate development and production activity

1. An administrator defines `dev` and `prod` environments for the workspace. Developers can write
   dev traces and evaluation runs but only read prod ones; the production deployment can write prod.
2. Developers and deployments supply an environment when logging traces or evaluations. Activity
   without one remains unassigned under the agent's base permissions.
3. On the agent overview, users filter by version and environment to compare activity without
   splitting the agent's identity or version history.

## Detailed design

### Overview of competitors

These products establish useful requirements and opportunities for MLflow:

| Platform   | Relevant capability                                                                                                                                                                                                                                                                                                                                                       | Requirement for MLflow                                                                             | MLflow opportunity                                                                                                              |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| AWS        | [Agent Registry](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry.html) supports discovery and approval, while [AgentCore Evaluations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations.html) scores traces.                                                                                                               | Discover approved agents and sync external descriptors.                                            | Bring agent versions, dependencies, traces, and evaluations into the same registry review workflow.                             |
| GCP        | [Agent Registry](https://docs.cloud.google.com/agent-registry/register-agents) indexes A2A Agent Cards for discovery; [bindings](https://docs.cloud.google.com/agent-registry/manage-bindings) connect agents to resources; [agent details](https://docs.cloud.google.com/agent-registry/manage-agents) show evaluation and observability for supported managed runtimes. | Import A2A capabilities and represent agent dependencies explicitly.                               | Provide a consistent version and evidence view for managed and externally hosted agents.                                        |
| Databricks | [Agent Services](https://docs.databricks.com/aws/en/ai-gateway/agent-services) register external agents as Unity Catalog assets with connections and permissions.                                                                                                                                                                                                         | Register external agents with ownership, permissions, and endpoints; no MLflow packaging required. | Add version, dependency, trace, and evaluation lineage while Unity Catalog can remain authoritative in the Databricks offering. |
| Microsoft  | [Agent Registry](https://learn.microsoft.com/en-us/microsoft-365/admin/manage/agent-registry) provides organization-wide inventory; [Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/development-lifecycle) supports versioning, tracing, and evaluation.                                                                                        | Support immutable versions and version comparisons alongside ownership and discovery controls.     | Link each version to registered assets and trace/evaluation evidence across runtimes in OSS MLflow.                             |

### Agents as the new GenAI container

#### What that means

The GenAI landing page becomes **Agents**. Users can create an agent in the UI or SDK, or convert an
eligible experiment. Agents can be registered before deployment, so an endpoint is optional.

The **Agents** page keeps the existing navigation and workflows from experiments, including traces,
evaluations, and usage, with these changes:

- Add **Agent versions**, **Tools**, **Skills**, **Agent Plugins**, **MCP Servers**, **Models**,
  **Related agents**, and **Endpoints**. "Related agents" distinguishes declared subagents and agent
  dependencies from outbound agent calls observed in traces, including agents found through registry
  discovery.
- Skills and agent plugins link to registered versions in MLflow's
  [Skill Registry](../0008-mvp-skill-registry/0008-mvp-skill-registry.md). MCP servers and prompts
  similarly link to their MLflow registries; declared agent dependencies resolve through the
  configured agent registry backend.
- The **Models** page links to registered models and lists external models by provider and model
  name (e.g. `GPT 6.1 Sol`).
- The **Tools** page shows declared tools and tools observed in traces, with links to registered MCP
  servers when known.

The agent overview retains the GenAI experiment overview and adds first-class **Version** and
**Environment** filters. The environment filter includes unassigned activity.

An agent version snapshots its composition and source. Users can compare versions, see their
evaluation results and production traces, and identify which prompt, tool, skill, model, or subagent
changed. Harness-based agents can also identify the harness and the configuration source that
defines them. One logical agent can span development, staging, and production, with version aliases
following the registered model and prompt pattern.

The proposed approach supports both explicit aliases such as `staging` and `production` and a
computed `latest` reference. Aliases allow deliberate promotion and rollback while newer versions
remain available for evaluation. `latest` can be selected as an alias, but advances when a newer
version becomes active; it does not mean that version is deployed to production.

Users and other agents should be able to find agents to reuse and see which versions are approved
for use.

Traces, evaluation runs, judges, and datasets can be associated with an agent instead of an
experiment. Each belongs to exactly one container. Relevant SDK and REST APIs accept the agent
association; autologging, trace ingestion, and trace search support it for traces.

An `AgentRef` pairs an agent ID with an optional `version` or `alias`, but not both. A version must
match an `AgentVersion` under that agent; an alias resolves to one. Search APIs accept lists of
references. Agent-level APIs such as dataset creation use a reference without either selector.

Illustrative API changes, with final names TBD:

```python
from mlflow.entities import AgentRef

mlflow.set_experiment(experiment_id="123")  # Existing workflow
mlflow.set_agent(agent=AgentRef(id="agent-456"))  # Unversioned development
# Alternatively, select an alias or a specific registered version:
mlflow.set_agent(agent=AgentRef(id="agent-456", alias="production"))
mlflow.set_agent(agent=AgentRef(id="agent-456", version="2"), environment="prod")

mlflow.search_traces(experiment_ids=["123"])
mlflow.search_traces(agents=[AgentRef(id="agent-456", version="2")])

mlflow.genai.create_dataset(name="support-eval", agent=AgentRef(id="agent-456"))
# create_dataset(..., experiment_id="123") remains supported.
```

Explicit experiment and agent arguments are mutually exclusive. Selecting an active agent replaces
the active experiment destination, and vice versa. With no selector, traces remain at the agent
level unless instrumentation can identify a registered version from the running agent. For an alias,
record the alias and its resolved version on each trace so later alias changes do not rewrite trace
history. The optional `environment` argument applies to traces and evaluation runs logged under the
active agent, not to the agent or its version.

A release pipeline can register a version with its source commit and/or image digest and declared
assets, then pass the resulting version ID to the deployment. This is a recommended practice, not a
requirement; manual registration and sync also work. We should provide example GitHub Actions and
GitLab CI/CD workflows for this pattern.

Deployments can select an agent without calling `set_agent` by setting `MLFLOW_AGENT_ID` and,
optionally, `MLFLOW_AGENT_VERSION` or `MLFLOW_AGENT_ALIAS`. The version and alias variables are
mutually exclusive. An explicit `set_agent` or `set_experiment` call takes precedence; without one,
agent and experiment environment selectors cannot both be set.

OTel exporters use `x-mlflow-agent-id` in place of `x-mlflow-experiment-id`, with an optional
`x-mlflow-agent-version` or `x-mlflow-agent-alias`. The two selectors are mutually exclusive and
must resolve under the supplied agent ID. An optional `x-mlflow-agent-environment` assigns all
traces in the export request to a workspace environment. It requires an agent ID and write access to
that agent/environment; without it, the traces remain unassigned. MLflow instrumentation can
populate the header from `MLFLOW_AGENT_ENVIRONMENT`. Supplying both experiment and agent headers is
rejected.

#### Agent discovery

The initial registry delivery includes agent search and retrieval through SDK and REST APIs. The
built-in MLflow MCP server exposes matching `search_agents` and `get_agent` tools. These operations
return agent identity, discoverable versions, advertised capabilities, an A2A card or custom
descriptor when available, and access endpoints, subject to the configured registry backend's
permissions.

In addition, a companion skill should be published for coding agents in
[mlflow/skills](https://github.com/mlflow/skills) to find and inspect registered agents through
these interfaces.

#### Tracing and observability

An agent version brings its registered composition and observed behavior into one view. Users can
filter traces and evaluations by agent and version, compare quality, latency, and cost across
versions, and open the traces behind those results. From a trace, they can return to the agent
version and its registered prompts, models, tools, skills, and plugins. This connects production
behavior to the assets that changed between versions.

Evaluation runs stay under the agent for comparison, including unversioned drafts. Each run records
the selected version, if any. Users may later link a draft run to a registered version as
pre-registration evidence without changing its original version or trace links. This user-provided
link does not verify provenance; show any captured source or asset conflicts.

An agent version's registered assets and external model references describe its declared
composition; they do not establish which resources a trace used. The agent view reuses
trace-to-asset links from the relevant registries when available, and separately shows tools and
agent calls observed in traces with their provenance. Traces without a known version remain visible
at the agent level; missing asset-level observations do not imply non-use.

#### Environments and permissions

Environments are defined at the workspace level so names such as `dev` and `prod` are consistent
across agents. Each trace and evaluation run has at most one environment, set at creation or left
unassigned. Environments do not apply to agents or agent versions: a version can be evaluated in
`dev` and later deployed in `prod` without creating another version. Traces and runs from each stage
keep their own environment.

Environment ACLs can apply across a workspace or to one agent. For example, a workspace rule can let
developers write `dev` traces and evaluation runs for every agent but only read `prod` ones;
agent-specific rules can override it. Unassigned traces and runs use agent permissions.

An authorized user can later assign or move a trace or evaluation run to another environment.

#### Registry backend

OSS MLflow provides the default agent registry backend. A deployment can integrate another
authoritative backend (e.g. Unity AI Gateway) for resolving agent and version identities, retrieving
records for discovery and display, and enforcing the applicable permissions. MLflow uses the
resolved identity to associate traces and evaluations with an agent and version; the authoritative
record need not be stored in MLflow's default registry tables. Agent IDs must be stable and
unambiguous within the deployment. The backend interface and UI behavior for alternative backends
remain follow-up design work.

#### Backwards compatibility and migration

Provide **Convert to agent** in the experiment UI and an equivalent API. Conversion creates or
selects a record through the configured registry backend, preserves existing traces, runs,
evaluations, and resource references, and presents them under the agent. Permissions follow the
configured backend's rules. If dev and prod experiments are converted into environments of one
agent, preserve their separate access rules so dev users do not gain write access to prod activity.

The GenAI experiment UI currently shows Logged Models as **Agent versions**, but these are not
registry `AgentVersion` records. If an experiment has any Logged Models, the UI warns and the API
rejects conversion before changing the experiment.

Existing experiments continue to work. After conversion, existing SDK calls and OTel exporters using
the old experiment ID must keep working through a compatibility mapping to the agent. Clients can
migrate incrementally.

### Data schema

Reuse the `Agent`, `AgentVersion`, and mutable access endpoint concepts from the
[original draft](https://github.com/mlflow/rfcs/pull/39). Agents use `AgentVersion` for version
tracking; Logged Models continue to serve experiment-based workflows, where the GenAI UI currently
labels them **Agent versions**.

The following Python types illustrate the main read responses and relationships:

```python
from __future__ import annotations

from typing import Literal, TypeAlias, TypedDict

from mlflow.entities.agent_plugin_version import AgentPluginVersion
from mlflow.entities.mcp_server_version import MCPServerVersion
from mlflow.entities.model_registry.model_version import ModelVersion
from mlflow.entities.model_registry.prompt_version import PromptVersion
from mlflow.entities.skill_version import SkillVersion


class ExternalModelRef(TypedDict):
    provider: str
    model: str
    version: str | None


class UnresolvedAsset(TypedDict):
    uri: str
    status: Literal["deleted", "unavailable"]


class HarnessInfo(TypedDict):
    name: str  # e.g., "opencode"
    version: str | None  # e.g., "0.5.3"; None if unknown


class ConfigurationSource(TypedDict):
    source_ref: str  # One of AgentVersion.source_refs
    subpath: str | None  # Path within the source; None means its root


class Agent(TypedDict):
    agent_id: str
    workspace: str | None
    name: str
    description: str
    owner: str
    version_scheme: Literal["semver", "monotonic", "freeform"]
    tags: dict[str, str]
    aliases: dict[str, str]  # Explicit alias -> version; latest is computed
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None


class AgentVersion(TypedDict):
    agent_id: str
    version: str
    source_refs: list[str]  # Git commit, image digest, configuration, or MLflow artifact
    harness: HarnessInfo | None
    configuration_source: ConfigurationSource | None
    registered_assets: list[str] | list[RegisteredAssetVersion | UnresolvedAsset]
    external_models: list[ExternalModelRef]
    capabilities: list[dict]  # A2A AgentSkill schema; imported or supplied directly
    a2a_card: dict | None  # Last-synced complete A2A Agent Card
    custom_descriptor: dict | None  # Publisher-supplied JSON for a non-A2A agent
    status: Literal["draft", "active", "deprecated", "deleted"]
    tags: dict[str, str]
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None


RegisteredAssetVersion: TypeAlias = (
    PromptVersion
    | ModelVersion
    | MCPServerVersion
    | SkillVersion
    | AgentPluginVersion
    | AgentVersion
)


class AgentAccessEndpoint(TypedDict):
    id: str
    agent_id: str
    url: str
    protocol: Literal["a2a", "mcp", "custom"]
    custom_protocol: dict | None  # Required for custom: protocol name and interface definition
    transport_type: str  # Protocol-specific; MCP uses streamable-http or sse
    agent_version: str | None  # Exactly one of agent_version / agent_alias
    agent_alias: str | None
    resolved_version: AgentVersion | None  # Read-only resolved target
    workspace: str | None
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None


class AgentTool(TypedDict):
    agent_id: str
    agent_version: str | None
    name: str
    schema: dict
    origin: Literal["declared", "scraped", "trace"]
    mcp_server: str | MCPServerVersion | None  # Version-pinned URI or resolved entity


class AgentSyncSource(TypedDict):
    agent_id: str
    provider: str  # a2a, git, or an installed provider
    location: str
    default_status: Literal["active", "draft"]  # Defaults to active for new versions
    # auth_ref: TBD
    schedule: str | None
```

Version composition and source snapshots are immutable. Capabilities and the stored card or
descriptor can be refreshed; version status, agent and version tags, and aliases can change. Status
follows the MCP and Skill registry lifecycle.

Audit fields follow the MCP and Skill registries and come from the configured registry backend when
available. `owner` identifies who is responsible for an agent, separately from who created or last
updated its record.

At most one of `a2a_card` and `custom_descriptor` may be set; neither is required. This follows
[AWS Agent Registry's support for A2A and custom agent descriptors](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-supported-record-types.html).
MLflow keeps commonly queried capabilities, asset links, and access endpoints in structured fields.
Agents without A2A can supply capabilities directly using the same AgentSkill shape. The card or
descriptor preserves the publisher's full document for inspection, including fields MLflow does not
model.

Composition follows these rules:

- `registered_assets` accepts version-pinned URIs or registry version objects when creating an agent
  version. MLflow resolves aliases and stores the corresponding URIs, such as
  `prompts:/name/version` or `skills:/name/version`. REST writes use URI strings; reads return typed
  version entities. If a referenced asset cannot be returned, reads include an `UnresolvedAsset`
  with its URI and `deleted` or `unavailable` status. When a returned asset is an `AgentVersion`,
  its own `registered_assets` remain version-pinned URI strings to avoid recursive expansion.
- `external_models` identifies providers and models without requiring Model Registry entries.
- `registered_assets` and `external_models` are optional when registering an agent version.
- Declared and observed tools are distinguished by origin and may link to a registered MCP server
  through `mcp_server`, using a version-pinned URI or `MCPServerVersion`.
- `harness` and `configuration_source` are optional. The configuration source selects an entry in
  `source_refs`, with `subpath` locating the configuration inside it. Framework details can remain
  in source metadata or a custom descriptor.

Each agent selects a version scheme when created:

- **SemVer:** Publisher-supplied versions, ordered by semantic-version precedence.
- **Monotonic:** MLflow-assigned sequential numbers, ordered numerically. This is the default when
  versions are not supplied.
- **Free-form:** Publisher-supplied strings, ordered by initial registration time.

This allows A2A agents to retain their version strings. The computed `latest` reference follows the
MCP and Skill registry status rule: select the latest active version under the agent's scheme,
falling back to the latest non-deleted version when none are active. If all versions are deleted or
no versions exist, there is no latest version.

Endpoints follow the [MCP access endpoint pattern](../0004-mcp-registry/0004-mcp-registry.md), using
the implementation's `MCPAccessEndpoint` field names with agent identity and protocol details. For
`protocol="custom"`, `custom_protocol` describes the protocol name, interface, and integration
requirements as JSON. This describes an endpoint, separately from the agent's `custom_descriptor`;
the exact JSON structure remains TBD.

Each endpoint targets exactly one version or alias. Its URL and target can change without rewriting
version history. Keep sync locations separate from access endpoints: a Git repository or card URL
supplies metadata, while an endpoint identifies where consumers connect.

#### Tools and relationships from traces

When declared tool metadata is unavailable, collect tool schemas exposed to the LLM in instrumented
calls, such as `ChatOpenAI` calls within LangGraph. Show their source trace and distinguish tools
offered to the model from tools actually called. Observed tools may be incomplete or
request-specific; they do not silently change a registered version's composition.

A2A autologging can later identify calls to another agent, resolve the endpoint/card to a registered
agent, and link the calling span to that entity and any available downstream trace. The **Related
agents** page lists agents called by the current agent. Selecting one filters the current agent's
traces to those containing a call to the selected agent. Within a trace, users can filter to the
corresponding outbound call spans.

### Syncing

Server-side sync jobs can run periodically or on demand from registration, the SDK, or a UI
**Refresh metadata** action. Show the last successful sync and any failure so users can judge
whether the record is current. Agent versions created by sync use the configuration's
`default_status`, which defaults to `active` and can be set to `draft` for review before activation.

#### Plugin-based sync providers

A shared sync framework should support agents and eventually other AI asset registries. Agent
providers discover definitions and changes from deployment platforms or source locations, returning
metadata, source provenance, and composition for registration.

TODO: Define the shared provider interface, scheduling, change detection, authentication (including
OAuth), and handling of conflicts with user-managed fields. Asset-specific providers handle parsing
and version creation. Server-side fetching should reject internal IP addresses by default, including
resolved and redirected targets, with an explicit administrator-controlled policy for private
deployments.

##### Built-in A2A scraper

A2A Agent Cards are proposed because
[A2A is an open standard](https://a2a-protocol.org/latest/specification/) for describing an agent's
advertised capabilities and connection details. The built-in provider fetches the card and retains
the most recently synced complete copy in `a2a_card` for each registered agent version for
inspection and comparison.

The agent endpoint remains the source for the live card; MLflow's copy is labeled with its source
and last successful sync time. The A2A specification allows discovery through
[registries and catalogs](https://a2a-protocol.org/latest/specification/#82-discovery-mechanisms)
and [recommends HTTP caching](https://a2a-protocol.org/latest/specification/#86-caching).
[AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-supported-record-types.html)
and
[Google](https://docs.cloud.google.com/agent-registry/reference/rpc/google.cloud.agentregistry.v1)
also expose stored card payloads. The cached copy preserves the metadata MLflow indexed for
discovery and remains available when the endpoint cannot be reached.

`AgentVersion.capabilities` follows the
[A2A AgentSkill schema](https://a2a-protocol.org/latest/specification/#445-agentskill), populated
from the card's `skills` list. These describe advertised capabilities, such as "handle customer
refunds," rather than installed `SKILL.md` packages or internal tools such as `lookup_order`.

Create a new MLflow agent version when the card declares a previously unregistered agent version. If
that version is already registered, refresh its stored card and advertised capabilities without
creating a new agent version.

##### Built-in Git scraper

Detect new releases in a configured repository and register agent versions with source provenance
and declared dependencies. TODO: Define how a repository identifies an agent and its composition,
and which release or tag events create versions.

## Phasing

The work can be organized into three phases without committing to release boundaries:

1. **Agent container:** Establish agent records and access checks through the configured registry
   backend. Make agents an alternative to experiments across GenAI APIs, autologging, trace
   ingestion, search, and the UI. Add workspace environments, trace and evaluation permissions, and
   environment filters. Support conversion of eligible experiments and compatibility for existing
   clients.
2. **Versions, discovery, and syncing:** Add versions, aliases, linked assets, harness and
   configuration metadata, endpoints, and governance. Add version-level trace and evaluation
   comparison. Expose discovery through SDK, REST, and the built-in MCP server, with a companion
   skill in [mlflow/skills](https://github.com/mlflow/skills). Add A2A and Git sync, including the
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
- Does `active` mean approved for discovery and use, or does approval need a separate workflow?
