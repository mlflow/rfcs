---
start_date: 2026-08-18
mlflow_issue: https://github.com/mlflow/mlflow/issues/23202
rfc_pr: https://github.com/mlflow/rfcs/pull/34
---

<!-- markdownlint-disable-file MD041 -->

| Author(s)              | [Aarav Mittal](https://github.com/a2105z) |
| :--------------------- | :--------------------------------------- |
| **Date Last Modified** | 2026-08-18                               |

<!-- markdownlint-disable MD025 -->

# Summary: Official Terraform Provider for MLflow Resources

Ship an **official Terraform provider** (`registry.terraform.io/mlflow/mlflow`)
that manages the *declarative control-plane* of an MLflow deployment —
registered models, prompts, AI Gateway / Deployments endpoints, registry
webhooks, and (later) related governance objects — through MLflow's existing
REST APIs.

Today, teams can provision the *machine* that runs MLflow with Terraform
(Helm, Kubernetes, VMs, databases, object storage), but they cannot manage the
*contents* of that MLflow installation the same way. Models, prompts, gateway
routes, and webhooks are created with Python scripts, the CLI, ad-hoc `curl`,
or the UI. That creates drift across environments and keeps MLflow outside
normal GitOps review.

This RFC proposes a first-party provider under the `mlflow` GitHub organization,
implemented in Go with the Terraform Plugin Framework, published to the
Terraform Registry, and designed so a Pulumi bridge can be generated from the
same schema (addressing community interest in Pulumi without maintaining two
hand-written providers forever).

# Basic example

```hcl
terraform {
  required_providers {
    mlflow = {
      source  = "mlflow/mlflow"
      version = ">= 0.1.0"
    }
  }
}

provider "mlflow" {
  tracking_uri = var.mlflow_tracking_uri
  # Auth mirrors the Python SDK:
  #   MLFLOW_TRACKING_TOKEN / MLFLOW_TRACKING_USERNAME+PASSWORD
  #   MLFLOW_TRACKING_SERVER_CERT_PATH / MLFLOW_TRACKING_INSECURE_TLS
}

resource "mlflow_registered_model" "fraud" {
  name        = "fraud-detector"
  description = "Production fraud model family"
  tags = {
    team = "risk"
    env  = "prod"
  }
}

resource "mlflow_registered_model_alias" "champion" {
  model_name = mlflow_registered_model.fraud.name
  alias      = "champion"
  version    = var.fraud_champion_version
}

resource "mlflow_prompt" "support_triage" {
  name        = "support-triage"
  description = "Classifies inbound support tickets"
  tags = {
    team = "support"
  }
}

resource "mlflow_prompt_version" "v1" {
  prompt_name = mlflow_prompt.support_triage.name
  template    = file("${path.module}/prompts/support_triage.v1.txt")
}

resource "mlflow_prompt_alias" "production" {
  prompt_name = mlflow_prompt.support_triage.name
  alias       = "production"
  version     = mlflow_prompt_version.v1.version
}

resource "mlflow_deployments_endpoint" "chat" {
  name = "openai-chat"
  config = {
    model = {
      provider = "openai"
      name     = "gpt-4o-mini"
    }
  }
  # Optional rate limits when the Deployments server supports them
}

resource "mlflow_webhook" "model_created" {
  name        = "notify-model-created"
  url         = var.webhook_url
  events      = ["registered_model.created", "model_version.created"]
  description = "Notify platform bus when registry objects change"
  # secret is write-only in Terraform state (ephemeral / sensitive)
  secret = var.webhook_secret
}
```

`terraform plan` / `apply` then create, update, and delete these objects on a
live tracking / deployments server the same way operators manage the rest of
their platform.

## Motivation

### The problem (reproduced)

There is **no** provider named `mlflow/mlflow` on the Terraform Registry.

Reproduction (Terraform 1.11.4):

```hcl
terraform {
  required_providers {
    mlflow = {
      source  = "mlflow/mlflow"
      version = ">= 0.1.0"
    }
  }
}

provider "mlflow" {}
```

```text
Error: Failed to query available provider packages

Could not retrieve the list of available versions for provider mlflow/mlflow:
provider registry registry.terraform.io does not have a provider named
registry.terraform.io/mlflow/mlflow
```

Direct registry lookup returns the same gap:

```text
GET https://registry.terraform.io/v1/providers/mlflow/mlflow
→ {"errors":["provider not found"]}
```

Existing "MLflow + Terraform" repositories almost exclusively provision
**infrastructure around** MLflow (EKS, Postgres, MinIO, Helm charts). They do
not manage MLflow application resources. The closest community artifact is a
narrow auth-oriented experiment (`wadhah101/mlflow-auth-terraform-provider`),
not a maintained control-plane provider. Separately, [Baubap's Pulumi MLflow
provider](https://github.com/Baubap/pulumi-mlflow) and
[discussion #25049](https://github.com/mlflow/mlflow/discussions/25049) show
clear demand for IaC over registry objects — and explicitly note that a
Terraform provider with a Pulumi bridge is the natural long-term shape.

Meanwhile, the **imperative** path works today. Against a local SQLite-backed
tracking server, the Python SDK successfully creates the same kinds of objects
teams want in Terraform:

```text
OK registered_model: demo-model
OK prompt: demo-prompt
```

So the gap is not "MLflow lacks APIs." The gap is "there is no first-party,
declarative, reviewable client for those APIs in the ecosystem operators
already use for everything else."

### Why this matters

1. **Environment parity.** Dev / staging / prod MLflow tenants drift when
   models, prompt aliases, gateway endpoints, and webhooks are created by hand.
2. **Reviewability.** Platform changes should show up in `terraform plan`
   alongside IAM, DNS, and Kubernetes — not only in chat logs or one-off
   notebooks.
3. **Disaster recovery.** Recreating a tracking server from backups restores
   *data*; recreating *policy and routing* (aliases, gateway routes, webhooks)
   still requires tribal scripts unless those objects are declared.
4. **Sustainability.** Tracking prompts, deployments, webhooks, and future
   agent/MCP resources as they evolve is realistic only if the provider lives
   under the MLflow org (or an explicitly blessed sister repo), with release
   coupling to server APIs.

### Goals

1. Provide an official `mlflow/mlflow` provider on the Terraform Registry.
2. Cover the high-value **declarative** resource surface first (see phases).
3. Match Python SDK auth and TLS environment variables so operators are not
   taught a second credential model.
4. Keep the provider a pure API client — no embedded MLflow server, no schema
   migrations, no ownership of run/trace *data* planes.
5. Design the schema so a Pulumi terraform-bridge (or equivalent) can expose
   the same resources without a second hand-maintained implementation.
6. Support `import` for existing registry objects so adoption does not require
   recreate-from-scratch.

### Out of scope

1. **Provisioning the MLflow server itself** (Helm charts, operators, VMs,
   databases, object stores). That remains the job of existing infra modules.
2. **Run / experiment / trace / metric / artifact data** as managed resources.
   Those are high-churn scientific data, not durable platform config. (Optional
   *data sources* for read-only lookup may be added later; they are not Phase 1
   resources.)
3. **Training, logging, or evaluating models from Terraform.** Terraform
   applies should not start training jobs or upload large artifacts as a
   substitute for CI/CD.
4. **Unity Catalog–specific governance** beyond what the open Model Registry /
   Prompt APIs already expose to a standard tracking URI. UC-only extensions can
   be a later provider major or separate resources once API stability is clear.
5. **Replacing the Python/Java/R/TS SDKs.** The provider complements them for
   operators; application code continues to use SDKs.
6. **Guaranteeing multi-tenant RBAC design.** If `mlflow.server.auth` users /
   permissions are included, they land in a later phase after the OSS auth
   surface is treated as stable enough to encode in state.

## Detailed design

### Decision matrix: where should the provider live?

| Option | Pros | Cons | Verdict |
| --- | --- | --- | --- |
| **A. New repo `mlflow/terraform-provider-mlflow`** under the MLflow org | Standard for Terraform providers; independent Go module/release cadence; clear CODEOWNERS; Registry publishing is straightforward | Second repo to maintain | **Chosen** |
| B. Inside the `mlflow/mlflow` monorepo | Single clone for contributors | Mixes Python/JS release machinery with Go provider releases; painful for Registry/GPG signing workflows | Rejected |
| C. Bless a community provider only | Low maintainer cost short-term | Fragmentation; no Registry namespace `mlflow/`; breaks the "official" request in #23202 | Rejected as the end state (community prototypes remain welcome as feeders) |
| D. Pulumi-only official provider | Serves Pulumi users quickly | Leaves the larger Terraform user base without a path; duplicates work vs bridge-from-Terraform | Rejected as sole strategy; Pulumi bridge is an explicit follow-on |

### Architecture

```text
┌──────────────────────────┐
│  Terraform / OpenTofu    │
│  (HCL or generated SDK)  │
└────────────┬─────────────┘
             │ Plugin Protocol v6
┌────────────▼─────────────┐
│ terraform-provider-mlflow│
│  (Go, Plugin Framework)  │
│  resources + data sources│
└────────────┬─────────────┘
             │ HTTPS REST
     ┌───────┴────────┐
     │                │
┌────▼─────┐   ┌──────▼────────┐
│ Tracking │   │ Deployments / │
│ Server   │   │ AI Gateway    │
│ Registry │   │ /api/2.0/     │
│ Prompts  │   │ endpoints/    │
│ Webhooks │   └───────────────┘
└──────────┘
```

Implementation principles:

- **Terraform Plugin Framework** (not SDKv2) for modern resource semantics,
  import, and ephemeral values for secrets.
- **One HTTP client** shared by resources, configured from provider schema +
  the same env vars the Python SDK documents (`MLFLOW_TRACKING_URI`,
  `MLFLOW_TRACKING_TOKEN`, `MLFLOW_TRACKING_USERNAME`,
  `MLFLOW_TRACKING_PASSWORD`, `MLFLOW_TRACKING_SERVER_CERT_PATH`,
  `MLFLOW_TRACKING_INSECURE_TLS`).
- **Server feature detection** where needed (e.g. Prompt Registry or webhooks
  unavailable on older servers) with clear diagnostics rather than silent no-ops.
- **Idempotent Create**: if a registered model/prompt name already exists when
  Create runs and `import` was not used, return a actionable error pointing
  operators at `terraform import` (do not adopt foreign objects implicitly).

### Provider configuration

```hcl
provider "mlflow" {
  tracking_uri    = "https://mlflow.example.com"   # or env MLFLOW_TRACKING_URI
  deployments_uri = "https://gateway.example.com"  # optional; defaults to tracking_uri when co-hosted

  username = var.username          # or MLFLOW_TRACKING_USERNAME
  password = var.password          # or MLFLOW_TRACKING_PASSWORD (sensitive)
  token    = var.token             # or MLFLOW_TRACKING_TOKEN (sensitive; wins over basic auth)

  server_cert_path = "/etc/ssl/certs/mlflow-ca.pem"  # or MLFLOW_TRACKING_SERVER_CERT_PATH
  insecure_tls     = false                            # or MLFLOW_TRACKING_INSECURE_TLS
}
```

Precedence matches the Python SDK: explicit provider attributes override env
vars; token overrides username/password.

### Resource phases

#### Phase 1 — Registry fundamentals (MVP, targets first Registry release)

| Terraform resource | MLflow API surface (illustrative) | Notes |
| --- | --- | --- |
| `mlflow_registered_model` | create / update / delete / get registered model; tags | Name is the primary key |
| `mlflow_registered_model_alias` | set / delete registered model alias | Separate resource so alias moves are first-class plans |
| `mlflow_prompt` | create / update / delete prompt | Metadata + tags |
| `mlflow_prompt_version` | create prompt version | Template/content; versions are append-only |
| `mlflow_prompt_alias` | set / delete prompt alias | Same pattern as model aliases |
| `mlflow_webhook` | create / update / delete / list webhooks | Secret stored as sensitive / write-only |

**Explicitly not in Phase 1 as managed resources:** `model_version` objects that
point at large `source` artifact URIs produced by training pipelines. Aliases
are in scope because they are the durable "what prod points at" knob; the
version *payload* usually appears from CI. A later phase may add
`mlflow_model_version` for teams that truly pin sources in Terraform.

#### Phase 2 — Deployments / AI Gateway

| Terraform resource | API surface | Notes |
| --- | --- | --- |
| `mlflow_deployments_endpoint` | `POST/GET/DELETE /api/2.0/endpoints/` | Provider + model config |
| `mlflow_deployments_endpoint_limits` | `/api/2.0/endpoints/limits/` | When rate limiting is available |

`deployments_uri` may differ from `tracking_uri` for split deployments.

#### Phase 3 — Auth & expansion

- Optional `mlflow_user` / permission resources if OSS basic auth remains a
  supported control plane and the community still needs it (overlap with
  Baubap's Pulumi auth resources).
- Agent / MCP registry objects once those APIs stabilize (see related RFCs).
- Read-only data sources: `mlflow_registered_model`, `mlflow_prompt`,
  `mlflow_deployments_endpoint`.

### State, identity, and import

- **Registered model / prompt**: Terraform ID = resource name (server-unique).
- **Aliases**: ID = `name/alias`.
- **Prompt / model versions**: ID = `name/version` (version string returned by
  the server).
- **Webhooks**: ID = server-assigned webhook id.
- **Endpoints**: ID = endpoint name.

Every Phase 1 resource implements `ImportState`. Docs will include:

```bash
terraform import mlflow_registered_model.fraud fraud-detector
terraform import mlflow_prompt_alias.production support-triage/production
```

### Secrets and sensitive attributes

- Provider `password` / `token` and webhook `secret` are marked **sensitive**.
- Prefer Terraform ephemeral values / write-only attributes (Plugin Framework)
  so webhook secrets are not round-tripped into state after create when the
  API does not return them.
- Never log Authorization headers in provider verbose logs.

### Deletes and remote destruction

| Resource | Destroy behavior |
| --- | --- |
| Registered model | Delete model (document that versions/aliases are removed per server semantics) |
| Alias | Delete alias only |
| Prompt | Delete prompt |
| Prompt version | Prefer mark-deleted / tombstone if API lacks hard delete; document exact call |
| Webhook | Delete webhook |
| Endpoint | Delete endpoint |

A provider-level option `skip_destroy = true` (resource lifecycle
`prevent_destroy` remains available) is **not** invented unless real users hit
a production footgun; start with standard Terraform lifecycle meta-arguments.

### Compatibility matrix

Document in the provider README:

| Provider version | Minimum MLflow server | Notes |
| --- | --- | --- |
| 0.1.x | MLflow 2.22+ or 3.x with Model Registry | Phase 1 models |
| 0.1.x | MLflow 3.x Prompt Registry | Prompt resources require prompt APIs |
| 0.2.x | Deployments server endpoints API | Phase 2 |

CI will run acceptance tests against:

1. SQLite-backed `mlflow server` in GitHub Actions (Phase 1).
2. Deployments server container when Phase 2 lands.

### Repository layout (proposed)

```text
mlflow/terraform-provider-mlflow/
  README.md
  docs/                 # generated + hand-written Registry docs
  examples/
    registered-model/
    prompt-registry/
    gateway-endpoint/
  internal/
    provider/
    client/             # thin REST client
    resources/
  tools/
  .github/workflows/
    ci.yml
    release.yml         # goreleaser + Registry publish
```

Release with GoReleaser, GPG-signed checksums, and Terraform Registry
publishing under the `mlflow` namespace (org ownership required).

### Pulumi bridge (follow-on, same schema)

After Phase 1 is stable, generate a Pulumi provider via
`pulumi-terraform-bridge` (or the then-current HashiCorp/Pulumi recommended
path) so TypeScript/Python/Go Pulumi programs can manage the same resources.
This is the recommended answer to discussion #25049 rather than maintaining
divergent hand-written providers long term. Existing community Pulumi work can
inform resource naming and test cases; it does not block the Terraform-first
official path.

### Documentation & DX

- Registry docs for every resource/data source.
- "Migrating from scripts" guide: map common `MlflowClient` calls to resources.
- Example modules for env promotion (dev → staging → prod) using aliases.
- Explicit **non-goals** callout so users do not expect `terraform apply` to
  train models.

### Rollout plan

1. **RFC acceptance** (this document).
2. Create `mlflow/terraform-provider-mlflow` (empty Apache-2.0 repo, CODEOWNERS).
3. Implement Phase 1 resources + acceptance tests.
4. Publish `0.1.0` to the Terraform Registry under `mlflow/mlflow`.
5. Link from MLflow docs ("Manage MLflow with Terraform").
6. Phase 2 gateway endpoints; Phase 3 auth / data sources / Pulumi bridge.

## Drawbacks

1. **Maintainer cost.** A Go provider is another release train (binaries per
   OS/arch, Registry publishing, Schema docs). Mitigated by keeping the client
   thin and phasing resources aggressively.
2. **API churn.** Prompts, gateway, and webhooks are younger than Model
   Registry. Mitigated by feature detection and a clear compatibility matrix;
   unstable surfaces stay out of 0.1.0.
3. **False expectations.** Users may try to put training pipelines into
   Terraform. Mitigated by docs and by refusing Phase 1 `model_version` upload
   flows that encourage huge applies.
4. **Could be done in user space.** Teams can wrap REST calls in
   `local-exec` or custom providers. That is exactly the fragmented status quo
   this RFC rejects as the long-term answer.
5. **Not a breaking change** to MLflow itself — additive tooling — but bad
   destroy semantics could delete production registry objects. Mitigated with
   careful destroy docs, examples using `prevent_destroy`, and integration
   tests.

# Alternatives

1. **Do nothing.** Continue with scripts and UI click-ops. Drift remains;
   issue #23202 stays open; Pulumi/Terraform communities keep inventing partial
   wrappers.
2. **Only document REST + example scripts.** Helps slightly; does not give
   plan/apply/import or Registry discoverability.
3. **Terraform `http` / `restapi` provider modules.** Fragile, untyped, poor
   state diffing, no first-class import UX.
4. **Official Pulumi provider only.** Leaves Terraform/OpenTofu users behind;
   most infra orgs standardize on Terraform first.
5. **Crossplane / Kubernetes CRDs instead of Terraform.** Useful for
   K8s-centric platforms, but does not replace Terraform for the broader
   audience named in #23202. Could be a separate future project.

# Adoption strategy

- Purely additive: no change to existing MLflow servers required beyond running
  a recent enough version for the resources in use.
- Operators adopt by:
  1. Installing the provider from the Terraform Registry.
  2. Importing existing models/prompts/aliases/webhooks.
  3. Deleting imperative bootstrap scripts as resources move into HCL.
- OpenTofu is expected to work via the same Registry provider binary.
- Docs in `mlflow.org` should link to the provider once `0.1.0` ships.
- Coordinate naming with any community Pulumi work so resource names stay
  aligned when the bridge lands.

# Open questions

1. **Org/Registry permissions.** Who on the TSC/maintainers owns the
   `mlflow` Terraform Registry namespace and signing keys?
2. **Model versions in Terraform.** Confirm Phase 1 exclusion of
   `mlflow_model_version` (alias-only) vs. including a narrow version resource
   for URI-pinned artifacts.
3. **Split URIs.** Is a separate `deployments_uri` in the provider config
   enough for Phase 2, or do we need per-resource overrides?
4. **Webhook secret rotation.** Does the server API support rotate-in-place
   without recreate? Affects resource `Update` semantics.
5. **Auth resources.** Should OSS `mlflow.server.auth` users/permissions enter
   Phase 3, or remain permanently out of the official provider?
6. **Pulumi bridge ownership.** Same repo via bridge generation, or a sibling
   `pulumi-mlflow` under the MLflow org?
7. **CODEOWNERS / triage.** Which maintainers review provider releases, and
   should community PRs follow the same DCO process as `mlflow/mlflow`?

# Appendix A — Reproduction notes

Commands run while preparing this RFC (2026-08-18):

```bash
# Registry: official provider missing
curl -s https://registry.terraform.io/v1/providers/mlflow/mlflow
# {"errors":["provider not found"]}

# Terraform init failure (trimmed)
terraform init
# Error: provider registry registry.terraform.io does not have a provider named
# registry.terraform.io/mlflow/mlflow

# Imperative API still works (SQLite-backed tracking URI)
python - <<'PY'
from mlflow import MlflowClient
c = MlflowClient()  # MLFLOW_TRACKING_URI=sqlite:///...
c.create_registered_model("demo-model", tags={"env": "dev"})
c.create_prompt("demo-prompt", description="imperative", tags={"team": "ml"})
PY
```

# Appendix B — Related links

- MLflow issue: https://github.com/mlflow/mlflow/issues/23202
- Pulumi discussion: https://github.com/mlflow/mlflow/discussions/25049
- Community Pulumi provider: https://github.com/Baubap/pulumi-mlflow
- Deployments endpoints base path: `/api/2.0/endpoints/`
  (`mlflow/deployments/server/constants.py`)
