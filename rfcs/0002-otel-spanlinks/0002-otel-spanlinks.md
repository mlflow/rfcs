start_date:    2026-03-24
mlflow_issue:  [21025](https://github.com/mlflow/mlflow/issues/21025)
rfc_pr:        # leave this empty
author(s): Khaled Sulayman (ksulayma@redhat.com)

# Summary

[Span Links](https://opentelemetry.io/docs/specs/otel/trace/api/#link) are an official OpenTelemetry API primitive that allow for linking spans together that may not have a direct parent-child relationship and come from separate traces.

This RFC proposes adding support for SpanLinks in MLFlow.

# Basic example

Consider an asynchronous multi-agent application deployed internally at a fictional company called TechSolutions who want to stay up-to-date on current public sentiment about their software offerings:

**Agent A (Social Monitoring)** - Runs hourly, trace ID: `tr-monitoring-*`
- Scrapes X, LinkedIn, HackerNews for mentions of TechSolutions
- Publishes links to some shared storage (e.g. queue, volume, message bus)
- Each run creates a new, independent trace

**Agent B (Summarization)** - Runs daily, trace ID: `tr-summarizer-*`
- Pulls mentions reported by Agent A from shared storage
- Summarizes each mention and adds to a running draft in shared storage
- Each processing creates a new trace

**Agent C (Distribution)** - Runs weekly, trace ID: `tr-distribution-*`
- Reads the accumulated draft from shared storage
- Proofreads and sends weekly company newsletter
- Creates a single trace per week

These agents cannot use parent-child relationships because:
1. **Different trace IDs** - Each agent creates independent traces
2. **Asynchronous execution** - Agent B doesn't "call" Agent A; it reads from shared storage
3. **Time separation** - Agent C runs days after Agent A's traces completed
4. **Many-to-many relationships** - One Agent C trace processes data from multiple Agent B traces

Adding SpanLinks allows users to correlate related (but not parent/child/sibling) Spans for more thorough tracing.

Currently, MLFlow neither provides a way to add these links upon Span instantiation or afterwards, nor does it import them during ingestion via OTLP:

```python
# mlflow/entities/span.py [418-446]
otel_span = OTelReadableSpan(
    name=otel_proto_span.name,
    context=build_otel_context(trace_id, span_id),
    parent=build_otel_context(trace_id, parent_id) if parent_id else None,
    start_time=otel_proto_span.start_time_unix_nano,
    end_time=otel_proto_span.end_time_unix_nano,
    ...
    attributes={
        ...
    },
    status=OTelStatus(status_code, otel_proto_span.status.message or None),
    events=[
        ...
    ],
    resource=_OTelResource.get_empty(),
)
```

## Motivation

In addition to the gaps described above, the motivation follows by way of Links being an OTel primitive, and full compatibility requires supporting them.

### Out of scope

- This RFE does not entail automatic or implicit span linking e.g. via `mlflow.autolog()`
- This RFE does not propose visualizations for links in the timeline tree or graph view of the UI

## Detailed design

### Requirements
- Span Links are preserved during OTLP ingestion
- New 'Links' column in Span table
- Span Links can be created via MLFlow Python API
- Span Links added to Span serialization format without breaking backward compatibility
- Span Links displayed among span metadata MLFlow's UI
- Does not break existing functionality

### OpenTelemetry Span Link Spec

An OpenTelemetry Span Link has the following properties:
- The [SpanContext](https://opentelemetry.io/docs/specs/otel/overview/#spancontext) of the Span it links to (will elaborate on this in serialization section)
- Zero or more additional [Attributes](https://opentelemetry.io/docs/specs/otel/common/#attribute) (note: these are 'link attributes', which is not the same as "the attributes of the span being linked to")

In MLFlow, the existing semantics around child-to-parent Span linking use Span IDs, and contexts are built around that, for example:

```python
# mlflow/entities/span.py:328

otel_span = OTelReadableSpan(
    ...
    parent=build_otel_context(otel_trace_id, parent_id) if parent_id else None,
    ...
)
```

However, because linked spans may come from other traces, simply linking by Span ID is not sufficient.

### Python API

First and foremost, we add a minimal helper class for creating span links:

```python
@dataclass
class Link:
    """
    Helper class for creating span links.

    Args:
        trace_id: The MLflow trace ID of the linked span
        span_id: The span ID within that trace
        attributes: Optional attributes describing the link relationship
    """
    trace_id: str
    span_id: str
    attributes: Optional[dict[str, Any]] = None

    # Internal: cached OTel SpanContext (built in __post_init__)
    _context: SpanContext = field(init=False, repr=False, compare=False)

    def __post_init__(self):
        """Build and cache the OTel SpanContext once at creation time."""
        ...
        # Build context once and cache it
        self._context = build_otel_context(
            otel_trace_id,
            otel_span_id,
        )

    def to_otel_link(self):
        """Convert to OpenTelemetry Link object using cached context."""
        ...

    def to_dict(self) -> dict[str, Any]:
        """Serialize to dictionary (for database storage)."""
        return {
            "trace_id": self.trace_id,
            "span_id": self.span_id,
            "attributes": self.attributes,
        }

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> "Link":
        """Deserialize from dictionary (rebuilds _context in __post_init__)."""
        return cls(
            trace_id=data["trace_id"],
            span_id=data["span_id"],
            attributes=data.get("attributes"),
        )
```

This allows for correlating trace and span IDs to correctly identify spans (and construct them internally to the class with SpanContexts) without bubbling up OTel logic to the user or requiring them to import OTel types.

The User-facing API should allow for the adding of links during span instantiation:

```python
span_a_links = []
with mlflow.start_span("agent_a", trace_id="tr-monitoring-1234") as span_a:
    public_sentiment = do_research()
    link = Link(trace_id="tr-monitoring-1234", span_a.span_id)
    span_a_links.append(link)  # Save for linking

# Later, in a separate process/trace
with mlflow.start_span("agent_b", trace_id="tr-summarizer-5678", links=[span_a_links]) as span_b:
    draft_snippet = summarize(research_output)
    running_draft.append(draft_snippet)
...
```

Below is a non-exhaustive list of different "span starting" entrypoints we would need to accomodate adding links from:
- `provider.start_span_in_context()`
- `provider.start_detached_span()`
- `fluent.start_span()`
- `fluent.start_span_no_context()`
- `MLFlowClient.start_span()`
- `mlflow/libs/typescript/core/src/core/api.ts`
    - `startSpan(options)`
    - `withSpan()`

Some other areas to consider are:
- `MLFlowClient.start_trace()`
- `@mlflow.trace` decorator in `fluent.py`:
    - `_wrap_function()` (uses `start_span()`)
    - `_wrap_generator()` (uses `start_span_no_context()`)

Additionally, spans can be added after instantiation via a new `LiveSpan.add_link()` method:

```python
for link in span_a_links:
    span_b.add_link(link)
```

Additionally would need changes to `from_otel_proto` and `to_otel_proto` to ensure links *aren't* silently dropped and *are* properly exported, respectively, when interfacing with OTLP.

### Database Schema & Migration

A nullable `links` column of type `MutableJSON` will be added to the `spans` table. Going off of the intersection between the OTel standard and what is currently implemented in MLFlow, each link object will be comprised of:
- `trace_id` - an entity of `SpanContext`
- `span_id` - an entity of `SpanContext`
- `attributes`

Each of these will be serialized and deserialized according to the conventions currently existing in `Span.to/from_dict()`

For migration, an Alembic migration script will be contributed to add the column to the DB.

Backwards compatibility is achieved by virtue of the fact that the column is nullable.

### UI Additions

For the UI, add a new type to `mlflow/server/js/src/shared/web-shared/model-trace-explorer/ModelTrace.types.ts`

```typescript
export type ModelTraceSpanLink = {
    trace_id: string;
    span_id: string;
    attributes?: Record<string, any>;
};
```

...and add it to the appropriate existing types and interfaces (`ModelTraceSpanV4`, `ModelTraceSpanNode`).

Additionally, add a component to `ModelTraceExplorerDefaultSpanView.tsx` that displays:
- Trace ID (with URL when available)
- Span ID (with URL when available)
- Link attributes (if any)

## Drawbacks

- Database migration required, which must then be tested across all DB providers
- Added serialization complexity
- Added UI complexity
- Increased testing surface area (serialization/deserialization, database persistence, OTLP compatibility, UI rendering, backward compatibility scenarios)

# Alternatives

Given that this feature implements an existing OTel primitive, it naturally follows to mimic their implementation. As far as the API, relevant discussion points are left as open questions at the end of this document.

# Adoption strategy

This is an additive change and usage is purely opt-in (user explicitly specifies links during span instantiation or calls `span.add_link()`), so a successful implementation will be non-breaking. However, due to the size of this change, it may be worth considering splitting up work across sequential PRs. Tentatively, I would suggest:

1. Python API, database migration, and storage layer and relevant testing
    - Can also be broken down to first introduce "preferred" entrypoints and follow with support for the rest
2. UI components and TypeScript types
3. Documentation and examples

# Open questions

1. If Span A links to Span B via some expected span_id, should we validate whether Span B exists at the time of linking?
    - It might be the case that an application expects a Span to link to some other Span that is still yet to be initiated
        - Additionally, users might want to link Spans across experiments (it is also an open question as to whether we want to allow this behavior)

2. Should SpanLinks be bidirectional? E.g., if Span A links to Span B, should Span B be made to link to Span A? In Otel, these are unidirectional, but there might be a discussion to be had around at least enabling bidirectional span linking based on the type of workflows we expect to find this useful.
    - If yes, it would necessitate validating that spans exist at link-time as discussed in item 1.
