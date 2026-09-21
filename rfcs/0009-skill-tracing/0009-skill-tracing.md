# RFC 0009: Skill Tracing

| start_date   | 2026-08-23 |
| :----------- | :--------- |
| mlflow_issue | |
| rfc_pr       | https://github.com/mlflow/rfcs/pull/37 |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat), [Matthew Prahl](https://github.com/mprahl) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-09-11 |
| **AI Assistant(s)**    | Claude Code |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
  - [Link model](#link-model)
  - [SDK method and attribute contract](#sdk-method-and-attribute-contract)
  - [Queries](#queries)
  - [Skill identification](#skill-identification)
  - [Content marker](#content-marker)
  - [Autologger behavior](#autologger-behavior)
  - [UI](#ui)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)

# Summary

Skill tracing connects MLflow traces to the registered skills that
produced them, so that agent developers and platform owners can answer
questions like "which traces used this skill?", "who is still on the
deprecated version?", and "how did behavior change when this skill was
updated?"

MLflow already traces agent conversations across harnesses and agent
frameworks: Claude Code via `mlflow autolog claude`, SDK applications
via framework autologgers such as `mlflow.langchain.autolog()` and
`mlflow.anthropic.autolog()`, and others. Those traces capture LLM
calls, tool use, timing, and token consumption as a tree of spans. What
they do not capture is which governed, versioned skill was active during
any part of the run.

This RFC links traces to skills. A skill activation produces a **link**
from the trace to the skill version, recorded through the span on which
the activation was observed and following the pattern MLflow already
uses to link prompts to traces. The link records that span, and the span
is annotated with the skill's registry coordinates: workspace,
organization, name, and version. Tool calls that use a skill's bundled
files are annotated the same way.

Two terms recur below. A **harness** is a packaged agent application
that skills are installed into and that runs them without code written
by the user: Claude Code, Codex, Gemini CLI, OpenCode, Goose, and
OpenHands are examples. An **agent framework** is a library that
application code uses to build an agent; the developer's own process
loads the skill and drives the run: LangGraph,
Google ADK, the OpenAI Agents SDK, CrewAI, Pydantic AI, and Semantic
Kernel are examples.

Links are recorded along three paths:

- **Explicit instrumentation.** Application code that composes its own
  agent records the link where it activates a skill. This is the path
  for custom agents built on the MLflow SDK, and it is also
  expressible through plain OpenTelemetry for callers that are not
  using the MLflow tracing API directly.
- **Automatic instrumentation in agent frameworks.** When
  application code resolves a skill from the registry and hands it to
  a framework such as LangGraph or the OpenAI Agents SDK, MLflow holds
  the mapping from that skill to its registry coordinates in process,
  and the framework autologger records the link and annotates the
  activation span without the developer writing tracing code.
  Frameworks that MLflow traces by receiving their native
  OpenTelemetry output rather than by running instrumentation inside
  them are handled like the equivalent harnesses (see the harness
  journey).
- **Automatic instrumentation in harnesses.** When content pulled
  through MLflow is installed into a harness such as Claude Code, the
  harness autologger identifies it by content: it hashes the
  installed skill with RFC-0008's digest rule and checks the result
  against the marker that `mlflow skills pull` wrote into the skill,
  then links and annotates without contacting the registry. Nothing
  is declared or configured.

Because traces carry skill links, they become queryable by skill. That
query surface is what turns tracing into governance evidence: adoption
tracking, impact analysis for deprecated or vulnerable versions, and
regression detection across skill versions.

**Relationship to other RFCs.** Skill tracing builds on
[RFC-0008: Skill Registry](https://github.com/mlflow/rfcs/blob/main/rfcs/0008-mvp-skill-registry/0008-mvp-skill-registry.md),
which defines the `Skill` and `SkillVersion` entities and the
`{workspace, organization, name, version}` coordinates. The harness
path identifies installed skills by the marker and digest
that `mlflow skills pull` gives their content, so this RFC specifies
no installer, install record, or package manager integration; pulled
content reaches a harness by whatever means the user chooses. Skill
linking follows the same lineage-record pattern as prompt linking
and the MCP Server Registry's trace linking
([RFC-0004](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md)).
It adds activation-span annotation because skill activation, unlike
an MCP server association, is an observable event inside the trace.

# Basic example

```python
import mlflow

with mlflow.start_span("review") as span:
    span.link_skill(name="code-review", version=1)
    result = llm.chat([{"role": "user", "content": "Review this..."}])

traces = mlflow.search_traces(
    locations=[experiment_id],
    filter_string="skill = 'code-review/1'",
)
```

The first call links the span's trace to version 1 of the registered
skill and annotates the span as the activation point. The query
returns every trace in the experiment linked to that version.

## Motivation

### The problem

Enterprises adopting the skill registry gain governance over skill
content: versions, status lifecycle, aliases, and discovery. What they
do not gain is any evidence about what happens when those skills run.
The registry knows what was published; the traces know what the agent
did; nothing connects the two.

1. **Traces do not say which skill was active.** A trace shows LLM
   calls, tool use, and token counts, but there is no way to tell
   whether the agent was operating under `code-review` version 1,
   `code-review` version 2, or no registered skill at all. Every
   question that starts with "for runs that used this skill" is
   unanswerable today.

2. **Harness-local names are not registry identity.** A skill installed
   into a harness may be renamed, prefixed, or namespaced by the package
   manager that installed it. Even when a harness emits the skill name
   it loaded, that string does not identify a registered version, and it
   carries no workspace or version. Matching on it heuristically after
   the fact is guesswork.

3. **Governance decisions have no evidence base.** Deprecating a version
   means telling consumers to move, but there is no way to see who is
   still on it. A security finding against a skill means asking which
   runs were exposed, and nothing can answer that. Promoting a version
   means asserting it is better, but there is no way to attribute a
   quality difference to the skill rather than to everything else that
   changed. Retiring an unused skill means knowing it is unused.

### User journeys

These journeys illustrate the end-to-end workflows that skill tracing
enables. They cover the three instrumentation paths and the analysis
workflows the resulting links support.

#### Instrument a custom agent application

A developer building an agent directly on the MLflow SDK wants the
traces it produces to record which registered skill was active.

1. Record the link on the span where the skill is activated:
   ```python
   import mlflow

   with mlflow.start_span("review") as span:
       span.link_skill(name="code-review", version=1)
       ...
   ```
   The method is on the span object, so the target is explicit: it
   annotates that span as the activation point and links the span's
   trace to the skill version. Code without a span in hand uses
   `mlflow.get_current_active_span()`. Like the rest of the registry
   surface, the method also accepts an alias:
   ```python
   span.link_skill(name="code-review", alias="production")
   ```
   The alias is resolved when the link is recorded and the trace
   stores the concrete version, consistent with RFC-0008's rule that
   aliases are accepted as input but never stored in place of
   versions. Aliases move; a trace that stored `@production` would
   become ambiguous the moment the alias was repointed. Alias
   resolutions are cached per process with a short expiry, following
   the prompt registry's alias cache, so repeated links in a hot path
   do not query the registry on every call; a caller can shorten or
   disable the cache per call. Pinned versions need no resolution.
   The method also accepts `organization` and `workspace`; the examples
   use the empty organization and current workspace.

   **Without the MLflow SDK:** a caller instrumenting with plain
   OpenTelemetry has no `link_skill` method and instead sets these
   attributes on the activation span. MLflow recognizes the
   `mlflow.skill.*` attributes as a skill activation when it receives
   the span and records the link from them:
   ```python
   span.set_attribute("mlflow.skill.name", "code-review")
   span.set_attribute("mlflow.skill.version", 1)
   span.set_attribute("mlflow.skill.workspace", "default")
   span.set_attribute("mlflow.skill.organization", "")
   ```

2. **UI path:** open the trace in the MLflow UI. The trace view lists
   the linked skill versions among the trace's linked entities, each
   linking to the skill's registry detail page; how linked entities
   are presented (for example, one consolidated lineage view across
   prompts, skills, and other assets) is aligned across assets outside
   this RFC. Annotated activation spans show the skill coordinates
   in the span detail view. When recorded coordinates do not resolve
   (the version was deleted, or the trace came from a different
   workspace), the UI shows a "not found in registry" indicator.

#### Trace skills loaded by an agent framework

A developer using an agent framework (e.g. LangGraph) resolves a skill from
the registry and hands it to the framework. The linkage should not require
explicit calls in their code.

1. Resolve and pull the skill through MLflow, then hand it to the
   framework in whatever form the framework expects:
   ```python
   path = mlflow.genai.pull(
       "skills:/code-review@production", destination="./skills",
   )
   agent = build_agent(skills=[path])
   ```
2. Enable the framework autologger as usual. No explicit call is
   needed: because the skill was resolved through MLflow, the
   autologger knows its registry coordinates, links the trace, and
   annotates the span where the framework activates the skill.
3. Run the agent and open the trace. The result is the same as the
   explicit path: the trace is linked to the skill version, and the
   activation is annotated where it happened.

This follows a pattern MLflow already uses for other registry
entities: resolving an entity records its identity, and tracing picks
that identity up automatically. Skill activation is observable in
current frameworks, most directly where loading a skill is itself a
tool invocation. The per-framework mechanisms belong in the
detailed design.

#### Trace skills installed into a harness

A platform owner installs skills into a harness such as Claude Code,
where there is no application code to instrument and no in-process
resolution step.

1. Pull the skill with `mlflow skills pull` and install the pulled
   content into the harness by any means: the harness's own
   installer, a package manager, or a plain copy. The pulled content
   carries a marker with its registry coordinates, and the marker
   travels with the content wherever it is copied. MLflow is not
   involved in the installation and nothing is declared to it.
2. Enable tracing for the harness:
   ```bash
   mlflow autolog claude
   ```
3. Run the agent. The autologger identifies skill activations in the
   recorded conversation by the harness's own signals where they
   exist (a dedicated skill tool call, a slash-command invocation)
   and otherwise by observing a tool call that reads a `SKILL.md`.
   For each activated skill it hashes the installed skill directory
   with RFC-0008's digest rule and compares the result with the
   digest in the skill's marker; when they match, the marker's
   coordinates identify the version (the result is cached locally
   and no registry call is made), and the autologger links the trace
   to that version and annotates the activation span. Tool calls
   that use a skill's bundled files, such as a script under the
   skill's directory, are annotated as skill usage by the same
   location matching.
4. Open the MLflow UI and navigate to the Traces page. Linked skills
   appear on each trace, and annotated spans carry the coordinates.

A skill that carries no marker, because it was installed from a
source other than `mlflow skills pull`, is not linked. A skill whose
content no longer matches its marker, because it
was edited after installation, is not linked either. In both cases
the agent runs normally and other autologging is unaffected. Because
identity comes from the marker and the content, an installer that
renames or prefixes the skill does not break the linkage, and a skill
installed under one name is recognized however it was named.

This journey applies in full to harnesses whose tracing integration
is provided by MLflow. Some harnesses instead trace themselves
through native OpenTelemetry export, with MLflow as the receiver. The
receiving server cannot see the harness's filesystem, so linkage on
this path requires the coordinates to reach the server inside the
spans themselves. Two routes do that. The first is the attribute
contract shown in the first journey, set by instrumentation on the
host. The second is a best-effort fallback: `mlflow skills pull`
appends a machine-readable marker carrying the coordinates to the
pulled skill body, and the server recognizes the marker inside
captured LLM input at ingestion, creating the link only when the
marker is well formed. The server preserves its coordinates without
resolving the registry version. This route covers only content
that was pulled through MLflow, and it works only when the harness
captures LLM content in its spans, which several harnesses leave off by
default.
Digest verification removes the marker suffix before hashing. MLflow
does not create additional spans on the harness's behalf.

#### Measure adoption of a registered skill

A platform owner wants to know whether a skill is being used, and which
versions are in play.

1. Query traces linked to the skill:
   ```python
   traces = mlflow.search_traces(
       locations=[experiment_id],
       filter_string="skill = 'code-review/1'",
   )
   ```
   Leaving the version off lists traces across all versions of the
   skill and shows the version spread.
2. **UI path:** open the skill's registry detail page. A "Related
   traces" link opens the Traces page filtered to that skill, and the
   version detail page does the same for a single version. The Traces
   page shows linked skills on each row.

The filter follows the precedent of the existing `prompt` filter for
prompt-to-trace links, but reads the dedicated skill-link table. Span
attributes mark where activation happened but are not the query path.
Links created at ingestion from attributes or content markers produce
the same records. A skill in a named organization is qualified as in
the registry's URI form,
`skill = '@acme/code-review/1'`. The name-only and
organization-qualified forms are extensions this RFC proposes;
the prompt filter accepts only the `name/version` form. Exact
matching matters here; substring matching over trace
content would match `code-review` inside `code-review-strict` and
version `1` inside version `10`, making an adoption count wrong
rather than approximate.

#### Assess the impact of a deprecated or vulnerable skill version

A skill owner is about to deprecate a version and needs to know who is
still on it. A security engineer has learned a skill version is
susceptible to prompt injection and needs to know which runs were
exposed. Both are the same query: find the traces linked to an
affected version.

1. Find recent traces linked to the affected version:
   ```python
   traces = mlflow.search_traces(
       locations=experiment_ids,
       filter_string="skill = 'code-review/1'",
       order_by=["timestamp_ms DESC"],
   )
   ```
   The query names the locations to search; the result is only as
   complete as the locations the organization traces into.
2. Group the results by experiment to see which teams and applications
   produced them. For the security case, this is the exposure set:
   the runs in which the vulnerable version was active.
3. Notify those consumers, then transition the version:
   ```bash
   mlflow skills update-version skills:/code-review/1 --status deprecated
   ```
4. Re-run the query after the migration window to confirm that traffic
   on the affected version has stopped.

The evidence is retrospective: it shows what has run, not what is
installed and idle. A consumer that has the version installed but has
not exercised it since tracing was enabled does not appear.

#### Evaluate and compare skill versions on a benchmark

A team updates a skill and wants to know whether agent quality or cost
changed, connecting evaluation results to the skill versions that were
active.

1. Register the updated content, producing a new version:
   ```bash
   mlflow skills register git --name code-review \
       --url https://github.com/acme/agent-skills.git \
       --ref v2.0.0 --subpath code-review
   ```
2. Run the benchmark suite before and after the change. Both sets of
   traces carry skill links, so which version was active in each run
   is recorded rather than inferred from when the run happened.
3. Run evaluation against the collected traces:
   ```python
   results = mlflow.genai.evaluate(
       data=traces_df,
       scorers=[correctness_scorer, helpfulness_scorer],
   )
   ```
   Each row in `results.result_df` includes a `trace_id`. Reading a
   trace's linked skills connects any evaluation result back to the
   skill versions that were active in its run, and querying traces by
   skill (as in the adoption journey) walks the same lineage in the
   other direction.
4. Compare the two versions' runs on the same scorers, including cost
   per run: the linked traces carry their usual token and cost
   metrics, and because the benchmark holds the workload constant,
   the skill version is the variable.
5. If the new version is an improvement, promote it:
   ```bash
   mlflow skills set-alias skills:/code-review \
       --alias production --version 2
   ```
6. **UI path:** the experiment page lists the skill versions linked
   from the experiment's traces alongside its other linked entities,
   such as prompts. When comparing evaluation runs, the
   comparison view shows a diff of the linked skill versions, so a
   quality change can be read against exactly what changed in the
   skill configuration.

#### Compare skill versions on production traffic

A team promotes an updated skill version to production and wants to
know whether success metrics changed, using the traffic the agent
already serves rather than a benchmark run.

1. Promote the new version:
   ```bash
   mlflow skills set-alias skills:/code-review \
       --alias production --version 2
   ```
   Traces recorded after the promotion link to version 2; earlier
   traces link to version 1. The links partition traffic by the
   version that actually ran within the application's single
   production location, which stays exact even when a rollout is
   gradual or both versions serve concurrently, where a
   split-by-deploy-time would misattribute runs.
2. After enough traffic accumulates, retrieve each version's traces
   from the production location with the adoption journey's query,
   one query per version.
3. Score each version's traces with the same scorers, producing one
   evaluation run per version in the same experiment, or compare
   assessments already collected on the production traces.
4. Compare the two evaluation runs side by side. The comparison view
   shows the diff of linked skill versions alongside the metric
   deltas, so the change in outcomes is read against the change in
   skill configuration.

Production comparison trades rigor for reach. The samples are
unpaired, so statistical significance requires more data than a
paired benchmark comparison; production traffic is what supplies that
volume. The input mix can also shift between the two periods, and
that risk is accepted as a cost of evaluating on production data.
The benchmark journey above is the controlled complement.

### Out of scope

- **Installing skills into harnesses.** This RFC identifies installed
  skills by content; it neither performs nor records installation. An
  MLflow installer with package manager integration could follow
  later without changing anything here.
- **Skill-level cost attribution.** Without a delimited region for a
  skill's influence, tokens cannot be attributed to a skill. Cost is
  compared per run across skill versions in the evaluation journeys.
- **Tracing of non-skill plugin members.** Linking traces to agents,
  hooks, and other agent plugin members is deferred to
  [RFC-0010: Extended Agent Plugins](https://github.com/mlflow/rfcs/pull/27),
  which reuses the mechanism defined here.
- **Filtering evaluation results by skill directly.** Evaluation
  results reach skill versions through their traces' links; a direct
  filter on evaluation results is not added.
# Detailed design

This section is deliberately high level. It fixes the shapes that
other work depends on (the link, the SDK method, the attribute
contract, skill identification, the query forms) and leaves internals
to the implementation.

## Link model

A skill link is a lineage record associating a trace with a skill
version. Prompt links store a composite `name/version` destination in
the generic `entity_associations` table. Skills instead use a dedicated
span-link table because each observation carries additional metadata.
Trace queries use a correlated `EXISTS` against this table, so multiple
observing spans still return each trace once. The SQL is illustrative;
the Alembic migration emits the equivalent syntax for each supported
database.

```sql
CREATE TABLE span_skill_links (
  trace_id VARCHAR(50) NOT NULL,
  span_id VARCHAR(50) NOT NULL,
  skill_workspace VARCHAR(63) NOT NULL,
  skill_organization VARCHAR(64) NOT NULL DEFAULT '',
  skill_name VARCHAR(128) NOT NULL,
  skill_version INTEGER NOT NULL,
  role VARCHAR(20) NOT NULL,
  created_at BIGINT NOT NULL,
  PRIMARY KEY (
    trace_id, span_id, skill_workspace, skill_organization,
    skill_name, skill_version
  ),
  FOREIGN KEY (trace_id) REFERENCES trace_info(request_id) ON DELETE CASCADE
);

CREATE INDEX ix_span_skill_links_skill
  ON span_skill_links (
    skill_workspace, skill_organization, skill_name,
    skill_version, trace_id
  );
```

`span_skill_links` stores one row for each span where a skill was
observed:

| Field | Value | Meaning |
|---|---|---|
| `role` | `activation` | The span activated the skill. |
| `role` | `usage` | The span used a file bundled with the skill. |

Storage behavior:

- **Foreign keys:** only `trace_id` is constrained. Spans may be stored
  outside the relational tracking store, and skill links must survive
  registry deletion.
- **Lifecycle:** skill moves and deletions do not rewrite links; their
  coordinates remain historical facts.
- **Access:** reading a link requires access to its trace. Resolving or
  navigating to the referenced skill separately requires skill access.
- **Indexes:** the primary key supports lookup by trace. The reverse
  index supports name-only and exact-version trace queries.

Span-link inserts are idempotent. The span is also annotated with the
`mlflow.skill.*` attributes in the
[SDK method and attribute contract](#sdk-method-and-attribute-contract)
for inspection in the trace view.

Links are materialized during span ingestion; there is no separate
public linking endpoint. After the trace row exists, the server passes
the links found in each ingestion batch to an internal
`upsert_span_skill_links(trace_id, links)` tracking-store method. The
method atomically upserts the span links. It requires the trace to
exist, but not a relational span row or a resolvable registry version.

## SDK method and attribute contract

```python
class Span:
    def link_skill(
        self,
        *,
        name: str,
        version: int | None = None,
        alias: str | None = None,
        workspace: str | None = None,
        organization: str = "",
        cache_ttl_seconds: float | None = None,
    ) -> None: ...
```

Exactly one of `version` and `alias` is given. An alias is resolved
through the registry and the concrete version is what the link
records. Alias resolutions are cached per process with a short
expiry, following the prompt registry's alias cache: the default TTL
comes from `MLFLOW_ALIAS_SKILL_CACHE_TTL_SECONDS` (60 seconds), a
per-call `cache_ttl_seconds` overrides it, and `0` bypasses the cache.
`workspace=None` uses the caller's current workspace; an explicit
workspace permits cross-workspace links and scopes alias resolution.
The method only adds the attributes below to the span. It makes no
separate linking or export request; the attributes follow the existing
span export flow, and ingestion materializes the corresponding trace
and span links.

Instrumentation that does not use the MLflow SDK sets the same
attributes directly, and MLflow creates the lineage record from them
when it receives the span:

| Attribute | Value |
|---|---|
| `mlflow.skill.name` | registered skill name |
| `mlflow.skill.version` | registered version (integer) |
| `mlflow.skill.workspace` | workspace name |
| `mlflow.skill.organization` | organization, empty string when none |
| `mlflow.skill.role` | `activation` (default) or `usage` |

These names are a public contract. No OpenTelemetry semantic
convention for skills exists today; if one is defined later, MLflow
can map it to these attributes without changing the contract.

## Queries

The `skill` filter on `search_traces` accepts only `=` and uses the
form `[@organization/]name[/version]`. Omitting the version matches all
versions of the skill. `skill.workspace` accepts only `=`, must
accompany `skill`, and defaults to the caller's active workspace. Both
predicates match the same link. Aliases are not accepted: aliases
passed to `link_skill` are resolved when the link is created, and
traces store the concrete version.

- `skill = 'code-review'`: all versions of a skill;
- `skill = 'code-review/1'`: one version;
- `skill = '@acme/code-review'`: all versions in a named organization;
- `skill = '@acme/code-review/1'`: one version in a named organization;
- `skill.workspace = 'shared' AND skill = 'code-review/1'`: a skill in
  another workspace.

## Skill identification

Autologgers identify an installed skill by its content, without
contacting the registry. On the first activation of a skill in each
trace, the autologger hashes the skill directory with RFC-0008's
canonical digest rule and checks the result against the content marker
defined below:

1. if the installed `SKILL.md` carries the marker and the marker's
   digest equals the computed digest, the marker's coordinates
   identify the version, and the autologger links and annotates;
2. if the marker is absent, the content is not linked;
3. if the marker is present but its digest differs from the computed
   digest, the content was modified after it was pulled; the
   autologger logs a warning and treats the content as unmarked.

Only content pulled through MLflow is linked on this path, so a
harness run never depends on registry availability.

The verification result is cached for the remainder of the trace. A
new trace verifies the content again, so no file watcher or persistent
cache is required.

The harness integrations must implement the digest rule identically
to the Python SDK; RFC-0008 defines the rule deterministically, and
shared test vectors guard the implementations.

## Content marker

`mlflow skills pull` appends a fixed suffix to the pulled `SKILL.md`:
two newline bytes, one marker line, and a final newline byte.

```
<!-- mlflow-skill: {"schema_version":1,"workspace":"default","organization":"","name":"code-review","version":3,"digest":"<64 lowercase hex>"} -->
```

`schema_version` versions the marker format; parsers accept `1` and
ignore unsupported versions. The JSON carries the registry coordinates
and digest computed by `pull`, in RFC-0008's raw, lowercase hexadecimal
form. Before applying RFC-0008's digest algorithm, verification removes
exactly one well-formed final marker suffix; malformed or non-final
markers remain part of the content. At OpenTelemetry ingestion, the
server scans captured LLM input (never output) for the marker and
creates links from each well-formed marker without resolving its
coordinates against the registry. Duplicate matches produce
one link per observing span. In testing against OpenHands and Goose,
the marker survived byte-for-byte wherever the harness captured LLM
content in its spans; whether content is captured at all, often an
opt-in, is the limiting factor.

## Autologger behavior

The MVP integrations are LangGraph, Claude Code, and Codex. Local
detectors treat an explicit skill event or a structured tool read of a
skill root's `SKILL.md` as activation, and structured tool access to
another path under that root, such as executing
`<skill-root>/scripts/check.py`, as usage. They do not inspect
arbitrary text or model output.

**Claude Code.** The autologger recognizes dedicated skill tool calls,
slash-command invocations, and reads of `SKILL.md`. It identifies the
skill by verifying its marker and digest. Tool calls whose inputs
reference paths under the skill directory are usage. Unmarked or
modified content produces no link.

**LangGraph.** `mlflow.genai.pull` accepts a skill URI
(`skills:/name@alias` or `skills:/name/version`) and records the
canonical absolute skill root, coordinates, and digest in a
thread-safe process-local map. The map identifies candidates; the
first activation in each trace is verified as above. Path matching uses
the most specific mapped root, and copied or moved content falls back
to its marker.

**OpenTelemetry receivers (Codex in the MVP).** A span from any
OpenTelemetry producer creates links when it carries workspace, name,
and integer version attributes; organization defaults to empty, role
to `activation`. Incomplete or malformed attributes are ignored with a
warning and do not reject the trace.
As a fallback, the server scans captured LLM input only, accepts each
well-formed marker without a registry lookup, creates separate links
for distinct markers, and ignores duplicates. Codex validates this
generic receiver path in the MVP; its marker fallback is best-effort
because content capture is opt-in.

## UI

Both the content and the presentation of these surfaces will be
decided with a designer and a prototype outside this RFC, alongside
the same decisions for prompts and other linked assets. The list
below is an initial draft of the content the journeys need, not a
UI specification.

- Trace view: linked skill versions listed among the trace's linked
  entities, each linking to its registry detail page; annotated spans
  show the coordinates in the span detail; unresolvable coordinates
  show a "not found in registry" indicator.
- Skill and version detail pages: a "Related traces" link opening
  the Traces page filtered by that skill or version.
- Experiment page: skill versions linked from the experiment's traces
  listed alongside its other linked entities.
- Run comparison: a diff of linked skill versions alongside the
  metric deltas.

# Drawbacks

- **Harness linking covers only content pulled through MLflow.** A
  skill installed from another source carries no marker and is not
  linked. The digest rule must be implemented identically in every
  harness integration.
- **The marker mutates pulled content.** Appending a marker to
  `SKILL.md` changes the file on disk and requires the digest rule to
  exclude the fixed marker suffix. It is also visible to the model,
  and the server-side fallback works only when the harness includes
  LLM message content in its telemetry, which is a separate setting
  from tracing itself and is off by default in the OpenTelemetry GenAI
  conventions. In practice this limits little: users who send traces
  to MLflow to understand agent behavior generally enable content
  capture as well, since prompts and responses are most of what they
  want to see.
- **Alias links can lag a repoint** for up to the cache TTL, the same
  trade-off prompt aliases make.
- **A public attribute contract** commits MLflow to the
  `mlflow.skill.*` names for as long as non-MLflow instrumentation
  writes them.

# Alternatives

## A SKILL span that parents the work

Modeling skill activation as a `SKILL` span whose children are the
LLM and tool spans produced while the skill is active was considered
and rejected. A span asserts an operation with a meaningful start and
end. Skill activation is broadly observable, but the end of a
skill's influence frequently is not: the Agent Skills specification
defines activation as a one-way load with no counterpart, and
harnesses commonly keep the loaded content in context for the rest
of the session. Parenting spans under a SKILL span therefore asserts
a containment the instrumentation cannot accurately record, and it
cannot represent concurrent activations.

## Attribute-based search instead of lineage records

Querying traces by span attributes was considered and rejected. Span
attributes are stored inside serialized span content, and the
existing attribute filter is substring matching over that content,
which cannot give the exact-match semantics an adoption or exposure
count needs; structured attribute search would be new store work for
a worse result than the lineage mechanism prompt links already use.

## A module-level linking call

A module-level `mlflow.genai.link_skill()` acting on the current
trace and span was considered and rejected in favor of the span
method, because a module-level call in the `genai` namespace does not
say whether it links a trace, a span, or an active evaluation run.

## An install record written by the user

Recording each installed skill's registry coordinates in a local file,
written by an MLflow installer or by a tracking command, was
considered and rejected. A record makes sense as a byproduct of an
installation MLflow performs, and this RFC performs none; asking users
to declare installed skills by hand is a step they can forget, and a
declared record can drift from the content it describes. Identifying
installed content by its marker and digest needs no declaration and
cannot drift.

# Adoption strategy

New feature, not a breaking change. Existing traces are unaffected.
This RFC delivers `Span.link_skill`, the attribute contract, lineage
records and the `skill` filter, the content marker written by `pull`,
and automatic recognition in LangGraph, Claude Code, and Codex. OpenCode
is the first follow-up integration; other harnesses and frameworks are
deferred. RFC-0010 reuses the link model for non-skill plugin members.
