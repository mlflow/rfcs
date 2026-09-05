# RFC 0009: Skill Tracing

| start_date   | 2026-08-23 |
| :----------- | :--------- |
| mlflow_issue | |
| rfc_pr       | https://github.com/mlflow/rfcs/pull/37 |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-08-27 |
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

This RFC links traces to skills. A skill activation produces a
**trace-level link** from the trace to the skill version, following
the pattern MLflow already uses to link prompts to traces. When the
activation is identifiable as a specific span, that span is also
annotated with the skill's registry coordinates: workspace,
organization, name, and version, plus the version's content digest
when the instrumentation has it. Tool calls that use a skill's
bundled files are annotated the same way. Each link records how it
was produced (explicit instrumentation, in-process resolution,
install-record matching, or content-marker inference), so consumers
of the linkage can weigh the evidence behind it.

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
- **Automatic instrumentation in harnesses.** When MLflow installs
  a skill into a harness such as Claude Code, it records which
  registry coordinates the harness-local skill came from. The harness
  autologger reads that record at run time to link and annotate.

Because traces carry skill links, they become queryable by skill. That
query surface is what turns tracing into governance evidence: adoption
tracking, impact analysis for deprecated or vulnerable versions, and
regression detection across skill versions.

**Relationship to other RFCs.** Skill tracing builds on
[RFC-0008: Skill Registry](https://github.com/mlflow/rfcs/blob/main/rfcs/0008-mvp-skill-registry/0008-mvp-skill-registry.md),
which defines the `Skill` and `SkillVersion` entities, the
`{workspace, organization, name, version}` coordinates, and the
client-asserted content `digest` that this RFC records on links.
Because the harness path depends on the install-time record described
above, harness installation commands and the package manager
integration behind them are in scope for this RFC (see
[Open questions](#open-questions) for the scope discussion). Skill
linking follows the same trace-level association pattern as prompt
linking and the MCP Server Registry's trace linking
([RFC-0004](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md)).
It adds activation-span annotation because skill activation, unlike
an MCP server association, is an observable event inside the trace.

# Basic example

TBD.

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

4. **Content identity and version identity diverge.** RFC-0008 re-mints
   a version on every import, so the same unchanged skill content can
   exist under many version numbers. Linking traces only to a specific
   `name/version` pair fragments the history of what is, by content, one
   skill. RFC-0008 already records a content `digest` on each version
   for exactly this grouping purpose, and traces should be able to use
   it.

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
   The method also accepts an `organization`; the examples here omit
   it and use the empty default organization, as RFC-0008's examples
   do.

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

2. **UI path:** open the trace in the MLflow UI. The trace view shows
   a "Skills" tab, alongside the existing linked-entity tabs, listing
   the linked skill versions; each links to the skill's registry
   detail page. Annotated activation spans show the skill coordinates
   in the span detail view. When recorded coordinates do not resolve
   (the version was deleted, or the trace came from a different
   workspace), the UI shows a "not found in registry" indicator.

#### Trace skills loaded by an agent framework

A developer using an agent framework (LangGraph, the OpenAI Agents
SDK, and similar) resolves a skill from the registry and hands it to
the framework. The linkage should not require explicit calls in
their code.

1. Resolve and pull the skill through MLflow, then hand it to the
   framework in whatever form the framework expects:
   ```python
   path = mlflow.genai.pull(
       name="code-review", alias="production", destination="./skills",
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

1. Install the skill into the harness through MLflow:
   ```bash
   mlflow skills install --skill-uri skills:/code-review@production \
       --harness claude-code
   ```
   MLflow resolves the alias to a concrete version, delegates the
   harness-specific install to the configured package manager plugin,
   and records the resulting harness-local skill name and location
   against the registry coordinates that were installed. The record
   maps each installed skill's harness-local name and location to its
   workspace, organization, name, version, and content digest.
   Recording this at install time is what makes the linkage survive a
   package manager that renames or prefixes the skill. Project-scoped
   installs record this in the project, and user-scoped installs
   record it in the user's MLflow configuration. When both define the
   same harness-local skill name, the project entry wins.
2. Enable tracing for the harness:
   ```bash
   mlflow autolog claude
   ```
3. Run the agent. The autologger identifies skill activations in the
   recorded conversation and, using the install record, links the
   trace to the skill version and annotates the activation span. It
   recognizes activations by the harness's own signals where they
   exist (a dedicated skill tool call, a slash-command invocation)
   and otherwise by observing a tool call that reads a skill's
   `SKILL.md` from its installed location. Tool calls that use a skill's bundled
   files, such as a script under the skill's directory, are annotated
   as skill usage by the same location matching. No registry call
   happens during the run, so there is no added latency and no
   runtime dependency on registry availability.
4. Open the MLflow UI and navigate to the Traces page. Linked skills
   appear on each trace, and annotated spans carry the coordinates.

A skill that was copied into the harness by hand, without an MLflow
install, has nothing recorded for it and produces no link.
Nothing about the run fails in that case: the agent runs normally,
other autologging is unaffected, and the developer can still link
explicitly. The same is true of a missing or unreadable install
record. Before linking, the autologger also validates the installed
content against the digest recorded at install time; on a mismatch
it logs an error and records no link, and the validation result is
cached locally so content is not re-hashed on every run.

This journey applies in full to harnesses whose tracing integration
is provided by MLflow. Some harnesses instead trace themselves
through native OpenTelemetry export, with MLflow as the receiver. The
receiving server cannot read an install record on the harness's
machine, so linkage on this path requires the coordinates to reach
the server inside the spans themselves. Two routes do that. The
first is the attribute contract shown in the first journey, set by
instrumentation on the host that can read the record. The second is
a best-effort fallback: installation appends a machine-readable
marker carrying the coordinates to the skill body, and the server
recognizes the marker inside captured LLM input at ingestion,
creating the link only when the marker's coordinates resolve in the
registry and its digest matches the version's. Marker-derived links
carry the content-marker provenance, and the route works only when
the harness captures LLM content in its spans, which several
harnesses leave off by default. Digest validation excludes the
marker line from hashing. MLflow does not create additional spans on
the harness's behalf.

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
prompt-to-trace links, and so does the storage behind it: a skill link
is a lineage record associating the trace with the skill version,
the same mechanism prompt links use, and the filter is an exact-match
query over those records. Span attributes mark where activation
happened but are not the query path. Links created at ingestion from
attributes or content markers produce the same lineage record, so
every provenance is reached by the one query. A skill in a named
organization is qualified as in the registry's URI form,
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

When the same content has been re-imported under several version
numbers, the affected traces span all of those versions. RFC-0008
indexes a skill's versions by content digest, so a registry lookup by
digest yields every version that shares the content, and the trace
query then covers those versions (see
[Open questions](#open-questions) on digest-based linking).

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
6. **UI path:** the experiment page shows a "Skills" tab listing the
   skill versions linked from the experiment's traces, following the
   existing prompts tab. When comparing evaluation runs, the
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

TBD.

# Detailed design

TBD.

# Drawbacks

TBD.

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

Other alternatives TBD.

# Adoption strategy

TBD.

# Open questions

- **OTel alignment.** The explicit journey shows a plain
  OpenTelemetry path that sets `mlflow.skill.*` attributes, from
  which MLflow records the link. That makes the attribute names part
  of the public contract rather than an implementation detail. Is
  that the right trade, and should the attribute names be namespaced
  differently if they are to be set by non-MLflow instrumentation?
- **Installation scope.** Automatic instrumentation for harnesses
  requires an install-time record connecting harness-local skills to
  registry coordinates, and this RFC currently brings harness
  installation commands and package manager integration into scope to
  produce it. That is a large undertaking in its own right, and it is
  worth asking whether this RFC should carry it. A narrower
  alternative would be a command that only activates tracing: the
  user installs the skill into the harness however they choose, then
  tells MLflow which registry coordinates the installed skill
  corresponds to, and MLflow writes the record without performing or
  managing the installation. That keeps the tracing linkage while
  leaving installation to the user, at the cost of an extra manual
  step and the risk that the declared coordinates do not match what
  was actually installed. Should this RFC include full installation,
  only the tracing-activation command, or the activation command now
  with installation as follow-on work?
- **Digest-based linking.** Digest queries resolve through the
  registry: RFC-0008 indexes a skill's versions by content digest, so
  grouping traces by content is a registry lookup followed by a trace
  query over the resulting versions. That index is scoped within a
  skill name, and the digest is client-asserted rather than
  server-verified. Should digest grouping also be supported across
  skill names, which would require a broader index?
