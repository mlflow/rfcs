# RFC 0011: Agent Registry

| start_date   | 2026-08-24 |
| :----------- | :--------- |
| mlflow_issue | https://github.com/mlflow/mlflow/issues/25572 |
| rfc_pr       | https://github.com/mlflow/rfcs/pull/39 |

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
  - [Design positions](#design-positions)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [Open questions](#open-questions)

# Summary

Add an Agent Registry to MLflow: a governed, record-level registry
that catalogs each agent's identity, ownership, composition, source
provenance, and lifecycle status, and anchors the agent's traces and
evaluation results. It answers "what agents exist, who owns them,
what are they made of, and how well do they work?"

The Agent Registry is the third registry in a series, following the
[MCP Server Registry
(RFC-0004)](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md)
and the [Skill Registry
(RFC-0008)](https://github.com/mlflow/rfcs/blob/main/rfcs/0008-mvp-skill-registry/0008-mvp-skill-registry.md).
It completes a progression: MLflow can govern the tools an agent calls
(MCP servers), the expertise it carries (skills), the models it
invokes (Model Registry), and the prompts it sends them (Prompt
Registry), but the agent itself, the thing that acts, has no
registry entry. This RFC adds one.

In brief, the design takes these positions, each stated in full in
[Design positions](#design-positions): the registry is
record-level, not runtime-aware; an agent's versions are immutable
snapshots of its composition (a bill of materials of skills, agent
plugins, MCP servers, models, prompts, other agents it calls, and
the harness or framework that runs it) plus at least one
definitional anchor
(typed source pointers, which for a harness-based agent point at its
configuration); A2A Agent
Cards are fetched from the agent's endpoint, never stored;
endpoints are mutable, protocol-typed access endpoint records
rather than version fields; and for GenAI work an agent, not an
experiment, is the entity users create and trace against: it
carries the
traces-and-evaluations experience, with a default trace location
plus one per deployment that needs its own, and the version
recorded on every trace and evaluation run, while experiments
continue for model training and underneath.

**Relationship to other RFCs.** RFC-0004 establishes the access
endpoint pattern this RFC reuses (its canonical-payload pattern is
deliberately not applied to Agent Cards; see
[Design positions](#design-positions)). RFC-0008 defines
the skills and agent plugins that agent BOMs reference; note that an
*agent plugin* (a package of components installed into a harness) is
not an *agent* (an application that acts); this registry governs the
latter. [RFC-0009: Skill
Tracing](https://github.com/mlflow/rfcs/pull/37) annotates spans
within a trace with skill coordinates; this RFC associates whole
traces with an agent version. The two compose: an agent-linked trace
containing `SKILL` spans shows which parts of the agent's BOM were
exercised in a given run.
[RFC-0010: Extended Agent Plugins](https://github.com/mlflow/rfcs/pull/27)
extends plugin composition and is complementary.

# Basic example

The API shapes below are illustrative sketches; exact signatures
belong to the detailed design.

## Register an agent

```python
import mlflow
from mlflow.genai import GitSource

mlflow.genai.register_agent(
    organization="acme",
    name="billing-agent",
    description="Answers customer billing questions.",
    sources=[
        GitSource(
            url="https://github.com/acme/billing-agent.git",
            ref="8f4e2a1",
        ),
        OciSource("quay.io/acme/billing-agent@sha256:9f2c1e"),
    ],
    skills=["skills:/billing-policy/1", "skills:/refund-rules/2"],
    agent_plugins=["agent-plugins:/billing-workflow/1.2.0"],
    agents=["agents:/@acme/records-agent/2"],
    mcp_servers=["mcp-servers:/acme.internal/payments-db/2.0.0"],
    models=["models:/acme-billing-llm/3", "gpt-4o"],
    prompts=["prompts:/billing-system-prompt/4"],
    framework="langgraph",
    framework_version="0.3.1",
)
```

This creates the `Agent` (if new) and an `AgentVersion` with status
`draft`. `GitSource` and `OciSource` are typed source pointers as
in RFC-0008. A version may record multiple sources; here, the Git
repo the agent is built from and the container image it ships as.
Unlike skill versions, which carry exactly one source, agent
versions allow several (see [Open questions](#open-questions) for
the rationale).

## Register an agent from an A2A Agent Card

```python
mlflow.genai.register_agent(
    organization="acme",
    name="travel-agent",
    a2a_endpoint="https://agents.acme.internal/travel",
    skills=["skills:/itinerary-planning/4"],
)
```

Given an endpoint, the client SDK fetches the Agent Card from the
endpoint's well-known path, stores its description, its free-form
name (which seeds the mutable MLflow-managed `display_name`), and
its skills list in registry fields, and creates an `a2a` access
endpoint record for it. The card as a document is not persisted: the
endpoint is the card's system of record, and the UI renders the
card read-only by fetching it through that record at view time. The
registry `name` is always chosen by the registrant, since a card's
`name` is a display string, not an identity. Fetches happen in the
client, never in the registry server, consistent with RFC-0008; a
caller can also fetch the card itself and pass it via `a2a_card=`
to inspect the imported metadata first. A version registered this
way, with no source, is an
interface-only record (see the register journey).

## Register a harness-based agent

```python
mlflow.genai.register_agent(
    organization="acme",
    name="oncall-helper",
    description="On-call assistant run in OpenCode.",
    harness="opencode",
    harness_version="0.5.3",
    skills=["skills:/runbook-triage/2"],
    mcp_servers=["mcp-servers:/acme.internal/pagerduty/1.2.0"],
    models=["claude-sonnet-5"],
    sources=[GitSource(
        url="https://github.com/acme/oncall-config.git",
        ref="c41d9e0",
    )],
)
```

Or, from a checkout of that configuration, let MLflow propose the
registration:

```bash
uvx mlflow@latest agent register --organization acme --name oncall-helper
```

The command scans the tree, shows the skills, MCP servers, models,
and prompts it found and which registry entries they match,
registers the agent after confirmation, and writes a coordinates
file back into the tree (the scan path in the register journey).

There is no agent code of the user's own: the agent is the harness
plus its configuration, so the source pointer points at the
configuration and serves as the definitional anchor. Any of the
Skill Registry's source types works: a Git repo, an OCI image, a
zip archive, or direct MLflow artifact storage (`mlflow`), the last
only in deployments where MLflow serves artifacts. Configuration
files often embed secrets, so the `mlflow` type carries the same
caveat it does for skill content: what is uploaded is what is
stored.

## Trace and evaluate against the agent

```python
mlflow.genai.set_active_agent("agents:/@acme/billing-agent", version=3)

with mlflow.start_span(name="answer-question"):
    result = agent.run(question)

mlflow.genai.evaluate(
    data=eval_dataset,
    scorers=[correctness_scorer],
    agent_id="agents:/@acme/billing-agent",
    agent_version=3,
)
```

`set_active_agent` does two separable things: it points the trace
destination at the agent's one default experiment (the version
plays no part in that), and it records the agent and version as
metadata on everything emitted while it is active. The trace
journey shows the underlying pieces.

# Motivation

## The problem

MLflow can already govern most of the components an agent is built
from, and it can already trace what agents do. What is missing is the
agent itself as a governed entity, and that gap shows up in four
ways.

1. **Agents have no record.** An agent is built, deployed, and
   iterated on with no formal registration: no owner, no lifecycle
   state, no accountability chain. Models, MCP servers, and skills
   each have registry entries; the agent that composes them, the
   thing that acts autonomously on behalf of the organization, has
   none. When an agent misbehaves, "who is responsible for this?" is
   answered by asking around.

2. **Composition is untracked.** An agent is a composition of skills,
   MCP servers, models, and prompts, each versioned independently.
   No record captures which versions of which components a given
   build of the agent used. When behavior changes, "something
   changed and the agent broke; what was it?" requires
   reconstructing the composition from memory, commit history, and
   luck.

3. **Experiments do not map to agents.** MLflow traces and evaluation
   runs attach to experiments. Experiments fit training workflows,
   where a run is an attempt at producing a model. They fit the agent
   development loop badly: a developer looking at an agent wants its
   traces and eval results immediately, not by way of an experiment
   wrapper created as ceremony. There is no first-class way to say
   "show me this agent's traces, for this version."

4. **Cross-registry questions require manual inspection.** The Skill
   and MCP registries can say a component version is deprecated or
   compromised, but nothing records which agents carry it. "Skill X
   is compromised; which agents are affected?" is answered today by
   inspecting every deployment individually. The registries hold the
   component half of the answer; the consumer half is missing.

These problems compound in regulated environments, where the
accountability and blast-radius gaps are commonly reported as
blockers for putting agents into production.

## User journeys

These journeys define the MVP scope. They are written against the
SDK, CLI, and UI surfaces the registry will offer; exact API shapes
are illustrative.

### Register an agent

A developer (or a CI/CD pipeline) has built an agent and wants it on
the record.

1. Register the agent, supplying identity, provenance, and
   composition:
   ```python
   mlflow.genai.register_agent(
       organization="acme",
       name="billing-agent",
       description="Answers customer billing questions.",
       sources=[GitSource(
           url="https://github.com/acme/billing-agent.git",
           ref="8f4e2a1",
       )],
       skills=["skills:/billing-policy/1"],
       mcp_servers=["mcp-servers:/acme.internal/payments-db/2.0.0"],
       models=["models:/acme-billing-llm/3"],
   )
   ```
   Required: a name, a description, and at least one of a
   definitional anchor (one or more typed source pointers) or an
   endpoint. A registration with an anchor must also declare
   composition (the BOM). Optional: tags, an explicit version where
   the agent's version scheme takes one (the default `monotonic`
   scheme assigns versions), and, alongside an anchor, an endpoint.
   A registration with an endpoint and no anchor produces an
   interface-only record, whose composition may be partial or
   undeclared.
2. MLflow creates an `AgentVersion` record with initial status
   `draft`.
3. The agent appears in the registry listing for its workspace, with
   its BOM entries linked to the skill, agent plugin, MCP server,
   and model registry pages where matching entries exist.
4. **A2A path:** an agent that serves an Agent Card registers from
   its endpoint. The UI registration form offers two modes, "import
   from A2A card" and "manual"; the import mode pre-fills the
   description and skills list from the card (its free-form name
   seeds the mutable `display_name`; the registry `name` is
   supplied by the registrant) and creates an `a2a` access endpoint
   record for it. In the SDK and CLI, the client fetches the
   card at import; in the UI, the browser fetches it when the
   endpoint permits, or the user pastes it, since the server never
   fetches user-supplied URLs. The BOM is supplied alongside, since
   the card schema does not carry component version pins.
5. **CI path:** the same call runs from a pipeline, registering a new
   version on each release build with the source ref set to the
   build's commit.
6. **Harness path:** an agent that runs as a configuration of a
   packaged harness (Claude Code, OpenCode, Goose, and similar; see
   [RFC-0009](https://github.com/mlflow/rfcs/pull/37) for the
   harness/framework distinction) has no agent code of its own: the
   agent *is* the harness plus its configuration. It registers with
   a harness reference and a source pointer to the configuration
   that defines it:
   ```python
   mlflow.genai.register_agent(
       organization="acme",
       name="oncall-helper",
       description="On-call assistant run in OpenCode.",
       harness="opencode",
       harness_version="0.5.3",
       skills=["skills:/runbook-triage/2"],
       mcp_servers=["mcp-servers:/acme.internal/pagerduty/1.2.0"],
       models=["claude-sonnet-5"],
       sources=[GitSource(
           url="https://github.com/acme/oncall-config.git",
           ref="c41d9e0",
       )],
   )
   ```
   The configuration is what distinguishes this agent from every
   other installation of the same harness: enabled tools, overridden
   defaults, and behavioral settings live there and nowhere else. It
   is referenced, not stored. Any of the Skill Registry's source
   types serves (Git, OCI, zip, or direct MLflow artifact storage
   where the deployment serves artifacts), so the registry treats
   configuration the way it treats skill content, pointing at where
   it belongs rather than becoming its home.

   A harness's configuration surface is not always a single file
   (Claude Code, for example, spreads it across a settings file,
   instruction files, and subagent definitions), which every source
   type accommodates as a tree. The configuration source should hold
   only configuration the registry does not otherwise represent:
   content that BOM references already govern, such as installed
   skill directories or MCP server definitions, stays out, because
   an embedded copy is invisible to cross-registry queries and can
   drift from the declared references.
7. **Scan path:** from the agent's source tree, `mlflow agent
   register` (also `uvx mlflow@latest agent register`, alongside the
   existing `agent setup`) scans the tree and proposes a BOM. The
   scan is exact for MLflow's own footprints: the Skill Registry's
   resolution lock file and any `skills:/`, `prompts:/`, `models:/`,
   or `mcp-servers:/` references in code or configuration. It is
   best-effort for harness-native configuration, such as MCP server
   declarations in a harness's settings file, through per-harness
   adapters scoped to the well-known harness list; a generic mode
   finds only the exact layer. The registrant reviews the proposal,
   with matched registry entries and unmatched findings shown
   apart, and confirms or edits it. Unmatched skills and MCP servers
   can be registered on the spot through the Skill Registry's
   import adapters and the MCP Server Registry's create path, so
   the references resolve; the agent and its version are then
   registered with the source pointer set to the tree's Git remote
   and commit. Finally the command writes a coordinates file into
   the tree recording the agent's coordinates and its resolved BOM
   (the form described in Design positions), so the next version
   registration and any later scan are exact. The file format is
   shared with the resolution lock the Skill Registry work defers
   to a separate RFC and is specified there.

The harness path is the newest part of this design and the least
settled (see [Open questions](#open-questions)). For framework-built
and custom agents, the registered source is the natural complete
record and remains the expected anchor; for harness-based agents,
requiring source would force registrations that point at the
harness vendor's repository, which identifies nothing about the
specific agent. Configuration files frequently embed secrets and
environment-specific values; pointing at configuration rather than
storing it keeps that custody outside MLflow, and the `mlflow`
source type carries the same caveat it does for skills.

### Publish and maintain an agent's endpoint

A platform operator deploys a registered agent and needs the
registry to say where, and how, it can be reached, without
disturbing the immutable version history.

1. The agent is already registered (any path above) and a version
   has been promoted. The platform team deploys it.
2. The operator creates an access endpoint for the deployment,
   targeting a version or an alias and declaring the endpoint's
   protocol:
   ```python
   mlflow.genai.create_agent_access_endpoint(
       agent="agents:/@acme/billing-agent",
       target_alias="production",
       endpoint_url="https://agents.acme.internal/billing",
       protocol="a2a",
       platform_url="https://console.acme.internal/agents/billing-prod",
   )
   ```
   The endpoint accepted at registration time is sugar for creating
   this record; the A2A registration path creates an `a2a` endpoint
   record automatically. Two optional fields describe the deployment
   without recording its state: `platform_url` links to wherever
   the serving platform shows this deployment (a console page, a
   Kubernetes resource), and a free-text `description` holds
   connection notes.
3. The agent's detail page lists its access endpoints. Those whose
   protocol is self-describing (`a2a`, `mcp`) are actionable: they
   are the entry points for the endpoint-driven tracing and
   evaluation in the trace-and-eval journey below. They are also
   how a developer, or another agent, goes from a registry search
   to a live endpoint: the record gives the URL, and the protocol
   gives the rest. An `other` endpoint is a documented pointer, and
   its `description` is where the operator says how to call it.
4. The deployment moves to a new URL. The operator updates the
   endpoint record; no version record changes.
5. The deployment is retired. The operator deletes the record; the
   agent, its versions, and its history remain untouched.

The protocol field is where agent access endpoints depart from the
MCP Server Registry's, which are always MCP and vary only by
transport. The field
is limited to values that tell a caller something actionable: `a2a`
and `mcp` are self-describing (an Agent Card at the well-known
path; the MCP handshake), so URL plus protocol is enough to
connect. Labels like REST or gRPC name a transport without telling
anyone how to call the agent, so they are deliberately collapsed
into `other`, which records where an agent lives without claiming
MLflow can invoke it. As in RFC-0004, an endpoint that targets an
alias such as `production` follows the alias as it moves between
versions.

### Version an agent and compare bills of materials

A developer iterating on an agent registers each change as a new
version. Later, when behavior shifts, whoever is investigating, and
it is often not the person who made the change, compares two
versions to see exactly what differs.

1. The developer updates the agent: bumps a skill version, adds an
   MCP server, or swaps a model.
2. They register the updated composition, producing a new
   `AgentVersion`. Each version is an immutable BOM snapshot; there
   is no in-place edit of composition.
3. In the UI, they open the agent's detail page, select two versions
   on the Versions tab, and choose Compare.
4. The comparison shows a side-by-side BOM diff: for example,
   `billing-policy` skill `1` → `2`, model `llama-3.1-70b` →
   `llama-3.1-405b`, one MCP server added, or the configuration
   source moved from one ref to another.
5. Changed components link to their Skill Registry and MCP Server
   Registry entries, where their own version histories and changelogs
   live.

Immutable versions are what make the diff trustworthy: the
comparison reflects what was registered, not what a mutable record
has drifted into. The diff is a shared record of declared changes:
its value is that a teammate, reviewer, or security engineer can
see what differs in the declarations for two versions without
depending on the change author's memory or availability. Combined
with the evaluation comparison in the next journey, it gives an
investigation its starting facts: what changed, and did it matter?
Discovering changes that nobody declared belongs to the deferred
auto-discovery work (see Out of scope).

Whether a change is a new version or a new agent is the
registrant's call, the same judgment developers already make for
any software: is this a new release of the same application, or a
different application? The registry enforces no rule; the practical
consequence of the choice is that version comparison exists only
within one agent.

### Develop against agent-centric traces and evaluations

A developer evaluating agent quality wants traces and eval results
organized by agent and version, not by experiment.

1. Log traces against the agent instead of an experiment:
   ```python
   mlflow.genai.set_active_agent("agents:/@acme/billing-agent", version=3)

   with mlflow.start_span(name="answer-question"):
       result = agent.run(question)
   ```
   `set_active_agent` is convenience over two separable pieces.
   First, it sets the trace destination to the agent's one default
   experiment, equivalent to calling `mlflow.set_experiment` on the
   result of
   `mlflow.genai.get_default_experiment_id("agents:/@acme/billing-agent")`,
   where `get_default_experiment_id` is a public lookup that takes
   only the agent: the version is never part of the destination. (An
   `MlflowAgentTraceLocation` naming the agent works anywhere MLflow
   accepts a trace destination.) Second,
   it records the agent and version as trace-level metadata, the way
   session and user metadata are recorded today; this is what
   per-version filtering and comparison use. A deployment that
   needs its own trace location (below) names it instead:
   `set_active_agent("agents:/@acme/billing-agent", version=3,
   deployment="prod-eu")` resolves to that deployment's location
   and still records the agent and version so the traces stay
   labeled.
   Framework and harness autologgers respect the active destination
   and metadata, so instrumented applications need only state which
   agent they are. Agents that export traces through OpenTelemetry
   without the MLflow SDK pass the same agent identity and version
   in export headers, mirroring the existing experiment-ID header.
2. Run evaluations against the agent:
   ```python
   mlflow.genai.evaluate(data=eval_dataset,
                         scorers=[correctness_scorer],
                         agent_id="agents:/@acme/billing-agent",
                         agent_version=3)
   ```
   As with tracing, `agent_id` determines where the results land
   (the agent's default experiment, unless overridden) and
   `agent_version` is recorded on the evaluation run for filtering
   and comparison.
3. Open the agent's detail page. A Traces tab shows the agent's
   traces, filterable by version; an Evaluations tab shows eval runs;
   a summary card shows latest eval score and trace volume.
4. Compare versions: select v2 and v3, see score deltas and regressed
   cases, and drill from a regressed case into its trace to identify
   the cause, cross-referencing the BOM diff from the previous
   journey.

Backward compatibility is preserved by construction. Where MLflow
already accepts a typed destination or location (trace
destinations, `search_traces` locations), agent identity becomes a
new accepted value; where it
does not (`evaluate`), `agent_id` is new, optional surface. Existing
experiment-based workflows (including model training and
fine-tuning) continue unchanged. The change is additive, not a data
model rewrite.

The default trace location is a default, not a router. A running
agent logs traces to whatever location its own deployment
configuration names; the registry is not in the call path. When
nothing is named, traces land in the agent's default location,
which is the right behavior for the development loop and
registry-driven evaluations. Scale-out replicas of one deployment
share its configuration, so their traces aggregate without further
arrangement. A deployment whose traces must be kept apart from
other deployments of the same agent gets its own trace location:
two teams each running the agent for their own users, per-tenant
or per-customer deployments whose prompts and data must not cross,
production deployments whose traces carry stricter access than
non-production ones, or deployments split by region or
jurisdiction. Versions share a location: a version is a filter
recorded on every trace, not an audience. Locations
never make traces hard to find, because the registry keeps the
list: an agent's registered trace locations are its default plus
one per registered deployment, presented on the agent's page as
deployments of the agent and searched across as one. A deployment
with its own location must be registered with the agent; the
default location is fixed when the agent is created and never
re-pointed, since re-pointing it would be redundant with
registering a deployment, so locations are added rather than moved.
Automatically registering deployments and their trace locations at
deploy time is part of the registry synchronization work deferred
in Out of scope.

None of this needs an endpoint when the developer has the agent's
code: the agent runs locally or in CI, autologging captures traces
during execution, and evaluation scores outputs against a test
dataset. Agents whose code the user cannot run are the next
journey.

Where [RFC-0009](https://github.com/mlflow/rfcs/pull/37) annotates
spans inside a trace with the skill that produced them, this journey
attaches the whole trace to the agent that ran. Together they give
component-level attribution within agent-level organization: from an
agent's trace list, the `SKILL` spans inside a trace show which BOM
entries were actually exercised.

Calls to other agents follow the same idea. Every call to another
agent is annotated at the call site with the callee's identity, so
the callee's registry page can find traces it participated in.
Where the callee's own work lands follows from where it runs: an
in-process callee (an agent used as a tool inside the caller's
process) nests as spans in the caller's trace, like nested skills;
a remote callee (delegation over A2A to a separately running agent)
produces its own trace in its own destination, linked to the
caller's trace through propagated trace context, using the span
links MLflow already supports for OpenTelemetry.

### Vet an agent you did not build

A platform or governance team must decide whether an agent the
organization did not build, a vendor's or a partner's, may be used;
a consuming team wants the same assurance about another team's
deployed agent. Neither can run the agent's code, and neither has
any access to whoever built it beyond the ability to call its
endpoint.

1. The agent is registered as an interface-only record with an
   `a2a` endpoint (the A2A path of the register journey); its
   composition is undeclared.
2. The team runs an evaluation from the registry against that
   endpoint, using its own test datasets and scorers. MLflow invokes
   the endpoint with each test input, records the request and
   response as a trace in the team's own MLflow instance, and scores
   the outputs. Nothing on the agent's side is touched or seen; this
   is black-box evaluation.
3. The results appear on the agent's page like any other evaluation
   run. The run records the agent version as usual and, for an A2A
   agent, the card's own version string read at run time, so that a
   later change in behavior can be matched to a change on the
   provider's side.
4. The results gate the lifecycle decision (next journey): the
   record is promoted to `active` once the agent meets the
   organization's bar and stays `draft` otherwise.
5. The team reruns the same evaluation on a schedule, because a
   third party can change the agent behind the endpoint without
   notice, and the agent's page holds the history of runs.

This journey needs an endpoint whose protocol MLflow can speak:
`a2a`, invoked through the card's declared interface, or `mcp`,
through the MCP handshake. An `other` endpoint records where the
agent lives but does not tell MLflow how to call it, so it does not
enable endpoint-driven evaluation.

### Manage an agent's lifecycle

An agent owner or platform team needs agents to carry an explicit,
auditable lifecycle state.

1. An agent version starts as `draft`: visible in the registry while
   the owner iterates.
2. The owner runs evaluations and reviews scores and traces (previous
   journey).
3. Satisfied with quality, the owner promotes the version to
   `active`, manually or from CI. Promotion is informed by evals but
   not gated on them; the registry records the decision, it does not
   make it.
4. When a version is superseded or found vulnerable, it transitions
   to `deprecated`: still visible, marked as superseded, discouraged
   from new use.
5. Every transition is recorded as an auditable event with a
   timestamp and an actor, whether the actor is a human or a CI/CD
   identity.

This is the same core `draft` → `active` → `deprecated` lifecycle
the MCP and Skill registries use (their soft-delete `deleted` state
and transition rules carry over as well), applied to the agent
itself. The auditable transition history is the accountability
chain that problem
1 identifies as missing: for any agent, the registry can say who
promoted it, when, and what its evaluation evidence looked like at
the time.

### Assess the blast radius of a compromised component

A security engineer learns a component is compromised and must find
every affected agent without inspecting deployments one by one.

1. The skill `k8s-troubleshooter` version `1` is flagged as
   compromised.
2. The engineer transitions that skill version to `deprecated` in the
   Skill Registry.
3. They query the Agent Registry for consumers of the compromised
   version:
   ```python
   versions = mlflow.genai.search_agent_versions(
       filter_string=(
           "bom.skill.name = 'k8s-troubleshooter' "
           "AND bom.skill.version = 1"
       )
   )
   ```
   Dropping the version clause widens the query to consumers of any
   version, useful when every version of the skill is suspect or when
   assembling the full consumer list before deciding who is affected.
4. The registry returns the affected agent versions with their owners:
   for example, three agents across two teams.
5. The engineer contacts the owning teams, who ship new agent
   versions with the skill removed or upgraded, and deprecate the
   affected versions (previous journey).

The same query works for the other BOM axes ("which agents use MCP
server Y whose tool schema changed?", "which agents call model Z
being retired?", "which agents send prompt W, whose latest edit
regressed?") and for source entries: "which agents ship OCI
image X?" is the container-CVE variant. Agent references close the
highest-impact case: "which agents call the compromised agent?" is
a query over the same axis. Agent plugin references
expand through their members: plugin versions immutably record
which registered skills they contain, so the query also finds
agents that consume a skill through a plugin. Expansion covers the
member types plugin versions record: skill members today, with more
member types (MCP servers, subagents) extending the expansion as
the extended agent plugins proposal lands. The query has
exact-match semantics, and the name and version predicates must
bind to the same BOM entry (a store-level obligation for the
detailed design, like exact span-attribute matching in RFC-0009).
The registry answers with consumers and owners; automated
notification of those owners is deliberately not in the MVP (see
Out of scope).

Coverage has two limits: the query sees only the component types
the registry tracks, and within those, only what registrants
declared. Agents with undeclared composition can never match, so
results should surface them alongside matches: "3 agents declare
the compromised skill; 12 more have undeclared composition."

## Out of scope

The following are explicitly out of scope for this RFC. Several are
natural follow-ons; their exclusion here is sequencing, not
rejection.

- **Runtime state.** Health, liveness, deployment status, scaling,
  and placement are the serving platform's domain. The registry
  stores no runtime state and performs no polling or health checks. A
  platform's runtime view can join registry records against its own
  inventory at query time; the full "which *running* agents are
  affected?" question is that join, with this registry supplying the
  consumer-and-owner half.
- **Deployment and orchestration.** The registry does not deploy
  agents. Access endpoint records say where an approved endpoint
  is; they do not create it.
- **Registry synchronization from deployments.** Auto-registering
  agents when they deploy, keeping BOMs fresh when composition
  changes at deploy time, and maintaining deployment trace-location
  links (see the trace journey) calls for platform-side glue (for
  example a Kubernetes controller) pushing to the registry APIs
  this RFC defines. Deferred.
- **Auto-discovery of composition.** BOMs are developer-asserted in
  the MVP. Inferring composition from traces is deliberately
  deferred, including the user-initiated form: select traces, infer
  the observed skills, MCP servers, models, and prompts, review the
  proposed BOM, and register it as a draft version; and, once a
  version exists, flag traces recorded against it that use
  undeclared components or different component versions. That is
  the intended shape of the follow-on, and this RFC lays its
  groundwork: every trace carries the agent version it was recorded
  against, which is what drift detection compares. The other half
  is trace conventions that identify components by registry
  reference, and coverage is uneven today. Prompts have one: MLflow
  links a prompt loaded from the Prompt Registry to the active
  trace by name and version. The skill tracing proposal supplies
  one for skills (`SKILL` spans). LLM spans record the provider's
  model name, which matches an external model identifier in the BOM
  but not a Model Registry entry. Tool spans record a tool name and
  no MCP server identity, so MCP server inference would rely on
  matching tool names against registered servers' tool lists. The
  follow-on should close those gaps before it is built.
- **Detection of unregistered agents.** Surfacing "shadow" agents
  running without registry entries requires runtime scanning,
  which is platform work built on top of this registry.
- **Automated notifications.** The blast-radius journey ends with the
  registry naming owners; notifying them is left to the organization
  in the MVP.
- **Agent-to-agent discovery beyond search.** The registry answers
  the first half of "find me an agent that can do X and call it": a
  developer or an agent can search it, and an `a2a` or `mcp`
  access endpoint leads to a live endpoint whose protocol
  describes the rest, an Agent Card in one case and the MCP
  handshake in the other. The second half is a gateway concern:
  routing requests, choosing among live instances by health or
  load, and mediating authentication are not registry functions.
- **Cross-referencing MCP Server Registry entries from `mcp`
  endpoints.** An agent exposed as an MCP server records its `mcp`
  endpoint on its own record. The same server may also be
  registered in the MCP Server Registry, and nothing links the two
  for now; linking them is a possible later addition if the
  duplication turns out to matter.
- **Cost attribution.** Per-agent token cost is an observability
  rollup over agent-linked traces, not registry metadata.
- **Cross-workspace federation.** Discovery across registries is
  future work, potentially via A2A.
- **Discovery for reuse as a first-class journey.** Registry listings
  are workspace-scoped and searchable, which gives teams a working
  answer to "what do we have?", but curated cross-team browsing
  experiences are not an MVP goal.

# Detailed design

## Design positions

The following positions are settled enough to draft against; the
remainder of the detailed design is TBD.

**The registry is record-level, not runtime-aware.** It stores what
an agent is, not whether it is running, healthy, or scaled. Runtime
state belongs to the serving platform, which can join its own
runtime inventory against registry records at query time. This is
the same division of responsibility RFC-0004 draws between the MCP
registry and a gateway.

The registry manages two primary entities under the `mlflow.genai`
SDK namespace, following the pattern of RFC-0004 and RFC-0008:

- **Agent**: a named, owned entity addressed by the same
  coordinates the Skill Registry uses, `(workspace, organization,
  name)`, with `organization` an optional field rather than a
  segment of the name, and referenced in the same grammar
  (`agents:/@acme/billing-agent/2`). Nothing external constrains
  agent naming (an A2A card's name is free-form), so consistency
  with the sibling MLflow-native registry decides; MCP server names
  differ only because the upstream MCP registry specification fixes
  them.
- **AgentVersion**: an immutable snapshot of the agent's composition,
  its **bill of materials (BOM)**: skill references, agent plugin
  references (a plugin is referenced as a composed unit and expands
  through its registered members for queries), MCP server
  references, model references (registry models or external model
  identifiers such as `gpt-4o`), prompt references (Prompt Registry
  entries, which cover system prompts and any other prompt the agent
  sends a model), references to other agents it calls
  (pinned to a version when the referencing team controls the
  callee's deployment, as with a set of agents versioned and
  deployed together as one application, and name-level when the
  callee is independently managed), and a reference to what runs
  the agent: a harness reference for agents that run as
  configurations of a packaged application (OpenCode, Claude Code),
  or a framework reference for agents built on an agent framework
  (LangGraph, CrewAI), each with a version. Harness and framework
  values come from a set of well-known identifiers shipped with
  MLflow, with `other` plus a free-text name as the escape hatch,
  the same shape as the endpoint protocol field, so that spelling
  variants of well-known names cannot fragment queries. Each
  version also carries at least one **definitional anchor**: source
  provenance, as one or more typed source pointers of the kinds the
  Skill Registry supports (a Git repo and ref, an OCI image, a zip
  archive, or direct MLflow artifact storage), which for a
  harness-based agent point at its configuration. An agent
  registered from an A2A endpoint
  alone is an interface-only record with no anchor (see below).
  Each change to composition is a new version.

Version identity is a per-agent choice made when the agent is
created: `monotonic` (registry-assigned serial numbers: 1, 2, 3),
`semver` (registrant-supplied semantic versions), or `freeform`
(registrant-supplied opaque strings). `monotonic` is the default
when registration supplies no version: registrants who never think
about versioning get serial numbers automatically, while an agent
that already carries its own versioning (a provider-versioned A2A
agent, for example) can keep it. The scheme may also be
autodetected from the first registration's input. This refines the
policy the earlier registries establish, where an entry adopts the
underlying artifact's version when its format defines one
(RFC-0004's `server_json`, RFC-0008's plugin manifests) and mints
serial numbers when it does not (RFC-0008's skills): no standard
agent artifact defines an inherent agent version, so minting is the
default rather than the rule. An A2A card's provider-defined
`version` string lives on the live card rather than in the registry;
an agent registered under `semver` or `freeform` may mirror it.
Aliases and `latest` resolution are well defined for `monotonic`
and `semver`; `freeform` versions order by registration time.

BOM entries are soft references, structured values rather than
foreign keys. They resolve against the Skill Registry (which
RFC-0008 defines for both skills and agent plugins), MCP Server
Registry, Model Registry, and Prompt Registry when matching entries
exist, and they remain valid when they do not. This makes
cross-registry questions ("which agents use skill X?") answerable
as registry queries without constraining registration order.

**A version's BOM is exported in resolved form.** The soft
references are what the registry stores; integrating systems
usually want the expansion. The registry therefore serves any
version's BOM as a JSON document in which each reference that
matches a registry entry is expanded to that entry's record (a
skill version, an MCP server version with its server definition, a
prompt version with its text, a registry model version) and each
that does not is kept as the pointer and marked unresolved. Aliases
are pinned to the concrete version at export time, and the document
records when it was resolved, since aliases move. Agent plugins are
expanded to their members, and agents the version calls are
resolved one level, with nested BOMs reachable by repeating the
call. The SDK and REST API expose this as an option on fetching a
version, and it is the same document the scan path writes back
into a source tree.

**The BOM is a component inventory, not a complete recipe.** Its
structured axes exist because corresponding registries or identifier
conventions exist, so it is bounded by MLflow's governance surface
rather than by agent anatomy: an agent's prompts have an axis
because the Prompt Registry governs them, while its memory
configuration or context-compaction strategy has none because
nothing governs one. Three layers
share the job of describing an agent. Structured BOM references are
selective but queryable across agents. Definitional anchors (source
pointers) are complete but opaque: they
capture everything about one agent without supporting cross-agent
queries. Free-form tags are the catch-all for facts that fit
neither. New structured axes are expected as the governance surface
grows.

**A2A Agent Cards are fetched, not stored.** The
[A2A protocol](https://a2a-protocol.org/) makes an agent's endpoint
the authoritative home of its Agent Card: every A2A client reads
the live card from the endpoint's well-known path, and the registry
follows suit. Registering from an endpoint stores three things
from the card in ordinary registry fields: its description, its
free-form name (which seeds the mutable MLflow-managed
`display_name`), and its skills list with each entry's tags, which
is what lets an agent be searched for by what it does. A2A's
"skills" are the card's own account of what the agent can do, not
Skill Registry entries; the two share a word and nothing else.
Registration also creates an `a2a` access endpoint. The card as a
document is not persisted. The display name stays,
although the Skill Registry dropped its own, because an agent's
registry name carries no human-readability guarantee: a skill name
is a slash command users type, readable by construction, while an
agent's identity comes from a protocol name or an endpoint path,
named for a purpose, an operation, or both. The UI renders the card
read-only by fetching it through the endpoint at view time, so what
MLflow displays can never drift from what the agent serves. This
deliberately departs from the canonical-payload pattern of RFC-0004
(`server_json`) and RFC-0008 (`plugin.json`): MLflow is the system
of record for those payloads, while an Agent Card's system of
record is the agent itself. An agent registered from an endpoint
alone, with no source, is an
**interface-only record**: the registry captures the claim surface
(identity, imported metadata, endpoint) and marks that it holds no
definitional anchor. Anchored records must declare their
composition, because the BOM is the registry's value and a
registrant with the source in front of them can supply it.
Interface-only records may declare composition as partial or
undeclared, because the registrant of a black-box agent cannot
supply it truthfully, and forcing a declaration would invite
invented BOMs that pollute cross-registry queries. An absent BOM is
recorded as *undeclared* composition rather than an empty
dependency list: the registry knows the agent's claim surface, not
its contents. Such a record is as thin as it sounds, name,
description, and endpoint, and nothing more is required to promote
it to `active`: whether a black-box agent is fit for use is a
judgment the vetting journey's evaluations inform, not a schema
gate.

**Endpoints are separate records, not version fields.** Some agents
are reachable at a URL (A2A agents inherently; deployed agents
generally), and recording that URL lets the registry drive tracing
and evaluation for agents whose code the user cannot run. It is
also what lets MLflow be the glue between the many systems that
define and serve agents: the registry is authoritative about which
agents exist and where each is reached, while those systems stay
authoritative about running them, and the registry persists no
runtime state. But
endpoints change independently of composition, and agent versions
are immutable. Following the MCP Server Registry's access endpoint
model (`MCPAccessEndpoint` in the implementation; the RFC text
still calls it `MCPAccessBinding`), approved endpoints are separate
mutable **access endpoint** records that target a version or
alias, created and deleted as connectivity changes without touching
version history. Where an MCP access endpoint's protocol is always
MCP, an agent's declares it: `a2a`, `mcp` (for agents exposed as
MCP servers), or `other`. The record may
also carry a free-text `description` and a `platform_url`, both
optional: the description tells a caller how to use an endpoint
whose protocol does not say, and the platform URL points at the
serving platform's own view of the deployment, so that runtime
state stays with the platform while the registry records where to
find it. Registration accepts an optional endpoint as a
convenience that creates the record.

**For GenAI work, an agent is the entity users create, not an
experiment.** Today traces and evaluation runs attach to
experiments, an abstraction that fits model training but not the
agent development loop. This RFC makes the agent the thing a GenAI
user creates and traces against: the agent's page carries the
traces-and-evaluations experience, and experiments continue as the
entity for model training and, underneath, as the storage and
permission unit for agent traces. Mechanically the change is
additive: a new `MlflowAgentTraceLocation` joins the existing
`MlflowExperimentLocation` (named to avoid confusion with an
agent's endpoint; a shorter name is welcome), usable wherever
MLflow accepts a trace destination today, and evaluation and
trace-search APIs gain agent identity alongside experiment
identity. A destination identifies the agent and optionally one of
its deployments, never the version; the version is recorded on
every trace and evaluation run as metadata, which is what
per-version filtering and comparison use. An agent has one or more
trace locations: a default, fixed when the agent is created, plus
one for each deployment that needs its own. Each location is an
experiment underneath, which is what makes the separation
enforceable: MLflow permissions are experiment-scoped, so audiences
are kept apart by location, not by trace tag. Versions never get
locations of their own; a version is an analysis dimension, not an
access boundary, and per-version locations would break the
longitudinal view of an agent's behavior across upgrades. The
agent's page presents the locations as deployments of the agent,
enumerating and searching across all of them. Existing
experiment-based workflows continue unchanged.

# Drawbacks

TBD.

# Alternatives

TBD.

# Adoption strategy

TBD.

# Open questions

- **Is the default experiment per agent or per agent-version?** This
  RFC says per agent: a version is an analysis dimension recorded on
  every trace, one experiment preserves the longitudinal view of an
  agent's behavior across upgrades, and the agent-to-experiment
  mapping stays one-to-one, with no version required to resolve a
  destination. The opposing position, held by at least one reviewer,
  is one default experiment per version, which partitions each
  version's traces physically, at the cost of making cross-upgrade
  monitoring a cross-experiment query and making the version a
  required argument wherever a destination is resolved. Whichever
  default is chosen, users who want the other behavior override it
  per deployment, so the question is which behavior makes the better
  default, not which is possible.

- **How should harness-based agents be described?** Agents that run
  as configurations of a packaged harness (Claude Code, OpenCode,
  Goose) have no user source repository; the agent is the harness
  plus its configuration. This RFC proposes two mechanisms for them,
  going beyond the code-centric design the journeys otherwise
  follow: a harness reference axis in the BOM (an external
  identifier, like external model references), and treating the
  configuration as the version's source, pointed at through the same
  typed source pointers used for skill content. Sub-questions:

  - Should the set of files that constitutes a harness's
    configuration surface be defined by per-harness integrations
    (the harness integrations contemplated by the skill tracing
    proposal would be a natural home) rather than assembled by each
    registrant, given that the surface must also exclude content the
    BOM already governs?

- **Should a version record multiple sources?** This RFC says yes: a
  version's source provenance is a list of typed pointers, so an
  agent built from a Git repo and shipped as an OCI image records
  both, and source entries are queryable like BOM axes ("which
  agents ship image X?"). This deliberately diverges from RFC-0008,
  where each skill version has exactly one source and the content
  digest reconciles identical content registered from different
  sources. Agents have no defined content bundle to digest, so the
  skill pattern applied to agents would produce irreconcilable
  duplicate versions of what is really one agent. The list is
  currently an unordered set of asserted pointers. Reviewers who
  weigh cross-registry consistency heavily should push back here if
  the divergence is not worth it.

- **What is the BOM reference format?** The journeys sketch URI-style
  references (`skills:/billing-policy/1`,
  `mcp-servers:/acme.internal/payments-db/2.0.0`,
  `models:/acme-billing-llm/3`). The skill and agent schemes follow
  the Skill Registry's member references, including the
  `@organization` prefix. RFC-0004 defines no MCP URI scheme, so the MCP
  scheme adopts the `mcp-servers:/` proposal from RFC-0010; the refs
  here respect RFC-0004's reverse-DNS server names and semantic
  versions. Bare identifiers cover external models (`gpt-4o`). BOM
  references are soft (string-resolved, valid when the target is
  unregistered), which URI syntax may misleadingly suggest
  otherwise. Alternatives include structured
  `{registry, name, version}` objects.
