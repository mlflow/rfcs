# RFC 0012: Trace-Aware Webhook Events

| start_date   | 2026-07-28 |
| :----------- | :--------- |
| mlflow_issue | |
| rfc_pr       | |

| Author(s)              | [Nehanth Narendrula](https://github.com/Nehanth) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-08-26 |
| **AI Assistant(s)**    | Claude Code |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [User journey](#user-journey)
  - [Out of scope](#out-of-scope)
- [High-level design](#high-level-design)
- [Detailed design](#detailed-design)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [Open questions](#open-questions)

# Summary

MLflow's webhook system supports 15 event types, and all of them belong to the model registry, prompt registry, or AI Gateway budgets. The tracking side emits nothing. Automatic evaluations score production traces and issue detection clusters failures, but the results sit in the database until someone opens the UI.

This RFC adds webhook events for the observability side: an evaluation score crossing a user-defined threshold, issue detection creating or updating an issue, and a trace completing in an error state. The existing webhook API and delivery engine are reused without changes. The additions are the new event types, a filter configuration for thresholds, and the emission points.

A note on terminology: MLflow assessments come in two types — Feedback (quality evaluations from judges, humans, or code) and Expectations (ground-truth labels). Wherever this document says "feedback," it means Feedback produced by LLM judge scorers during automatic evaluations (`source_type=LLM_JUDGE`).

What consumers do with the events is up to them. The same event can drive a Slack message, a PagerDuty page, a CI run, or an automated remediation pipeline. A follow-up RFC (Agent Improvement Workflow) builds a guided improvement loop on top of these events, but this RFC stands on its own.

# Basic example

```python
from mlflow import MlflowClient

client = MlflowClient()

# Notify an endpoint when the rolling correctness average on sampled
# production traces drops below 0.7
client.create_webhook(
    name="prod-quality-alert",
    url="https://ops.example.com/mlflow/quality",
    events=["trace_feedback.threshold_breached"],
    filter_config={
        "experiment_ids": ["12345"],
        "feedback_name": "correctness",
        "threshold": 0.7,
        "direction": "below",
        "window_minutes": 5,
        "cooldown_seconds": 1800,
    },
    secret="my-signing-secret",
)
```

The payload follows the same envelope every existing webhook uses (`entity`, `action`, `timestamp`, `data`):

```json
{
    "entity": "trace_feedback",
    "action": "threshold_breached",
    "timestamp": "2026-08-04T15:30:00Z",
    "data": {
        "experiment_id": "12345",
        "trace_id": "tr-abc123",
        "feedback_name": "correctness",
        "value": 0.45,
        "window_average": 0.62,
        "threshold": 0.7,
        "window_minutes": 5
    }
}
```

# Motivation

## The problem

MLflow already runs the expensive half of production quality monitoring. Automatic evaluations execute LLM judges server-side on sampled live traces and write the results as feedback — assessments of type Feedback with source `LLM_JUDGE`. Issue detection analyzes traces and produces clustered issues with severity and root-cause summaries. Both answer the question "is my agent degrading, and why." Neither can tell anyone the answer.

This creates three problems:

1. **Quality regressions are discovered late.** A judge score can collapse at 2am, and the team finds out whenever someone next opens the experiment page. The time between "MLflow knew" and "a human knew" is unbounded.
2. **The evaluation spend is underused.** Users pay for judge inference on production traces, then poll a UI to see the results. The scores exist at the exact moment they are written; there is no way to act on them at that moment.
3. **No automation can be built on top.** Teams that want to react to quality signals — re-run a test suite, open a ticket, trigger an analysis — have nothing to hook into. The webhook system exists, but none of its events cover traces, feedback, or issues.

Alerting on production eval scores is also standard across the ecosystem — [LangSmith](https://docs.langchain.com/langsmith/alerts), [Langfuse](https://langfuse.com/docs/metrics/features/monitors), [Arize](https://arize.com/docs/ax/machine-learning/machine-learning/how-to-ml/monitors/configure-monitors/notifications-and-integrations), [Braintrust](https://www.braintrust.dev/docs/guides/automations), [Datadog LLM Observability](https://docs.datadoghq.com/llm_observability/monitoring/), and [Galileo](https://docs.galileo.ai/how-to-guides/basics/set-up-alerts-on-logs) all ship it — and MLflow has none of it.

## User journey

One incident, end to end. A team runs a support agent in production, traced to MLflow, with automatic evaluations scoring correctness on sampled traffic.

1. **Setup.** The team creates the quality webhook with a threshold rule (the basic example above) and an error webhook pointed at PagerDuty:

   ```python
   client.create_webhook(
       name="support-agent-errors",
       url="https://events.pagerduty.com/v2/enqueue",
       events=["trace.errored"],
       filter_config={"experiment_ids": ["12345"], "cooldown_seconds": 300},
   )
   ```

   **UI path:** the webhooks management page's event picker includes the new events, with a form for threshold rules instead of hand-written configuration — and the experiment page offers **Set up alert** whenever automatic evaluations are running, pre-filled with the registered judge's name and a default threshold.

2. **The quality signal.** At 2am, the agent's answers quietly get worse — nothing errors, nothing crashes. The team changed nothing: the provider silently updated the model behind the agent, and its behavior shifted. Requests keep succeeding, users start getting worse answers, and no infrastructure monitor anywhere has anything to say about it. The only thing in the stack that can notice is the correctness judge scoring the sampled traffic. As the scores come in lower, the windowed average crosses the threshold and the quality webhook fires. The team's endpoint routes it to Slack:

   ```
   MLflow quality alert
   Experiment: prod-support-agent
   correctness: 0.45 (5-minute average 0.62, threshold 0.70)
   View traces: https://mlflow.example.com/experiments/12345/traces
   ```

   The engineer clicks through to the low-scoring traces, sees the answers degrading, and starts investigating — hours before the first user complaint would have surfaced it. This is the alert that only evaluation-aware infrastructure can produce: a failure with no error anywhere.

3. **The hard failure.** Weeks later, the same provider deprecates the model identifier one of the agent's tools uses. This time runs fail outright, and each failed trace arrives at the server with state `ERROR`. The first one fires the error webhook:

   ```json
   {
       "entity": "trace",
       "action": "errored",
       "timestamp": "2026-08-05T02:14:11Z",
       "data": {
           "experiment_id": "12345",
           "trace_id": "tr-err456",
           "error": "The model 'gpt-4o-mini' does not exist or you do not have access to it."
       }
   }
   ```

   The on-call engineer is paged with a direct link to the failing execution. The cooldown means an agent failing on every request produces one page, not hundreds.

4. **The aftermath.** In the morning, the team runs issue detection over the night's traces. The run writes a new issue — "tool call failures in `summarize_ticket`, 34 traces affected, severity high" — and `trace_issue.created` fires into the CI webhook the team configured, which re-runs that tool's test suite and posts the result on the tracking ticket. A run that finds nothing sends nothing.

5. What happens after each signal is up to the team — a page, a Slack triage, a CI run, or an automated remediation pipeline. The webhooks' job was to make each signal available at the moment MLflow knew, instead of whenever someone next opened the UI.

The `window_minutes` option averages scores over a fixed window of time rather than reacting to a single score — judge scores on individual traces are noisy, and a fixed time unit behaves the same for busy and quiet agents alike. `cooldown_seconds` stops repeat firing while quality stays degraded. Both are optional; without them the rule fires on every score, which suits low-traffic experiments.

## Out of scope

- Remediation, fix generation, and improvement workflows.
- New scorer types or changes to how automatic evaluations and issue detection run.
- Changes to the 15 existing webhook event types or to the delivery engine.
- A full alerting product: alert entities with state and history, native Slack and PagerDuty integrations, monitors that run on a schedule. Every piece of one would sit on top of these events, so the events ship first.

# High-level design

## What triggers each event

The events being added, and where each one fires:

| Event | Fires from |
|---|---|
| `trace_feedback.threshold_breached` | The online scoring job (automatic evaluations), immediately after it writes a judge score — server-internal, not a public API endpoint. Scores logged manually through the feedback API do not trigger it. |
| `trace_issue.created` | Completion of an issue detection run that wrote a new issue — the event fires regardless of how the run was started. |
| `trace_issue.updated` | An existing issue's severity or lifecycle state changing. |
| `trace.errored` | The trace ingestion path, when a trace is persisted with state `ERROR`. |

Webhook events are validated against fixed entity and action lists, so this adds three entities (`trace`, `trace_feedback`, `trace_issue`) and two actions (`errored`, `threshold_breached`) to those lists. Note that only `trace.errored` fires from a public API endpoint; the other events fire from server-side jobs.

This proposal adds no background services: no scheduler, no monitor, no process watching the database. Each event is sent by code MLflow already runs, at the moment that code saves something — which is the same firing model the existing 15 events use.

**`trace_feedback.threshold_breached` fires when a judge score is saved and trips a rule.** Automatic evaluations already run a scoring job inside the server: it picks up new traces, runs the LLM judges on a sample, and saves each score. This RFC adds one step right after that save — check the webhook rules for this experiment and feedback name, and send the event if the score or its windowed average crosses the threshold. Alerts are therefore only as fresh as the scoring behind them: latency is the scoring cadence plus delivery. This also holds up as the job framework ([RFC-0002](https://github.com/mlflow/rfcs/blob/main/rfcs/0002-job-executor-plugins/0002-job-executor-plugins.md)) evolves — whether scoring runs inside the server process or, once the [Docker and Kubernetes executors](https://github.com/mlflow/rfcs/pull/3) land, in its own container, the score is saved back through the server, and that is where the rule check and the webhook send happen. Webhook secrets never leave the server.

**`trace_issue.created` and `trace_issue.updated` fire when a detection run saves its findings.** The events hang off the end of the run, not the button that started it — so when detection is eventually started by an API call or a schedule instead of a person, the same events fire with no changes.

**`trace.errored` fires when a failed trace is saved — and only then.** The trigger is the trace itself being persisted with state `ERROR`. A trace that completes `OK` with an errored span inside it — a tool call that failed but the agent recovered — does not fire the event: those recovered failures surface through the judge scores and issue detection's execution category instead, and firing on every span error would flood busy agents with events. A span-level error event can be revisited if demand appears.

An event that fires on every individual judge score is deliberately not included: it would fire once per judge per sampled trace, a volume profile unlike any existing webhook event, and dashboards that want a firehose are better served by querying the tracking API.

## What MLflow stores

The webhook entities themselves already exist; this proposal adds two small pieces of state, kept in the SQL backend the webhook system already requires and deleted with their webhook:

1. **When each rule last fired**, to enforce `cooldown_seconds` — used by both the threshold event and `trace.errored`.
2. **The recent scores within each rule's time window**, so the windowed average can be evaluated the moment a new score is written, without re-querying past feedback.

This is the first webhook feature that keeps state between deliveries. The footprint is one row per rule, plus one small score buffer per threshold rule.

## UI

MLflow's settings already include a webhooks management page ([added March 2026](https://github.com/mlflow/mlflow/pull/21483)) where webhooks are created, edited, and tested. This RFC extends it in two ways:

1. The event picker gains the four new events, and selecting the threshold event shows a form for the rule — judge, threshold, direction, window, cooldown — instead of hand-written configuration.
2. A new **Set up alert** entry point on the experiment page, shown when automatic evaluations are running, pre-filled with the registered judge and a default threshold — one step from "scoring is on" to "someone gets notified when scores drop."

Both additions are just forms that call the existing webhook API — **Set up alert** creates an ordinary webhook, the same one the Python example creates. No new alerting system is added behind the UI.

# Detailed design

TBD.

# Drawbacks

TBD.

# Alternatives

TBD.

# Adoption strategy

TBD.

# Open questions

1. Should alert rules be configured on each webhook (the smallest change, matching how webhooks work today), or become their own concept that a user defines once and points at several destinations — Slack and PagerDuty from one rule? Compound conditions push strongly toward the latter: an experiment runs multiple scorers, and a team may want to alert only when several conditions hold together or when any one of them does (AND/OR across conditions, each with its own aggregate and time window). That no longer fits naturally in a single webhook's filter, so the initial proposal keeps single-condition rules and leaves the rule primitive to the detailed design. Review feedback also points to an in-progress design for aggregating trace signals into fixed-unit metrics for alerting; combining with that work would put the rule layer there, with webhooks simply subscribing to the events it emits.
2. Should an alert also fire when data stops arriving entirely? An agent that produces no traces at all is arguably the worst failure, and a rule that only evaluates when scores are written stays silent through it. Some agents legitimately go quiet (a chat agent nobody is chatting with, a monitoring agent whose trigger condition has not occurred), so a blunt no-data alert would false-positive on them. Possible mechanisms: alerting on statistically unusual gaps relative to the agent's own trace cadence, or tracking agent-trigger events separately and alerting only when the agent triggers but no reasoning occurs — with the alert being opt-in for agents where quiet is normal.
3. Should the issue detection endpoints be stabilized and documented as part of this work, so consumers of the issue events have a supported way to look up the issues they reference?
