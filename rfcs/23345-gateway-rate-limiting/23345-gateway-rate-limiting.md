---
start_date: 2026-10-02
mlflow_issue: https://github.com/mlflow/mlflow/issues/23345
rfc_pr:
---

<!-- markdownlint-disable-file MD041 -->

| Author(s)              | [2sumtech](https://github.com/2sumtech) |
| :--------------------- | :-------------------------------------- |
| **Date Last Modified** | 2026-10-02                              |

<!-- markdownlint-disable MD025 -->

> **Code references.** File paths and symbol names below were checked against MLflow `3.16.2.dev0`
> (`mlflow/mlflow` master at `0b1e3700e`). Symbol names are the stable reference; no line numbers are given.

# Summary: Request Rate Limits for the AI Gateway

The legacy MLflow AI Gateway let an operator attach a `limit` (`calls` per `renewal_period`) to each route. The
new AI Gateway, which runs inside the tracking server and stores endpoints in the database, has no equivalent. It
has budget policies, which cap spend in dollars, but nothing that caps how many requests reach a provider in a short
window.

This RFC adds a `rate_limits` list to gateway endpoints. Each entry counts requests over a fixed window and applies
at one of two scopes:

- **`SERVICE`**: one counter shared by every caller of the endpoint. This bounds the total load the endpoint can put
  on its provider.
- **`USER_DEFAULT`**: one counter per caller, each with the same allowance. This stops a single caller from using up
  the endpoint's capacity.

Both scopes can be set on the same endpoint. A request is admitted only if every applicable counter has room, so the
stricter limit wins. Rejected requests get HTTP 429 with a `Retry-After` header and are not counted.

Counting goes through a `RateLimitTracker` interface with an in-process implementation and a Redis implementation,
selected the same way the budget tracker selects its backend. The field names and enum values follow the Databricks
AI Gateway rate limit API so a configuration can move between the two products with few changes; the Databricks
concepts that have no OSS counterpart (groups, service principals, per-principal overrides) are left out.

# Basic example

An endpoint that allows each caller 60 requests per minute, while keeping the endpoint as a whole under 600 requests
per minute and 20,000 per hour:

```yaml
# Body of CreateGatewayEndpoint / UpdateGatewayEndpoint (JSON shown as YAML for readability)
name: support-bot-chat
model_configs:
  - model_definition_id: d-1234
    linkage_type: PRIMARY
rate_limit_config:
  rate_limits:
    - key: SERVICE
      requests: 600
      renewal_period: MINUTE
    - key: SERVICE
      requests: 20000
      renewal_period: HOUR
    - key: USER_DEFAULT
      requests: 60
      renewal_period: MINUTE
```

A caller who sends a 61st request inside the same minute receives:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 23
Content-Type: application/json

{
  "detail": {
    "error_code": "RESOURCE_EXHAUSTED",
    "message": "Rate limit exceeded for endpoint 'support-bot-chat': 60 requests per minute per caller (USER_DEFAULT). Retry after 23 seconds."
  }
}
```

If instead the endpoint as a whole had already admitted 600 requests that minute, the message names the `SERVICE`
limit. Every caller is rejected until the window rolls over, including callers who are under their own allowance.

## Motivation

The gateway sits between applications and paid model providers. Operators need two guarantees that budgets alone
do not give:

1. **The provider is protected.** Providers enforce their own per-key quotas. If ten applications share one
   gateway endpoint and therefore one provider key, a burst from any of them can trip the provider's quota and cause
   errors for all ten. A shared request cap on the endpoint keeps total traffic under a number the operator picks.
2. **Callers are isolated from each other.** A runaway loop in one notebook should fail fast for that notebook, not
   use up the endpoint for everyone else.

Budget policies (`mlflow/gateway/budget.py`, `mlflow/gateway/budget_tracker/`) do not cover either case. They are
measured in dollars, and cost is recorded only after the provider responds (`make_budget_on_complete`). A burst of
concurrent requests all pass `check_budget_limit` before any of their cost is recorded, so a budget cannot bound the
number of requests in flight to the provider. Budgets are also not split per caller unless an admin creates a `USER`
policy for each named user.

### What the legacy gateway did

In the legacy gateway, `EndpointConfig.limit` (`mlflow/gateway/config.py`, class `Limit` with `calls`,
`renewal_period`, and an unused `key` field) is turned into a `slowapi` decorator in
`_get_endpoint_handler` (`mlflow/gateway/app.py`). The `Limiter` is built in `create_app_from_config` with
`key_func=get_remote_address` and an optional storage backend from `MLFLOW_GATEWAY_RATE_LIMITS_STORAGE_URI`. That
gives:

- The limit is configured per endpoint, but counted **per client IP** within that endpoint. Every caller gets the
  same allowance on its own counter.
- `slowapi` uses its default fixed-window strategy.
- Rejections return 429 through `slowapi`'s default handler. Rate limit headers, including `Retry-After`, are off by
  default and the gateway does not turn them on.
- There is **no cap on the endpoint's total traffic**. Load on the provider grows with the number of distinct
  clients.

The `USER_DEFAULT` scope in this RFC reproduces the legacy behavior. The `SERVICE` scope closes the gap that
@aaronteo-db identified on the issue.

### Goals

1. Request-count limits on gateway endpoints, at endpoint scope (`SERVICE`) and per-caller scope (`USER_DEFAULT`).
2. Correct counting across multiple server workers and replicas when Redis is configured.
3. A configuration shape close enough to Databricks AI Gateway that users of both do not have to learn two models.
4. No behavior change for endpoints that do not set `rate_limits`.

### Out of scope

1. **Token-based limits.** Token counts are known only after the provider responds, so a token limit cannot reject
   the request that crosses it. The `tokens` field is reserved in the schema (see below) so this can be added later
   without a new shape.
2. **Per-principal overrides, groups, and service principals.** Databricks lets an admin give a named user, group, or
   service principal its own limit. OSS MLflow has users (when auth is on) but no groups or service principals. Named
   per-user overrides are a natural follow-up and are listed under open questions.
3. **Concurrency limits** (maximum in-flight requests). A long streaming request counts once, however long it runs.
4. **Fairness across regions or across separate Redis instances.** One Redis is one counting domain.
5. **UI.** The endpoint editor can add a rate limit section later; this RFC defines the API and enforcement.
6. **The legacy gateway.** It keeps its `slowapi` implementation.

## Detailed design

### Configuration schema

A new message is added to `mlflow/protos/service.proto`:

```protobuf
enum RateLimitKey {
  RATE_LIMIT_KEY_UNSPECIFIED = 0 [(enum_value_visibility) = PUBLIC_UNDOCUMENTED];
  // One counter shared by all callers of the endpoint.
  SERVICE = 1;
  // One counter per caller; every caller gets the same allowance.
  USER_DEFAULT = 2;
}

enum RateLimitRenewalPeriod {
  RATE_LIMIT_RENEWAL_PERIOD_UNSPECIFIED = 0 [(enum_value_visibility) = PUBLIC_UNDOCUMENTED];
  MINUTE = 1;
  HOUR = 2;
}

message GatewayRateLimit {
  optional RateLimitKey key = 1;
  optional RateLimitRenewalPeriod renewal_period = 2;
  // Maximum number of requests per window. 0 rejects all requests.
  optional int64 requests = 3;
  // Reserved for token-based limits. Must be unset in this version.
  optional int64 tokens = 4;
}

message GatewayRateLimitConfig {
  repeated GatewayRateLimit rate_limits = 1;
}
```

`GatewayEndpoint` gains `repeated GatewayRateLimit rate_limits`. `CreateGatewayEndpoint` and
`UpdateGatewayEndpoint` gain `optional GatewayRateLimitConfig rate_limit_config`. The wrapper message exists so an
update can tell "leave rate limits alone" (wrapper absent) apart from "remove all rate limits" (wrapper present with
an empty list). A bare `repeated` field cannot express that difference.

Enum values in `service.proto` share the `mlflow` package scope, so they must not repeat names used by other enums.
`SERVICE`, `USER_DEFAULT`, `MINUTE`, and `HOUR` are currently free; `BudgetDurationUnit` uses `MINUTES` and `HOURS`,
and `BudgetTargetScope` already uses `USER` and `ENDPOINT`.

Validation on create and update:

| Rule | Reason |
| --- | --- |
| `key` is `SERVICE` or `USER_DEFAULT` | The only scopes OSS can resolve today |
| `renewal_period` is `MINUTE` or `HOUR` | Both are fixed windows that are cheap to count; longer periods are what budgets are for |
| `requests` is set and `>= 0` | `0` is a deliberate "block everything" setting, matching Databricks |
| `tokens` is unset | Reserved; rejected with a message that says token limits are not supported yet |
| At most one entry per `(key, renewal_period)` | Two entries for the same counter would be ambiguous |

Because of the last rule there are at most four entries per endpoint.

#### Relation to the Databricks shapes

Databricks has two rate limit shapes, and MLflow already speaks one of them. `DatabricksDeploymentClient` in
`mlflow/deployments/databricks/__init__.py` (`update_endpoint_rate_limits`) sends the serving-endpoint shape: a
`rate_limits` list of `{calls, key: endpoint | user, renewal_period: minute}`. The newer AI Gateway model service
API, which the issue links to, uses `config.rate_limits` entries with `key`, `renewal_period`, `requests`, `tokens`,
and `principal`, and its key values include a service-wide scope, a default per-user scope, and per-user, per-group,
and per-service-principal scopes.

This RFC follows the newer shape because that is the API the issue asks us to stay close to:

| Databricks AI Gateway | This RFC | Note |
| --- | --- | --- |
| `key` = service scope | `SERVICE` | Same meaning |
| `key` = default user scope | `USER_DEFAULT` | Same meaning; "user" is resolved as described below |
| `key` = user / group / service principal, with `principal` | not supported | No groups or service principals in OSS; named-user overrides deferred |
| `requests` | `requests` | Same meaning |
| `tokens` | reserved | Rejected for now |
| `renewal_period` minute / hour | `MINUTE` / `HOUR` | Enum values drop the long prefix to match MLflow's proto style (`BudgetTargetScope`, `GatewayModelLinkageType`) |

For the older shape, `endpoint` maps to `SERVICE`, `user` maps to `USER_DEFAULT`, and `calls` maps to `requests`.

### Who is the caller?

`USER_DEFAULT` needs a caller identity for every request. The resolution order is:

1. **Authenticated username**, when MLflow auth is enabled. The auth middleware in `mlflow/server/auth/__init__.py`
   already sets `request.state.username`, and `_get_request_username` in `mlflow/server/gateway_api.py` already reads
   it to enforce `USER`-scoped budget policies. Rate limiting uses the same value.
2. **Client IP address** (`request.client.host`), when auth is disabled. This is what the legacy gateway did through
   `slowapi.util.get_remote_address`, so it gives parity.

The identity is prefixed with its kind (`user:` or `ip:`) so a username can never collide with an address.

**Proxies.** If the server runs behind a reverse proxy or load balancer, `request.client.host` is the proxy's address
and every caller looks the same. `USER_DEFAULT` then behaves like a second `SERVICE` limit. The fix is the one
uvicorn already provides: configure the trusted proxy addresses (`--forwarded-allow-ips`, passed through
`mlflow server --uvicorn-opts`) so that uvicorn rewrites `request.client` from `X-Forwarded-For`. The gateway should
not parse `X-Forwarded-For` itself. Any client can set that header, and only the server operator knows which hops
to trust. The docs for this feature must explain this, because it fails quietly: limits still work, just at the
wrong granularity.

Without auth, a caller who can change source addresses can get more than one allowance. That is why the `SERVICE`
cap matters: it holds no matter how callers identify themselves.

### Enforcement point

`check_budget_limit` (`mlflow/gateway/budget.py`) is called near the top of every provider-facing route in
`mlflow/server/gateway_api.py`, eleven call sites in total: `invocations`, `chat_completions`, each provider
passthrough route (OpenAI chat, embeddings, and responses; Anthropic messages; Gemini generate and stream), the
`typesafe` passthrough, and `raw_proxy`. Each site has already resolved `endpoint_config` through
`get_endpoint_config` and has the `Request`.

A new `check_rate_limit(endpoint_config, request)` is called immediately **after** `check_budget_limit` at each of
those sites. The order matters:

- The budget check does not record anything, so running it first costs nothing.
- The rate limit check does record an admitted request. Running it last, just before guardrails and the provider
  call, means a request rejected by the budget never uses up a rate limit slot.
- A request admitted by the rate limiter and then blocked by a pre-LLM guardrail still counts. That is intended: the
  guardrail did real work, and not counting would let a caller probe guardrails without limit.

`list_models` (`GET /mlflow/v1/models`) does not call a provider and is not limited.

One gateway request counts as one request, even if fallback routing tries several models or the response streams for
a long time.

`rate_limits` is added to `GatewayEndpointConfig` (`mlflow/store/tracking/gateway/entities.py`), so it travels with
the endpoint config that `get_endpoint_config` (`mlflow/store/tracking/gateway/config_resolver.py`) already caches.
Unlike budget policies, rate limits need no separate refresh loop. An update reaches each worker when its cached
endpoint config is invalidated or expires.

### Admission algorithm

For a request to endpoint `E` from caller `C`, the tracker receives the endpoint's limits and builds one counter key
per entry:

- `SERVICE` entry: `(E.endpoint_id, SERVICE, -, period, window_index)`
- `USER_DEFAULT` entry: `(E.endpoint_id, USER_DEFAULT, C, period, window_index)`

`endpoint_id` is used instead of the name so renaming an endpoint does not reset its counters, and so endpoints with
the same name in different workspaces do not share counters.

Admission is all-or-nothing:

1. Read every counter for the current window.
2. If any counter is already at its `requests` value, reject. Pick the rejecting counter whose window ends last and
   report its remaining time as `Retry-After`, because the request cannot succeed before that one resets.
3. Otherwise increment every counter and admit.

Steps 1 to 3 happen atomically: under one lock in process, and in one Lua script in Redis. Checking before
incrementing is what keeps rejected requests from being counted. With a plain `INCR`-then-compare approach, a caller
rejected by the `SERVICE` cap would still use up their own `USER_DEFAULT` allowance, and a client that keeps retrying
would push its own counter further and further past the limit.

### Window type

| | Fixed window (proposed) | Sliding log (PR #26123) | Sliding window counter (two buckets, weighted) |
| --- | --- | --- | --- |
| Matches legacy gateway | Yes (`slowapi` default) | No | No |
| Matches budget windows | Yes (budget windows are fixed and epoch aligned) | No | No |
| State per counter | One integer | One timestamp per admitted request | Two integers |
| Redis cost | One key, `INCR` + `EXPIRE` | Sorted set, grows with `requests` | Two keys |
| `Retry-After` | Exact: time until the window ends | Exact | Approximate |
| Worst-case burst | Up to 2x `requests` across a window boundary | None | Small |

This RFC proposes **fixed windows aligned to the epoch**, using the same alignment as `_compute_window_start` in
`mlflow/gateway/budget_tracker/__init__.py`. The reasons are parity with the legacy gateway and with budgets, constant
memory per counter, and an exact `Retry-After`.

The cost is the boundary burst. A caller can send `requests` calls at the end of one window and `requests` more at
the start of the next. If an operator cares about this, they can add an `HOUR` limit, which bounds the total over
longer periods. A related effect is that all rejected callers are told to retry at the same moment. Clients should
add jitter. The server will not randomize `Retry-After`, because then the header would no longer be accurate.

If reviewers think the boundary burst is not acceptable, the sliding window counter is the fallback. It fits the same
tracker interface and costs one extra key per counter.

### Trackers

```python
class RateLimitTracker(ABC):
    @abstractmethod
    def try_acquire(
        self, endpoint_id: str, caller: str, limits: list[GatewayRateLimit], now: float | None = None
    ) -> RateLimitDecision: ...


@dataclass(frozen=True)
class RateLimitDecision:
    allowed: bool
    violated: GatewayRateLimit | None = None
    retry_after_seconds: int | None = None
```

`get_rate_limit_tracker()` copies `get_budget_tracker()`: a process-wide singleton built lazily under a lock. It
returns the Redis tracker when a Redis URL is configured and the in-process tracker otherwise. The rate limiter reads
the same `MLFLOW_GATEWAY_BUDGET_REDIS_URL` by default, with an optional `MLFLOW_GATEWAY_RATE_LIMIT_REDIS_URL` override.
Operators who already run Redis for budgets then get correct rate limits without extra configuration.

**In-process tracker.** A dictionary from counter key to `(window_index, count)`, guarded by one `threading.Lock`. No
I/O happens inside the lock. Entries from past windows are overwritten when touched. A sweep removes stale entries once
the dictionary passes a size threshold, so a scan from many client addresses cannot grow memory without bound.

The in-process tracker counts **per worker process**. `mlflow server` starts 4 workers by default (`--workers`,
`mlflow/utils/cli_args.py`), so without Redis an endpoint limited to 60 requests per minute can admit up to 240,
depending on how requests are spread across workers. Budgets have the same issue but partly hide it by backfilling
spend from trace history (`calculate_existing_cost_for_windows`). Rate limits have no history to backfill from. The
server should log a warning at startup when rate limits are in use, there is more than one worker, and no Redis URL is
set. The docs should say plainly that the in-process tracker is exact only with one worker and one replica.

**Redis tracker.** Uses the same client setup as `RedisBudgetTracker` (`mlflow/gateway/budget_tracker/redis.py`), which
already runs atomic Lua scripts (`_ENSURE_WINDOW_LUA`, `_RECORD_COST_LUA`). One script does the whole admission step:

1. Read the server clock with `TIME`, so all replicas agree on the window index even if their clocks drift.
2. Build each counter key from the index, for example `mlflow:ratelimit:{endpoint_id}:USER_DEFAULT:{caller_hash}:MINUTE:{index}`.
3. `GET` every key. If any value is at its limit, return the position of the violated limit and its remaining TTL.
4. Otherwise `INCR` every key, and on a key's first increment set `EXPIRE` to the window length plus a small margin.

The caller identity is hashed (SHA-256, truncated) in the key. This keeps keys short and keeps usernames and addresses
out of plain view in Redis. One script call per request is one Redis round trip, the same cost the budget check already
pays.

**When Redis is unavailable.** The proposal is to fail open: admit the request, log at warning level with rate
limiting so the log is not flooded, and count the failure. Rejecting all gateway traffic because the limiter's cache is
down would make an optional protection the cause of an outage. This is listed as an open question because some
operators will want the opposite.

### Rejection response

`check_rate_limit` raises the `HTTPException` that the gateway routes already use for budget rejections, with two
changes:

- `headers={"Retry-After": str(seconds)}`, where `seconds` is the time until the violated window ends, rounded up and
  at least 1.
- `detail` is the structured `{"error_code", "message"}` dictionary that `translate_http_exception`
  (`mlflow/gateway/utils.py`) already produces for `MlflowException`. The error code is `RESOURCE_EXHAUSTED`, which
  `mlflow/exceptions.py` already maps to 429. The budget path currently returns a plain string `detail`. Using the
  structured form here lets clients tell a rate limit apart from other 429s without parsing message text.

The message names the endpoint, the scope, the limit, and the period. It does not include the caller's IP address or
username.

Unlike `check_budget_limit`, rate limit rejections **do not create an error trace**. `_create_budget_error_trace`
makes sense for budgets, which trip rarely. A rate limiter can reject thousands of requests per minute during exactly
the kind of event it exists for, and writing a trace for each would move the load from the provider to the tracking
store.

### Persistence and API plumbing

The field follows the path that `fallback_config` and `usage_tracking` already take:

| Layer | Change |
| --- | --- |
| `mlflow/protos/service.proto` | Messages above; Python and Java stubs regenerated with `dev/generate-protos.sh` (Docker) |
| `mlflow/entities/gateway_endpoint.py` | `rate_limits` on `GatewayEndpoint`, with proto conversion |
| `mlflow/store/tracking/dbmodels/models.py` | `rate_limits_json` `Text` column on `SqlGatewayEndpoint`, stored like `fallback_config_json` |
| `mlflow/store/db_migrations/versions/` | One Alembic migration adding the nullable column |
| `mlflow/store/tracking/gateway/abstract_mixin.py`, `sqlalchemy_mixin.py`, `rest_mixin.py` | `rate_limits` argument on `create_gateway_endpoint` / `update_gateway_endpoint` |
| `mlflow/store/tracking/gateway/entities.py`, `config_resolver.py` | `rate_limits` on `GatewayEndpointConfig`, filled in `get_endpoint_config` |
| `mlflow/server/handlers.py` | Pass the field through the create / update handlers |
| `mlflow/gateway/rate_limit.py`, `mlflow/gateway/rate_limit_tracker/` | `check_rate_limit` and the trackers |
| `mlflow/server/gateway_api.py` | One call next to each `check_budget_limit` |

Storing the list as JSON on the endpoint row, instead of a separate table, matches how `fallback_config` is stored.
It also fits the data: at most four small entries, always read together with the endpoint, and never queried on their
own.

Who may change rate limits follows the existing gateway endpoint permissions in the auth plugin. Changing a limit is
an endpoint update.

### Observability

- A debug log line for each rejection (endpoint name, scope, period), and a rate-limited warning when the tracker
  fails.
- A rejected-request counter per `(endpoint_id, key)` kept in the tracker. This is useful for tests and a future
  admin view.
- MLflow's Prometheus exporter (`mlflow/server/prometheus_exporter.py`) is built on `prometheus_flask_exporter` and
  does not see the FastAPI gateway routes. Exporting the counter there needs its own small change. This RFC does not
  depend on it.

## Drawbacks

1. **The in-process tracker is wrong by default for multi-worker servers.** With the default four workers, the
   effective limit can be up to four times the configured one. A warning and docs reduce the surprise but do not fix
   it. The real fix is Redis, which is one more thing to run.
2. **Fixed windows allow a 2x burst at window boundaries.** This is the same behavior the legacy gateway had, but it
   is still a weaker guarantee than a sliding window.
3. **Caller identity without auth is weak.** IP-based identity breaks behind a misconfigured proxy and can be dodged
   by a caller who controls several addresses. The `SERVICE` cap limits the damage, but `USER_DEFAULT` is only as good
   as the identity it is given.
4. **More surface area.** A new proto message, a column, a migration, a tracker package, and eleven call sites. The
   tracker package largely repeats the budget tracker structure, and a later refactor could share the Redis client
   setup and backend selection.
5. **A second meaning of "user".** Budget policies use `USER` scope with a specific username in `target_value`.
   Rate limits use `USER_DEFAULT` to mean "each caller". The names follow Databricks, but users of both features have
   to learn the difference.
6. **Could this be done outside MLflow?** Partly. An API gateway or reverse proxy in front of MLflow can rate limit by
   IP. It cannot see MLflow usernames or endpoint IDs inside a path such as `/gateway/mlflow/v1/chat/completions`,
   where the endpoint name is in the request body. That is why the limits belong inside the gateway.

# Alternatives

**A single `calls_per_minute` per endpoint, sliding window, in process (closed PR mlflow/mlflow#26123).** This was the
first attempt. It added one integer column, a sliding log per endpoint name, and 429 with `Retry-After`. It was closed
for two reasons that this RFC takes as requirements: with only an endpoint-level counter, one caller can starve the
others, and it did not match the legacy gateway's per-client counting. It also had no Redis backend, so it had the
multi-worker problem with no way out. Its enforcement point and response format carry over into this design.

**Modelling rate limits as policies, like budgets.** Budget policies are separate entities with
`target_scope` (`GLOBAL`, `WORKSPACE`, `ENDPOINT`, `USER`) and their own CRUD API. A rate limit policy could have
the same shape and apply to many endpoints at once. This was rejected for now because the issue asks for per-endpoint
configuration in the Databricks shape, where limits live in the endpoint config. Endpoint-scoped config is also
simpler to cache and reason about. If workspace-wide or global request limits are needed later, a policy entity can be
added on top without changing the endpoint field.

**Reusing `slowapi` from the legacy gateway.** `slowapi` (through the `limits` library) already supports fixed and
sliding windows and Redis storage. It is built around route decorators keyed by a function, while the new gateway
looks up endpoints at request time and needs two counters checked together, with neither counted if either fails.
Using `limits` directly as a storage layer is possible, but it would add a dependency for what is one Lua script and
one dictionary.

**Token bucket.** Allows smoother bursts and is common in API gateways. It has no clear mapping to the
`requests` per `renewal_period` shape that both the legacy gateway and Databricks use, so configurations would no
longer move between the products.

**Not doing this.** Operators keep relying on provider-side quotas and budgets. Provider quotas are per API key, so
gateway callers sharing a key still affect each other. Budgets react after cost is recorded. Users moving from the
legacy gateway lose a feature they had.

# Adoption strategy

The feature is opt-in. Endpoints without `rate_limits` behave exactly as they do today, and the migration only adds a
nullable column.

Users of the legacy gateway can convert their config as follows:

| Legacy `limit` | New `rate_limits` entry |
| --- | --- |
| `calls: N` | `requests: N` |
| `renewal_period: minute` | `renewal_period: MINUTE` |
| `renewal_period: hour` | `renewal_period: HOUR` |
| Other `limits` granularities (second, day, ...) | Not supported; use a budget for day-scale control |
| (implicit) per client IP | `key: USER_DEFAULT` |
| `key` field | Ignored by the legacy gateway; no equivalent needed |
| `MLFLOW_GATEWAY_RATE_LIMITS_STORAGE_URI` | `MLFLOW_GATEWAY_BUDGET_REDIS_URL` or `MLFLOW_GATEWAY_RATE_LIMIT_REDIS_URL` |

A legacy limit converted this way gives the same behavior. Adding a `SERVICE` entry then adds the total cap that the
legacy gateway lacked.

Suggested implementation order, each step a separate PR:

1. Schema, storage, and REST plumbing, with validation. No enforcement yet.
2. `check_rate_limit` and the in-process tracker, wired into all gateway routes, with the multi-worker warning.
3. The Redis tracker.
4. Docs, next to `docs/docs/genai/governance/ai-gateway/budget-alerts-limits.mdx`.
5. UI, separately.

# Open questions

1. **Fail open or fail closed when Redis is down?** This RFC proposes fail open. An environment variable could let
   operators choose fail closed.
2. **Should `USER_DEFAULT` require auth?** Falling back to the client IP gives legacy parity but weak identity. The
   other option is to reject `USER_DEFAULT` entries when auth is off. That is safer but breaks the legacy migration
   path. This RFC proposes the IP fallback plus clear docs.
3. **A trusted caller header.** Some deployments put an authenticating proxy in front of MLflow with auth turned off
   inside MLflow, and the proxy passes the user in a header. A setting naming that header would give real per-user
   limits there. It is left out of the first version because, unless the operator controls every path to the server,
   any client can set the header. Is there demand for it?
4. **Named per-user overrides.** Databricks allows a limit for a specific user, through `principal`. OSS has usernames
   when auth is on, so a per-user key with `principal` could be supported. Is that wanted in the first version, or
   later? Note that the proto enum value `USER` is already taken by `BudgetTargetScope`, so that key would need a
   different name in the proto (for example `USER_OVERRIDE`) even if the JSON form stays close to Databricks.
5. **Rate limit response headers.** Should admitted responses carry remaining-quota headers (for example
   `X-RateLimit-Remaining`)? That would need one more return value from the tracker. This RFC sends only
   `Retry-After`, and only on 429.
6. **Budget error format.** Should the budget 429 switch to the same structured `detail` for consistency? That would
   be a small behavior change for clients that parse the current string.
