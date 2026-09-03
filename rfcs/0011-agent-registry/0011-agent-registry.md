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
(MCP servers), the expertise it carries (skills), and the models it
invokes (Model Registry), but the agent itself, the thing that acts,
has no registry entry. This RFC adds one.

In brief, the design takes these positions, each stated in full in
[Design positions](#design-positions): the registry is
record-level, not runtime-aware; an agent's versions are immutable
snapshots of its composition (a bill of materials of skills, agent
plugins, MCP servers, models, and other agents it calls) plus at
least one definitional
anchor (source pointers and/or a configuration snapshot); A2A Agent
Cards are fetched from the agent's endpoint, never stored;
endpoints are mutable, protocol-typed access bindings rather than
version fields; and an agent is a trace destination with one
default experiment, with the version recorded on every trace and
evaluation run.

**Relationship to other RFCs.** RFC-0004 establishes the access
binding pattern this RFC reuses (its canonical-payload pattern is
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
    name="acme/billing-agent",
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
    agents=["agents:/acme/records-agent/2"],
    mcp_servers=["mcp-servers:/acme.internal/payments-db/2.0.0"],
    models=["models:/acme-billing-llm/3", "gpt-4o"],
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
    name="acme/travel-agent",
    a2a_endpoint="https://agents.acme.internal/travel",
    skills=["skills:/itinerary-planning/4"],
)
```

Given an endpoint, the client SDK fetches the Agent Card from the
endpoint's well-known path, imports its descriptive metadata
(description, capabilities, and its free-form name, which seeds the
mutable MLflow-managed `display_name`), and creates an `a2a` access
binding for the endpoint. The card content is not persisted: the
endpoint is the card's system of record, and the UI renders the
card read-only by fetching it through the binding at view time. The
registry `name` is always chosen by the registrant, since a card's
`name` is a display string, not an identity. Fetches happen in the
client, never in the registry server, consistent with RFC-0008; a
caller can also fetch the card itself and pass it via `a2a_card=`
to inspect the imported metadata first. A version registered this
way, with no source and no configuration snapshot, is an
interface-only record (see the register journey).

## Register a harness-based agent

```python
mlflow.genai.register_agent(
    name="acme/oncall-helper",
    description="On-call assistant run in OpenCode.",
    harness="opencode",
    harness_version="0.5.3",
    skills=["skills:/runbook-triage/2"],
    mcp_servers=["mcp-servers:/acme.internal/pagerduty/1.2.0"],
    models=["claude-sonnet-5"],
    config_snapshot="./opencode.json",
)
```

There is no user source repository: the agent is the harness plus
its configuration, so the configuration snapshot serves as the
definitional anchor. `config_snapshot` names a file or directory on
the caller's local disk; the client reads it and uploads the
content to MLflow artifact storage as an immutable artifact on the
version, which means any redaction of secrets must happen
client-side before upload.

## Trace and evaluate against the agent

```python
mlflow.genai.set_active_agent("acme/billing-agent", version=3)

with mlflow.start_span(name="answer-question"):
    result = agent.run(question)

mlflow.genai.evaluate(
    data=eval_dataset,
    scorers=[correctness_scorer],
    agent_id="acme/billing-agent",
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
   MCP servers, and models, each versioned independently. No record
   captures which versions of which components a given build of the
   agent used. When behavior changes, "something changed and the
   agent broke; what was it?" requires reconstructing the composition
   from memory, commit history, and luck.

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
       name="acme/billing-agent",
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
   Required: name, description, and at least one of: a definitional
   anchor (source provenance or a configuration snapshot) or an A2A
   endpoint. These combine freely; a first-party A2A agent registers
   with both source and endpoint. Only a registration with an
   endpoint and no anchor produces an interface-only record (see
   below). Composition (the BOM) is required when an anchor is
   present and may be empty or partial for interface-only records
   (see below). Optional: tags, an explicit
   version where the agent's version scheme takes one (the default
   `monotonic` scheme assigns versions automatically), and an
   endpoint (a URL plus protocol; the endpoint journey below covers
   bindings).
2. MLflow creates an `AgentVersion` record with initial status
   `draft`.
3. The agent appears in the registry listing for its workspace, with
   its BOM entries linked to the skill, agent plugin, MCP server,
   and model registry pages where matching entries exist.
4. **A2A path:** an agent that serves an Agent Card registers from
   its endpoint. The UI registration form offers two modes, "import
   from A2A card" and "manual"; the import mode pre-fills
   descriptive and capability fields from the card (its free-form
   name seeds the mutable `display_name`; the registry `name` is
   supplied by the registrant) and creates an `a2a` access binding
   for the endpoint. In the SDK and CLI, the client fetches the
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
   harness/framework distinction) typically has no source repository
   of its own: the agent *is* the harness plus its configuration. It
   registers with a harness reference in place of user source, and
   attaches the configuration that defines it:
   ```python
   mlflow.genai.register_agent(
       name="acme/oncall-helper",
       description="On-call assistant run in OpenCode.",
       harness="opencode",
       harness_version="0.5.3",
       skills=["skills:/runbook-triage/2"],
       mcp_servers=["mcp-servers:/acme.internal/pagerduty/1.2.0"],
       models=["claude-sonnet-5"],
       config_snapshot="./opencode.json",
   )
   ```
   The configuration snapshot is read from the caller's local disk
   and stored as an immutable artifact on the version. It is what
   distinguishes this agent from every other installation of the
   same harness: enabled tools, overridden defaults, and behavioral
   settings live there and nowhere else.

   A harness's configuration surface is not always a single file
   (Claude Code, for example, spreads it across a settings file,
   instruction files, and subagent definitions), so
   `config_snapshot` accepts a file or a directory. It should
   capture only configuration the registry does not otherwise
   represent: content that BOM references already govern, such as
   installed skill directories or MCP server definitions, stays
   out, because an embedded copy is invisible to cross-registry
   queries and can drift from the declared references.

