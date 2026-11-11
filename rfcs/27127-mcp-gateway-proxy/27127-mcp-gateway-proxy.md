# RFC 27127: MCP Gateway Proxy

| start_date   | 2026-10-27 |
| :----------- | :--------- |
| mlflow_issue | https://github.com/mlflow/mlflow/issues/27127 |
| rfc_pr       | |

| Author(s)              | [gps23](https://github.com/gps23) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-10-27 |
| **AI Assistant(s)**    | Claude (Cursor) |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [Why not use the MCP Registry alone?](#why-not-use-the-mcp-registry-alone)
  - [Who benefits](#who-benefits)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
  - [Architecture](#architecture)
  - [1. Endpoint type and data model](#1-endpoint-type-and-data-model)
  - [2. Transparent MCP proxy](#2-transparent-mcp-proxy)
  - [3. Credential management](#3-credential-management)
  - [4. Governance features](#4-governance-features)
  - [5. Tracing](#5-tracing)
  - [6. Gateway UI](#6-gateway-ui)
  - [7. Interaction with MCP Registry](#7-interaction-with-mcp-registry)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [Open questions](#open-questions)

# Summary

Extend the MLflow AI Gateway to proxy MCP (Model Context Protocol) tool calls with the same governance and observability that LLM calls already receive. This adds a new `mcp/v1/proxy` endpoint type that acts as a transparent reverse proxy for MCP servers, applying rate limiting, RBAC, guardrails, budget tracking, and auto-tracing to every proxied tool call.

This RFC is the **gateway companion** to [RFC 0004 (MCP Registry)](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md), which established the MCP Registry as "the governed source of truth" and noted that "a future MLflow MCP Gateway should build on the same server identities, versions, and aliases." This RFC delivers that gateway layer.

# Basic example

### Configuration

```yaml
endpoints:
  - name: brave-search
    endpoint_type: mcp/v1/proxy
    mcp_server:
      registry_name: "io.github.anthropic/brave-search"
      transport_type: streamable-http
      version_alias: production
    rate_limits:
      - scope: SERVICE
        window: MINUTE
        max_requests: 200
```

### Agent connection

Standard MCP clients connect to the gateway endpoint instead of the upstream server:

```python
# Before: agent connects directly to the MCP server
# mcp_url = "https://mcp.example.com/brave-search"

# After: agent connects through the AI Gateway
# mcp_url = "http://localhost:5000/gateway/brave-search/mcp"
```

The gateway transparently proxies JSON-RPC messages (`tools/call`, `tools/list`, `resources/read`, etc.) to the upstream server, applying governance policies at each step.

### SDK

```python
import mlflow

mlflow.gateway.create_endpoint(
    name="brave-search",
    endpoint_type="mcp/v1/proxy",
    mcp_server={
        "registry_name": "io.github.anthropic/brave-search",
        "transport_type": "streamable-http",
        "version_alias": "production",
    },
    rate_limits=[{"scope": "SERVICE", "window": "MINUTE", "max_requests": 200}],
)
```

## Motivation

### The problem

AI agents in production increasingly rely on MCP servers for real-world actions: database queries, web searches, code execution, and file operations. Today, these tool calls travel directly from agent to MCP server, completely bypassing the AI Gateway. This creates a governance gap:

| Capability | LLM calls (via Gateway) | MCP tool calls (today) |
|---|---|---|
| Centralized credentials | ✅ LLM connections | ❌ Scattered in agent configs |
| Rate limiting | ✅ Per-endpoint, per-user | ❌ None |
| RBAC | ✅ Role-based access | ❌ None |
| Budget tracking | ✅ Token cost attribution | ❌ None |
| Guardrails | ✅ Input/output policies | ❌ None |
| Tracing | ✅ Auto-traced spans | ❌ Manual instrumentation only |
| Usage analytics | ✅ Gateway dashboard | ❌ None |
| Audit trail | ✅ Full request logging | ❌ None |

As agents gain autonomy, tool-call governance becomes as critical as LLM-call governance. An agent that can query databases, write files, or call external APIs without centralized controls is a production risk.

### Why not use the MCP Registry alone?

The MCP Registry (introduced in MLflow 3.15.0) catalogs, versions, and manages MCP server metadata. It is a **metadata catalog**, not a runtime proxy. It stores `MCPAccessEndpoint` records (URL, transport type, version/alias) but does not:

- Intercept or forward MCP traffic
- Manage upstream credentials
- Apply rate limits, guardrails, or RBAC to tool calls
- Track tool call usage or costs
- Generate traces for proxied calls

The AI Gateway already has all this middleware for LLM traffic. This RFC proposes extending the gateway to also handle MCP traffic, using the registry as its metadata source where applicable.

### Who benefits

- **Platform teams** gain centralized control over all agent traffic (LLM + MCP) through a single governance layer
- **Security teams** get audit trails for tool calls and can enforce input/output policies
- **Finance teams** can attribute tool call costs and enforce budgets
- **Agent developers** get auto-tracing for MCP calls without manual instrumentation

### Out of scope

- **MCP server implementation**: This RFC does not add or modify MCP server functionality within MLflow. The existing MCP server that exposes MLflow tracking primitives is unaffected.
- **stdio transport**: Only network transports (`streamable-http`, and optionally `sse`) are in scope. The `stdio` transport is local-only and not suited for gateway proxying.
- **Tool-level routing**: Routing different tools from the same MCP server to different upstream endpoints is deferred.
- **MCP server discovery/registration**: The MCP Registry already handles this; the gateway references registry entries but does not replace them.

## Detailed design

### Architecture

```
┌──────────────┐     JSON-RPC      ┌─────────────────────────────────────┐     JSON-RPC      ┌──────────────┐
│              │  (streamable-http) │            AI Gateway               │  (streamable-http) │              │
│  MCP Client  │ ────────────────► │                                     │ ────────────────► │  MCP Server  │
│  (Agent)     │ ◄──────────────── │  ┌─────────┐ ┌──────┐ ┌─────────┐  │ ◄──────────────── │  (Upstream)  │
│              │                   │  │  Auth/   │ │ Rate │ │  Guard- │  │                   │              │
└──────────────┘                   │  │  RBAC    │ │Limit │ │  rails  │  │                   └──────────────┘
                                   │  └─────────┘ └──────┘ └─────────┘  │
                                   │  ┌─────────┐ ┌──────┐ ┌─────────┐  │
                                   │  │ Budget  │ │Trace │ │ Usage   │  │
                                   │  │Tracking │ │ Span │ │Analytics│  │
                                   │  └─────────┘ └──────┘ └─────────┘  │
                                   └─────────────────────────────────────┘
```

The gateway acts as a transparent MCP reverse proxy. Clients connect to the gateway endpoint URL using standard MCP transport. The gateway:

1. Terminates the client connection
2. Parses the JSON-RPC message to extract the method (`tools/call`, `resources/read`, etc.)
3. Runs the governance middleware pipeline (auth → rate limit → budget → input guardrails)
4. Forwards the request to the upstream MCP server
5. Receives the response
6. Runs output guardrails
7. Records a trace span and usage metrics
8. Returns the response to the client

### 1. Endpoint type and data model

#### New endpoint type

Add `MCP_V1_PROXY` to the endpoint type enum. The gateway currently defines endpoint types for LLM operations (`LLM_V1_CHAT`, `LLM_V1_COMPLETIONS`, `LLM_V1_EMBEDDINGS`). The new type is parallel to these.

```protobuf
enum EndpointType {
  LLM_V1_CHAT = 1;
  LLM_V1_COMPLETIONS = 2;
  LLM_V1_EMBEDDINGS = 3;
  MCP_V1_PROXY = 4;  // new
}
```

#### Endpoint configuration

MCP endpoints use an `mcp_server` configuration block instead of `model_configs`:

```protobuf
message McpServerConfig {
  // One of: reference a registry server by name, or provide a direct URL
  oneof target {
    string registry_name = 1;  // e.g. "io.github.anthropic/brave-search"
    string url = 2;            // e.g. "https://mcp.example.com/brave-search"
  }
  string transport_type = 3;     // "streamable-http" or "sse"
  string version_alias = 4;     // optional: "production", "staging", or semver
  string mcp_connection_name = 5; // optional: reference to stored credentials
}
```

#### Database schema

A new `mcp_server_configs` table (or a nullable `mcp_server_config` column on the existing endpoints table) stores the MCP-specific configuration. The design choice between these two approaches should be made during implementation review. The migration must not affect existing LLM endpoint rows.

#### CRUD API

The existing `create_gateway_endpoint` / `update_gateway_endpoint` / `delete_gateway_endpoint` REST and SDK APIs are extended:

- `create_gateway_endpoint` accepts either `model_configs` (LLM) or `mcp_server_config` (MCP), validated by `endpoint_type`
- The UI endpoint creation flow adds an "MCP Proxy" option alongside the existing LLM provider selector

### 2. Transparent MCP proxy

#### Route

```
POST /gateway/{endpoint_name}/mcp
```

This single route handles all MCP JSON-RPC methods. The gateway inspects the `method` field to determine which governance policies apply:

| JSON-RPC method | Governance actions |
|---|---|
| `initialize` | Auth only (establish session) |
| `tools/list` | Auth, rate limit |
| `tools/call` | Auth, rate limit, budget, input guardrails, output guardrails, tracing |
| `resources/list` | Auth, rate limit |
| `resources/read` | Auth, rate limit, input guardrails, output guardrails, tracing |
| `prompts/list` | Auth, rate limit |
| `prompts/get` | Auth, rate limit |
| Other / unknown | Auth, rate limit (pass through) |

#### Transport: streamable-http

The MCP `streamable-http` transport sends JSON-RPC messages over HTTP POST with `Content-Type: application/json` and can return either a single JSON response or an SSE stream. The gateway:

1. Accepts the client's POST request
2. Runs pre-request middleware
3. Forwards the POST to the upstream URL with the same body, adding any injected credentials
4. If the upstream returns `Content-Type: application/json`, runs post-response middleware and returns the response
5. If the upstream returns `Content-Type: text/event-stream`, streams events through, running post-response middleware on the final result event

#### Session management

MCP sessions are identified by `Mcp-Session-Id` header. The gateway:

- Forwards the header transparently between client and server
- Uses the session ID for sticky routing in multi-worker deployments (consistent hash on the session ID to select a worker)
- Does **not** manage session state itself — the upstream server owns session lifecycle

#### SSE transport (deferred)

Legacy `sse` transport support is deferred to a later phase. The `streamable-http` transport is the recommended MCP transport and covers both request-response and streaming patterns. If user demand arises, `sse` can be added as a separate endpoint within the same `mcp/v1/proxy` type.

### 3. Credential management

#### MCP connections

Introduce "MCP connections" following the same pattern as existing LLM connections:

```python
mlflow.gateway.create_mcp_connection(
    name="brave-search-key",
    connection_type="api_key",  # or "oauth", "mtls"
    credentials={"api_key": "sk-..."},
)
```

Credentials are stored encrypted, just like LLM provider API keys. An MCP endpoint references a connection by name:

```yaml
endpoints:
  - name: brave-search
    endpoint_type: mcp/v1/proxy
    mcp_server:
      registry_name: "io.github.anthropic/brave-search"
      transport_type: streamable-http
      mcp_connection: brave-search-key
```

The gateway injects credentials into upstream requests as configured:
- `api_key`: Added as `Authorization: Bearer <key>` header
- `oauth`: Token exchange performed by the gateway; access token injected as `Authorization: Bearer <token>`
- `passthrough`: Caller's credentials forwarded as-is (for environments where agents carry their own tokens)

### 4. Governance features

#### Rate limiting

MCP endpoints use the same `rate_limits` mechanism as LLM endpoints (per the rate limits RFC, mlflow/rfcs#63):

```yaml
rate_limits:
  - scope: SERVICE
    window: MINUTE
    max_requests: 200
  - scope: USER_DEFAULT
    window: MINUTE
    max_requests: 20
```

Rate limiting applies at the endpoint level. Per-tool rate limiting is deferred.

#### RBAC

When RBAC is enabled, MCP gateway endpoints are resources that can be granted to roles, the same as LLM endpoints. The existing RBAC framework for gateway endpoints should extend naturally — an MCP endpoint is an endpoint with a different type.

#### Guardrails

MCP guardrails inspect tool call payloads:

- **Input guardrails** run on `tools/call` requests, inspecting the `params.arguments` object. Example: block PII in tool arguments.
- **Output guardrails** run on `tools/call` responses, inspecting the `result.content` array. Example: redact secrets in tool output.

The guardrail interface is the same as for LLM calls, but the payload extraction differs:

```python
# LLM guardrails extract from chat messages
# MCP guardrails extract from JSON-RPC params/result

def extract_mcp_input(jsonrpc_request):
    """Extract guardable content from an MCP tools/call request."""
    return jsonrpc_request.get("params", {}).get("arguments", {})

def extract_mcp_output(jsonrpc_response):
    """Extract guardable content from an MCP tools/call response."""
    return jsonrpc_response.get("result", {}).get("content", [])
```

Guardrail policies are configured per-endpoint. The same guardrail definitions used for LLM endpoints (block_pii, redact_secrets, etc.) should be reusable for MCP endpoints.

#### Budget tracking

MCP tool calls to paid APIs have costs, but the cost model differs from LLM calls (which are token-based). For MCP endpoints, cost can be configured as:

```yaml
budget:
  cost_per_call: 0.003  # USD per tool call
```

This is a simple flat-rate model. More granular models (per-tool costs, usage-based pricing) are deferred. The budget tracker records the configured cost per call and enforces limits using the same `BudgetTracker` infrastructure.

### 5. Tracing

Every proxied MCP call generates an MLflow trace span with the following attributes:

| Attribute | Source |
|---|---|
| `mcp.tool.name` | `params.name` from `tools/call` |
| `mcp.tool.arguments` | `params.arguments` from `tools/call` |
| `mcp.tool.output` | `result.content` from response |
| `mcp.server.name` | Endpoint's registry name or URL |
| `mcp.session.id` | `Mcp-Session-Id` header |
| `mcp.method` | JSON-RPC method name |
| `mlflow.gateway.endpoint` | Gateway endpoint name |
| `mlflow.gateway.latency_ms` | Round-trip time |
| `mlflow.gateway.policy_violations` | List of triggered guardrail policies |

These attributes align with the emerging [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) direction and MLflow's existing trace attributes for LLM calls.

### 6. Gateway UI

MCP endpoints appear in the AI Gateway dashboard alongside LLM endpoints:

- **Endpoint list**: MCP endpoints are shown with an "MCP" badge, distinguishing them from LLM endpoints
- **Endpoint detail**: Shows connected MCP server, registry version, connection status
- **Usage analytics**: Tool call count, latency percentiles (p50/p95/p99), error rate, top tools by call volume
- **Budget view**: Cost-per-call spending against configured limits

### 7. Interaction with MCP Registry

The gateway and registry interact as follows:

- The gateway **reads** from the registry to resolve `registry_name` → upstream URL and transport type
- The gateway **caches** the resolved URL and tool definitions, refreshing on a configurable interval or on-demand via `?refresh=true`
- The gateway does **not write** to the registry — it is a consumer, not a producer
- If a registry server's version alias is updated (e.g., `production` points to a new version), the gateway picks up the change on next cache refresh
- If the registry is unavailable, the gateway uses its cached configuration and logs a warning

## Drawbacks

### Implementation complexity

This is a large change. The gateway's data model, schema, CRUD APIs, proxy logic, governance middleware, tracing, and UI all need MCP-aware extensions. The triage bot estimated "Large" effort, which is accurate.

**Mitigation**: Phased implementation (see Adoption strategy) limits the blast radius of each PR.

### Protocol complexity

MCP's JSON-RPC transport with stateful sessions is more complex than proxying REST-style LLM API calls. Session stickiness, streaming responses, and the `initialize` handshake add edge cases.

**Mitigation**: Start with `streamable-http` only (the simplest network transport) and defer `sse`. Session stickiness via consistent hashing is a well-understood pattern.

### Maintenance burden

The MCP specification is still evolving. Changes to the protocol may require gateway proxy updates.

**Mitigation**: The transparent proxy approach minimizes coupling — the gateway inspects method names but does not deeply parse MCP-specific semantics beyond what governance requires.

### Cost model simplicity

The flat `cost_per_call` model is simplistic compared to LLM token-based costing. Some MCP servers may have usage-based pricing that this model cannot capture accurately.

**Mitigation**: Start simple, iterate. The flat model covers the common case (per-call API pricing). A more granular model can be added later without breaking changes.

# Alternatives

### 1. Extend the MCP Registry instead of the Gateway

Add proxy capabilities directly to the MCP Registry (`gateway_proxied: true` on access endpoints).

**Rejected because**: The registry is a catalog; adding runtime proxy logic would blur its purpose and duplicate the gateway's existing middleware pipeline. The gateway is architecturally the right place for request proxying.

### 2. MLflow Plugin

Implement MCP proxying as an external MLflow Plugin.

**Rejected because**: MCP proxying requires deep integration with the gateway's internal middleware pipeline (auth, rate limiting, guardrails, tracing, budget tracking). This pipeline is not exposed through the plugin interface.

### 3. External proxy (nginx/Envoy) + manual configuration

Deploy a standalone reverse proxy in front of MCP servers and manually configure rate limiting, auth, and logging.

**Rejected because**: This duplicates governance logic already in the AI Gateway, requires separate infrastructure, and does not integrate with MLflow Tracing, the MCP Registry, or the Gateway UI. It is the current workaround, not a solution.

### 4. REST-style API wrapper instead of transparent proxy

Expose MCP functionality through REST endpoints (`/mcp/tools/call`, `/mcp/tools/list`, `/mcp/resources/read`) instead of a transparent JSON-RPC proxy.

**Rejected as the primary interface because**: This requires custom clients — standard MCP clients (Claude Code, Cursor, OpenAI agents) could not connect without modification. However, REST convenience endpoints could be offered as an optional secondary interface.

# Adoption strategy

### Phased rollout

| Phase | Scope | Estimated PRs |
|-------|-------|---------------|
| **Phase 1: Core proxy** | `MCP_V1_PROXY` endpoint type, schema migration, CRUD API, transparent proxy for `streamable-http`, basic rate limiting, tracing spans | 2-3 PRs |
| **Phase 2: Credentials & policies** | MCP connections (credential storage/injection), RBAC for MCP endpoints, input/output guardrails | 2-3 PRs |
| **Phase 3: UI & analytics** | Gateway UI for MCP endpoints, usage analytics dashboard, budget tracking with `cost_per_call` | 1-2 PRs |

### Migration path

This is a purely additive change. Existing deployments are unaffected:

- LLM endpoints continue to work exactly as before
- The `MCP_V1_PROXY` type defaults to absent — no MCP endpoints are created unless explicitly configured
- Agents currently connecting directly to MCP servers can migrate incrementally by pointing their MCP URL at the gateway endpoint

### Breaking changes

None. The new endpoint type, routes, and SDK methods are all additions.

### Documentation

- AI Gateway docs: new "MCP Proxy Endpoints" section
- MCP Registry docs: cross-link to gateway proxy setup
- Migration guide: "Moving from direct MCP connections to gateway-proxied connections"

# Open questions

1. **Endpoint creation API shape**: The current `create_gateway_endpoint` requires `model_configs`. Should we add `mcp_server_config` as an alternative parameter, or create a separate `create_mcp_gateway_endpoint` function?

2. **Guardrail payload format**: LLM guardrails operate on chat messages (structured text). MCP tool arguments can be arbitrary JSON (objects, arrays, nested structures). Should MCP guardrails serialize the arguments to a string for policy evaluation, or should the guardrail interface be extended to accept structured data?

3. **Multi-worker session routing**: For multi-worker gateway deployments, consistent hashing on `Mcp-Session-Id` is the proposed approach. Should this use the same mechanism as Redis-backed rate limiting (i.e., require Redis for session routing), or should it be handled at the load balancer level?

4. **Registry version change behavior**: When a registry server's version alias is updated, should the gateway immediately disconnect existing sessions and reconnect to the new version, or should existing sessions continue on the old version until they naturally end?

5. **Metrics and telemetry**: Should MCP endpoint metrics be stored in the same tables/structures as LLM endpoint metrics, or in separate MCP-specific tables? Using the same structures keeps the UI simple but requires generalizing column names (e.g., "tokens" → "units").
