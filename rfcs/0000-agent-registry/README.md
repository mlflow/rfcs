---
start_date: 2026-10-08
mlflow_issue: # TBD
rfc_pr:
---

# Agent Registry

## Summary

Make agents the primary GenAI container in MLflow. Each agent has a persistent identity that brings
traces and evaluations together. Eligible experiments can be
converted to agents, while experiments remain supported for existing workflows and model training.
Optional agent fleets provide views across cooperating agents.

OSS MLflow provides the default registry backend. Managed deployments can instead resolve agent
records from an authoritative catalog or gateway, such as Unity AI Gateway. This draft seeks
agreement on product direction; storage and API details remain subject to follow-up design.

## Motivation

An agent should have a persistent identity in MLflow. Its
registry page would bring evaluations and traces together, including production quality, latency,
and cost, with links to the underlying traces. Teams could compare evaluation results and
investigate production regressions using the same record.

Today, MLflow captures traces and evaluations but organizes GenAI work primarily under experiments,
even for production applications. "Experiment" suggests a temporary development activity. An
experiment's agent versions are currently logged models which provide version tracking, but there is
no persistent agent registry identity. The proposal keeps the development loop—tracing, evaluating,
and revising prompts or tools—under the agent identity.

### Why MLflow

MLflow holds the behavioral evidence used to decide whether an agent is ready and to explain how it
behaves after deployment. Linking that evidence to an authoritative agent record puts it where teams
and assistants inspect it. OSS MLflow supplies the default registry. A managed deployment can keep
an existing catalog or gateway (e.g. Unity AI Gateway) authoritative while MLflow resolves its agent
records.

### Out of scope

- Hosting agents, proxying invocations, or managing runtime health and scaling.

## User journeys

Tracing and evaluation still work as they do today, with an agent as the container instead of an
experiment.

### Start tracing an agent

1. Create an agent in the UI. It gets a persistent identity before deployment.
2. Select it by ID with `mlflow.set_agent(...)` and run the instrumented application.
3. Open the agent's **Traces** page.

### Review a fleet of agents

1. An operator creates a fleet and adds several cooperating agents.
2. A developer with read access opens the fleet to compare quality and usage, then filters its
   traces and sessions to selected agents.
3. From the fleet's **Agents** tab, they open one agent to investigate its activity.

## Detailed design

### Agents as the new GenAI container

#### What that means

The GenAI landing page becomes **Agents**. Users can create an agent in the UI or SDK, or convert an
eligible experiment. Agents can be registered before deployment.

The **Agents** page keeps the existing navigation and workflows for traces, evaluations, and usage
from experiments, but does not support **Agent versions** in this proposal. A future RFC building on
this foundation will address agent versioning. The agent overview retains the GenAI experiment
overview. The main **Agents** page also supports filtering by fleet. Separate agent containers are
recommended for separate environments, such as development, staging, and production; environments
are not modeled by this proposal.

Traces, evaluation runs, judges, and datasets can be associated with an agent instead of an
experiment. Each belongs to exactly one container. Relevant SDK and REST APIs accept the agent
association; autologging, trace ingestion, and trace search support it for traces.

Each agent has a globally unique, stable ID and a mutable name unique within its workspace.
Agent-level APIs use the ID; names support search and display.

Illustrative API changes, with final names TBD:

```python
import mlflow
from mlflow import MlflowClient

client = MlflowClient()
agent_id = client.create_agent(name="support-agent")
mlflow.set_experiment(experiment_id="123")  # Existing workflow
mlflow.clear_active_destination()  # Switch from experiment to agent
mlflow.set_agent(agent_id=agent_id)

mlflow.search_traces(experiment_ids=["123"])
mlflow.search_traces(agent_ids=[agent_id])

mlflow.genai.create_dataset(name="support-eval", agent_id=agent_id)
client.rename_agent(agent_id, "support-assistant")
# create_dataset(..., experiment_id="123") remains supported.
```

Explicit experiment and agent arguments are mutually exclusive. Setting an active agent while an
experiment is active, or an experiment while an agent is active, is an error.

Deployments can select an agent without calling `set_agent` by setting `MLFLOW_AGENT_ID`. An explicit
`set_agent` or `set_experiment` call takes precedence; without one, agent and experiment selectors
supplied through environment variables cannot both be set.

#### Trace ingestion

OTLP request headers provide defaults; root-span attributes can override them:

| Field | Request header      | Root-span attribute |
| ----- | ------------------- | ------------------- |
| Agent | `x-mlflow-agent-id` | `mlflow.agent.id`   |

This allows one export to contain traces from different agents. MLflow validates each agent and
agent permissions.

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

#### Tracing and observability

Instance-level filtering is separate from introducing agent containers, but strengthens the
observability experience within them and should be prioritized alongside this work.

OTLP ingestion retains and indexes `service.instance.id` for SDK and REST trace search. The agent
**Traces** view offers an **Instance ID** filter; a trace matches when one of its spans comes from
that instance. This assumes one `service.namespace`/`service.name` pair sends traces to each agent
container.

Evaluation runs stay under the agent for comparison.

#### Registry backend

Agent registry backends use a new plugin interface, separate from the tracking-store interface,
that the tracking store can hook into. The MLflow tracking store provides the default backend for
agent identity and container metadata in OSS MLflow. A deployment can instead use another
authoritative backend (e.g. Unity AI Gateway) to
resolve agent records and enforce agent permissions. MLflow associates traces and evaluations with
the resolved agent; the external agent record need not be copied into the tracking store's agent
registry tables. The configured backend supplies globally unique IDs and enforces workspace-unique
names. The plugin interface and UI behavior for alternative backends will be defined during
implementation.

#### Backwards compatibility and migration

Provide **Convert to agent** in the experiment UI and an equivalent API. Conversion creates or
selects a record through the configured registry backend, preserves existing traces, runs,
evaluations, and resource references, and presents them under the agent. Permissions follow the
configured backend's rules.

The GenAI experiment UI currently shows Logged Models as **Agent versions**, which agent containers
do not support in this proposal. If an experiment has any Logged Models, the UI warns and the API
rejects conversion before changing the experiment.

Existing experiments continue to work. After conversion, existing SDK calls and OTel exporters using
the old experiment ID must keep working through a compatibility mapping to the agent, except for
Logged Model operations. Clients can migrate incrementally.

### Data schema

Adapt the `Agent` concept from the [original draft](https://github.com/mlflow/rfcs/pull/39).
Logged Models continue to serve experiment-based workflows, where the GenAI UI currently labels them
**Agent versions**.

The following Python types illustrate the main read responses and relationships:

```python
from typing import TypedDict


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
    tags: dict[str, str]
    created_by: str | None
    last_updated_by: str | None
    creation_timestamp: int | None
    last_updated_timestamp: int | None
```

Public APIs use the current workspace when no workspace is supplied.

Audit fields follow the MCP and Skill registries. Agent audit fields can come from the configured
backend. `owner` identifies who is responsible for an agent, separately from who created or last
updated its record.

## Future direction

This RFC covers the agent container: agent records and access checks, GenAI APIs and UI, optional
fleets, experiment conversion, and compatibility for existing clients.

A follow-up RFC can build on this foundation with immutable snapshots, linked assets, endpoint
discovery, and metadata syncing, followed by observed tool and agent relationships with trace
provenance.

## Drawbacks

- Moving GenAI ownership from experiments to agents affects many APIs, permissions, and UI flows;
  migration is a substantial part of the implementation.

## Open questions

- Should datasets eventually be decoupled from agents and experiments for reuse across containers?
- Should judges be reusable across a workspace, with optional agent association?