Composition is required wherever it is knowable, because the BOM is
the value: a registry record without composition is just a name in
a list. A registrant anchoring on source or a configuration
snapshot has the composition in front of them. The registrant of a
black-box vendor or partner agent does not, and forcing a
declaration would invite invented BOMs that pollute cross-registry
queries; for interface-only records the BOM may therefore be empty
or partial, an absent BOM is recorded as *undeclared* composition
rather than an empty dependency list, and the record is marked as
holding no definitional anchor: the registry knows the agent's
claim surface, not its contents. Everything else is progressive
enrichment. The endpoint is optional; the trace-and-eval journey
below explains which agents need one.

The harness path is the newest part of this design and the least
settled (see [Open questions](#open-questions)). For framework-built
and custom agents, the registered source is the natural complete
record and remains the expected anchor; for harness-based agents,
requiring source would force registrations that point at the
harness vendor's repository, which identifies nothing about the
specific agent. Configuration files frequently embed secrets and
environment-specific values, so the snapshot mechanism needs
redaction guidance at minimum.

### Publish and maintain an agent's endpoint

A platform operator deploys a registered agent and needs the
registry to say where, and how, it can be reached, without
disturbing the immutable version history.

1. The agent is already registered (any path above) and a version
   has been promoted. The platform team deploys it.
2. The operator creates an access binding for the deployment,
   targeting a version or an alias and declaring the endpoint's
   protocol:
   ```python
   mlflow.genai.create_agent_access_binding(
       agent="acme/billing-agent",
       target_alias="production",
       endpoint_url="https://agents.acme.internal/billing",
       protocol="a2a",
   )
   ```
   The endpoint accepted at registration time is sugar for creating
   a binding; the A2A registration path creates an `a2a` binding
   automatically.
3. The agent's detail page lists its bindings. Bindings whose
   protocol is self-describing (`a2a`, `mcp`) are actionable: they
   are the entry points for the endpoint-driven tracing and
   evaluation in the trace-and-eval journey below. An `other`
   binding is a documented pointer.
4. The deployment moves to a new URL. The operator updates the
   binding; no version record changes.
5. The deployment is retired. The operator deletes the binding; the
   agent, its versions, and its history remain untouched.

The protocol field is where agent bindings depart from RFC-0004,
whose bindings are always MCP and vary only by transport. The field
is limited to values that tell a caller something actionable: `a2a`
and `mcp` are self-describing (an Agent Card at the well-known
path; the MCP handshake), so URL plus protocol is enough to
connect. Labels like REST or gRPC name a transport without telling
anyone how to call the agent, so they are deliberately collapsed
into `other`, which records where an agent lives without claiming
MLflow can invoke it. As in RFC-0004, a binding that targets an
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
   `llama-3.1-405b`, one MCP server added. When both versions carry
   configuration snapshots, those diff as content: for example, v4
   switched the harness's shell configuration.
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
   mlflow.genai.set_active_agent("acme/billing-agent", version=3)

   with mlflow.start_span(name="answer-question"):
       result = agent.run(question)
   ```
   `set_active_agent` is convenience over two separable pieces.
   First, it sets the trace destination to the agent's one default
   experiment, equivalent to calling `mlflow.set_experiment` on the
   result of
   `mlflow.genai.get_default_experiment_id("acme/billing-agent")`,
   where `get_default_experiment_id` is a public lookup that takes
   only the agent: the version is never part of the destination. (An
   `MlflowAgentLocation` naming the agent works anywhere MLflow
   accepts a trace destination.) Second,
   it records the agent and version as trace-level metadata, the way
   session and user metadata are recorded today; this is what
   per-version filtering and comparison use. A deployment that
   overrides its destination (below) swaps only the first piece,
   pointing `set_experiment` at its own experiment, and still
   declares the agent and version so its traces stay labeled.
   Framework and harness autologgers respect the active destination
   and metadata, so instrumented applications need only state which
   agent they are.
2. Run evaluations against the agent:
   ```python
   mlflow.genai.evaluate(data=eval_dataset,
                         scorers=[correctness_scorer],
                         agent_id="acme/billing-agent",
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

The default experiment is a default, not a router. A running agent
logs traces to whatever destination its own deployment
configuration sets; the registry is not in the call path. When
nothing is set, traces land in the agent's default experiment,
which is the right behavior for the development loop and
registry-driven evaluations. Scale-out replicas of one deployment
share its configuration, so their traces aggregate without further
arrangement. A deployment that needs its traces kept separate from
other deployments of the same agent (a different owner or user
base) overrides the destination in its own configuration; because
permissions are experiment-scoped, separation is done with
destinations, not trace tags. Versions share the agent's experiment
for the same reason: a version is an analysis dimension recorded on
every trace, not an access boundary, and per-version experiments
would break the longitudinal view of an agent's behavior across
upgrades. So that overridden deployments stay findable from the
agent's registry page, this RFC proposes that a deployment using a
non-default experiment notify the registry, which records the
experiment ID on that deployment's access binding. A deployment
with no binding would need another place to record the link, which
is one reason this mechanism is a proposal rather than settled.
Automating the setup and upkeep of these links at deploy time
belongs to the deferred registry synchronization glue.

No endpoint is needed for any of this when the developer has the
agent's code: the agent runs locally or in CI, autologging captures
traces during execution, and evaluation scores outputs against a
test dataset. The exception is agents whose code the user
cannot run: another team's A2A agent, a vendor agent, a partner
service. For those, the endpoint is the only execution surface, and
tracing and evaluation work by invoking the agent's access binding
with test inputs and observing responses. This track requires a
binding whose protocol MLflow can speak: `a2a` (invoked through the
card's declared interface) or `mcp` (through the MCP handshake). An
`other` binding records where the agent lives but does not by
itself tell MLflow how to call it, so it does not enable
endpoint-driven tracing or evaluation. The registry supports both
tracks; the optional endpoint exists largely for the second.

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
being retired?") and for source entries: "which agents ship OCI
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
  agents. Access bindings record where an approved endpoint is; they
  do not create it.
- **Registry synchronization from deployments.** Auto-registering
  agents when they deploy, keeping BOMs fresh when composition
  changes at deploy time, and maintaining deployment trace-location
  links (see the trace journey) calls for platform-side glue (for
  example a Kubernetes controller) pushing to the registry APIs
  this RFC defines. Deferred.
- **Auto-discovery of composition.** BOMs are developer-asserted in
  the MVP. Inferring actual composition from traces (for example,
  from RFC-0009 `SKILL` spans) and notifying owners when assertion
  and observation disagree is a follow-on that the span data from
  this RFC and RFC-0009 is designed to enable.
- **Detection of unregistered agents.** Surfacing "shadow" agents
  running without registry entries requires runtime scanning,
  which is platform work built on top of this registry.
- **Automated notifications.** The blast-radius journey ends with the
  registry naming owners; notifying them is left to the organization
  in the MVP.
- **Agent-to-agent runtime discovery.** A programmatic "find me an
  agent that can do X and call it" surface for running agents is a
  gateway concern, as is any request routing.
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

- **Agent**: a named, owned entity with DNS-style naming
  (`org/agent-name`), in the spirit of the namespaced names the MCP
  and Skill registries use; exact alignment with RFC-0008's
  `{workspace, organization, name}` coordinates is a detailed-design
  point.
- **AgentVersion**: an immutable snapshot of the agent's composition,
  its **bill of materials (BOM)**: skill references, agent plugin
  references (a plugin is referenced as a composed unit and expands
  through its registered members for queries), MCP server
  references, model references (registry models or external model
  identifiers such as `gpt-4o`), references to other agents it calls
  (pinned to a version when the referencing team controls the
  callee's deployment, as with a set of agents versioned and
  deployed together as one application, and name-level when the
  callee is independently managed), and, for agents that run as
  configurations of a packaged harness, a harness reference (a
  proposed axis; see [Open questions](#open-questions)). Each
  version also carries at least one **definitional anchor**: source
  provenance (one or more typed source pointers as in RFC-0008: a
  Git repo and ref, an OCI image, an archive) and/or an immutable
  configuration snapshot. An agent registered from an A2A endpoint
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
Registry, and Model Registry when matching entries exist, and they
remain valid when they do not. This makes cross-registry questions
("which agents use skill X?") answerable as registry queries
without constraining registration order.

**The BOM is a component inventory, not a complete recipe.** Its
structured axes exist because corresponding registries or identifier
conventions exist, so it is bounded by MLflow's governance surface
rather than by agent anatomy: an agent's prompt strategy or memory
configuration has no axis because nothing governs one. Three layers
share the job of describing an agent. Structured BOM references are
selective but queryable across agents. Definitional anchors (a source
pointer, a configuration snapshot) are complete but opaque: they
capture everything about one agent without supporting cross-agent
queries. Free-form tags are the catch-all for facts that fit
neither. New structured axes are expected as the governance surface
grows.

**A2A Agent Cards are fetched, not stored.** The
[A2A protocol](https://a2a-protocol.org/) makes an agent's endpoint
the authoritative home of its Agent Card: every A2A client reads
the live card from the endpoint's well-known path, and the registry
follows suit. Registering from an endpoint imports the card's
descriptive metadata (description, capabilities, and its free-form
name, which seeds the mutable MLflow-managed `display_name`) into
ordinary registry fields and creates an `a2a` access binding; the
card content itself is not persisted. The UI renders the card
read-only by fetching it through the binding at view time, so what
MLflow displays can never drift from what the agent serves. This
deliberately departs from the canonical-payload pattern of RFC-0004
(`server_json`) and RFC-0008 (`plugin.json`): MLflow is the system
of record for those payloads, while an Agent Card's system of
record is the agent itself. An agent registered from an endpoint
alone, with no source and no configuration snapshot, is an
**interface-only record**: the registry captures the claim surface
(identity, imported metadata, endpoint) and marks that it holds no
definitional anchor.

**Endpoints are access bindings, not version fields.** Some agents
are reachable at a URL (A2A agents inherently; deployed agents
generally), and recording that URL lets the registry drive tracing
and evaluation for agents whose code the user cannot run. But
endpoints change independently of composition, and agent versions
are immutable. Following RFC-0004's `MCPAccessBinding` model,
approved endpoints are separate mutable binding records that target
a version or alias, created and deleted as connectivity changes
without touching version history. Where an MCP binding's protocol
is always MCP, an agent binding declares its protocol: `a2a`, `mcp`
(for agents exposed as MCP servers), or `other`. Registration
accepts an optional endpoint as a convenience that creates a
binding.

**Agents become the primary anchor for GenAI traces and
evaluations.** Today traces and evaluation runs attach to
experiments, an abstraction that fits model training but not the
agent development loop. This RFC makes an agent a trace
destination: a new `MlflowAgentLocation` joins the existing
`MlflowExperimentLocation`, usable wherever MLflow accepts a trace
destination today, and evaluation and trace-search APIs gain agent
identity alongside experiment identity. A destination identifies
the agent only and resolves to the agent's one default experiment;
the version is never part of the destination and is instead
recorded on every trace and evaluation run as metadata, which is
what per-version filtering and comparison use. Traces and eval
results appear on the agent's registry page, filterable by version.
The change is additive: the default experiment exists under the
hood, and experiment-based workflows continue unchanged.

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

- **How thin may an interface-only record be?** A black-box A2A
  agent registers with a name, metadata imported from its card, an
  access binding, undeclared composition, and no definitional
  anchor. Is that enough of a record to be worth governing, and
  should the registry require anything more of it before such a
  record can be promoted to `active`?

- **Should agent-centric traces and evaluations be a separate RFC?**
  The experiments bridge (`agent_id` resolving to a default
  experiment) touches tracing APIs, evaluation APIs, and UI surface
  area well beyond the registry itself, and there is precedent for
  splitting: RFC-0008 defined the Skill Registry and RFC-0009
  followed with skill tracing. The counter-argument is that
  agent-anchored traces and evals are the registry's core value; a
  registry without them is a list of names. This RFC keeps the
  journey in scope on the additive framing above, but if reviewers
  prefer a narrower registry RFC, the journey splits cleanly along
  the RFC-0008/0009 seam.

- **Do endpoint records belong in MLflow at all?** The access binding
  model resolves the mechanical objections to endpoints (mutability
  against immutable versions, staleness on version records), and
  RFC-0004 sets the precedent. The remaining objection is
  architectural: an endpoint could be considered runtime metadata, and
  the record/runtime boundary could place all of it on the platform
  side. The position taken here is that "where an approved endpoint
  for this agent is" belongs to the record, while "whether anything
  answers there" belongs to the platform, and that without endpoints
  the registry cannot trace or evaluate agents whose code the user
  cannot run. This boundary needs explicit review. A related
  sub-question: when an agent is exposed as an MCP server, should an
  `mcp` binding cross-reference the MCP Registry entry for the same
  endpoint instead of duplicating it?

- **How should harness-based agents be described?** Agents that run
  as configurations of a packaged harness (Claude Code, OpenCode,
  Goose) have no user source repository; the agent is the harness
  plus its configuration. This RFC proposes three mechanisms for
  them, going beyond the source-centric design the journeys otherwise
  follow: a harness reference axis in the BOM (an external
  identifier, like external model references), an immutable
  configuration snapshot stored as a version artifact to serve as the
  definitional anchor, and a relaxation of the required fields from
  "source" to "at least one definitional anchor" (source,
  configuration snapshot, or A2A card). Sub-questions:

  - Do configuration snapshots invite secret leakage badly enough to
    need enforced redaction rather than guidance?
  - Should a harness axis wait for some notion of harness identity
    governance?
  - What are the identifier semantics of a harness reference? The
    options are an opaque asserted name/version pair; an open
    vocabulary in which free-text names are always accepted but
    names matching MLflow's known harness integrations are
    recognized and normalized; or a resolvable package identity. The
    open vocabulary is the likely landing: fragmented spellings
    would undermine cross-agent queries, while no single package
    ecosystem could serve as an authority given how heterogeneously
    harnesses are distributed.
  - Should the set of files that constitutes a harness's
    configuration surface be defined by per-harness integrations
    (the harness integrations contemplated by RFC-0009 would be a
    natural home) rather than hand-picked by each registrant, given
    that the surface must also exclude content the BOM already
    governs?
  - Does configuration-as-artifact belong in this RFC or a
    follow-on?

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
  `models:/acme-billing-llm/3`). The skill scheme follows RFC-0008's
  member references. RFC-0004 defines no MCP URI scheme, so the MCP
  scheme adopts the `mcp-servers:/` proposal from RFC-0010; the refs
  here respect RFC-0004's reverse-DNS server names and semantic
  versions. Bare identifiers cover external models (`gpt-4o`). BOM
  references are soft (string-resolved, valid when the target is
  unregistered), which URI syntax may misleadingly suggest
  otherwise. Alternatives include structured
  `{registry, name, version}` objects.
