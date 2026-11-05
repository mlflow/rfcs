---
start_date: 2026-10-30
mlflow_issue: # assign the RFC number when the enhancement issue is created
rfc_pr:
---

# AI Asset Version Synchronization

## Summary

Automatically synchronize skills, agent plugins, and MCP server definitions from GitHub into
MLflow's registries. The implementation uses an asset-independent source plugin interface rather
than GitHub-specific logic: the built-in Git provider discovers SemVer tags or follows a branch.
Administrators can allow other Git hosts, and third-party providers can support non-Git sources.

Users configure sync policies on each registry to select sources, discovery paths, and import
settings. MLflow discovers assets and periodically registers missing versions, recording where to
fetch their content from the original source by default rather than copying it. Users can instead
configure the sync policy to copy skill and plugin content into MLflow artifact storage.

Skills currently receive server-assigned integer versions, so importing an upstream version such as
`1.2.0` would lose that version identity. This RFC adds optional SemVer versioning so synchronized
skills imported from tags use the publisher's versions. Branch-tracked standalone skills use the default
integer scheme.
Agent plugins and MCP servers already use SemVer.

## Motivation

Teams already publish skills and agent plugins through versioned repositories. Importing each
version manually or maintaining a separate CI integration duplicates work across registries.
Integer-only skill versions also lose the publisher's version identity, making it harder to
relate a registered skill to its upstream package.

## Basic example

A user configures a source on the Skills registry using defaults:

```yaml
provider: git
url: https://github.com/acme/agent-skills.git
```

MLflow discovers skills throughout the repository and creates SemVer skills using their declared
names. Synchronization starts at the highest available SemVer tag and imports future missing
versions at or above that floor. Stable versions become `active`, prereleases become `draft`, and
content remains at its original source.

The same tag-based policy with all optional settings overridden:

```yaml
provider: git
url: https://github.com/acme/agent-skills.git
organization: acme
allowed_paths:
  - skills/review
  - skills/testing
starting_point: "1.2"
default_status: draft
storage_mode: artifact
minimum_age_seconds: 86400
enabled: false
```

This policy is initially disabled. Once enabled, it discovers skills under the two allowed paths
and imports missing versions at or above `1.2.0` into the `acme` organization as drafts. It copies
their content into MLflow artifact storage, which requires artifact serving.
Versions with timestamps must be at least one day old; versions without timestamps are not delayed.
The floor also admits a maintenance version such as `1.2.1` published after `1.3.0`.

To follow a branch instead:

```yaml
provider: git
url: https://github.com/acme/agent-skills.git
allowed_paths:
  - skills
branch: main
```

MLflow imports the current head, then processes every newly reachable commit on subsequent syncs.
Discovered skills use monotonic versioning. Changes to each skill create integer versions;
unrelated repository changes do not.

## Out of scope

- Replacing, restoring, or deleting versions in response to upstream changes.
- Automatically promoting versions or moving named aliases.
- GitHub release events, webhooks, and non-Git built-in providers.

## Detailed design

### Version identity

Add `versioning_scheme` to the registered skill, with values `monotonic` (default) and `semver`.
The scheme is set when the registered skill is created and cannot change afterward.
Creating a SemVer skill version requires
an explicit version, supplied by the caller or synchronization.

For standalone skills, tag-based sync requires SemVer and branch tracking requires monotonic
versioning. Agent plugins and MCP servers use their manifest versions in either mode; packaged
member skills inherit their agent plugin's SemVer version.

Reuse MLflow's SemVer utilities and MCP ordering approach:

- Accept `1` and `1.2` as `1.0.0` and `1.2.0`. Upstream discovery also accepts a leading `v`.
- Preserve the upstream spelling without the leading `v` for display, while using the normalized
  version for registry identity, uniqueness, lookup, and aliases.
- Follow the MCP registry's version-ordering approach: store parsed SemVer fields and use them
  for ordering. Build metadata remains part of identity but does not affect SemVer precedence;
  use the same stable tie-breaking behavior.
- Agent plugins and MCP servers use their manifest versions as registry identities, even when
  they differ from the Git tag. Standalone skills have no version metadata: tag-based imports
  inherit the tag version, while branch imports receive server-assigned integer versions.

Equivalent spellings identify one version. If discovery returns multiple tags for that version,
coalesce them when they resolve to the same source; otherwise report an ambiguity before importing
it.

Preserve existing numeric responses for monotonic skills; SemVer skill versions are strings.
Reuse lifecycle-aware latest resolution, changing its version ordering according to the parent's
scheme.

### Sync policy

Policies belong to a workspace and one registry: Skills, Agent Plugins, or MCP Servers. Each
registry can have multiple source policies, and each policy can discover and register multiple
assets.

| Field | Behavior |
| --- | --- |
| `provider` | Name of an administrator-enabled source plugin |
| `url` | Source location understood by the provider |
| `organization` | Target organization for skills/plugins; default empty organization; not applicable to MCP servers |
| `allowed_paths` | Optional nonempty list of source-relative directories to search recursively; omitted means the entire source |
| `branch` | Git branch to follow instead of discovering tags; omitted by default |
| `starting_point` | Inclusive SemVer floor for tags or commit SHA for branches; defaults to the highest available version or current branch head |
| `default_status` | `active` (default) or `draft` for stable versions |
| `storage_mode` | `pointer` (default) or `artifact` for skill/plugin content |
| `minimum_age_seconds` | Nonnegative minimum age; default `0` |
| `enabled` | Whether to schedule future syncs; default `true` |

Skill synchronization reuses the existing CLI import discovery, validation, and registration
behavior, restricted to `allowed_paths`. The sync policy determines versioning, status, and storage.
Unlike the CLI's source-ref-based reuse check, branch sync skips unchanged skill content even when
the commit changes.

Plugins and MCP servers are discovered by their manifests and definition files within
`allowed_paths`. Overlapping paths must not import the same asset twice.

For agent plugins, paths select whole packages, not individual members or supporting files.
The imported package remains complete. Restricting discovery or removing an asset upstream does
not delete previously registered assets.

`branch` selects branch tracking and cannot be combined with a nonzero
`minimum_age_seconds`. Commit timestamps do not establish when content arrived on a branch.
Without `branch`, the policy discovers upstream versions.

Resolve and persist `starting_point` during configuration; it is immutable once syncing starts.
For branches, resolve it to a full commit SHA and require it to be an ancestor of the branch head.
For tags, an explicit SemVer floor need not exist upstream; an omitted start requires at least one
valid version.

In tag mode, the starting point is a fixed lower bound on source versions, not manifest versions. Each sync
considers all source versions at or above that bound, including versions older than those already
imported, and registers only missing registry versions. Prerelease source or manifest versions
are always imported as `draft`. The
existing draft lifecycle and resolution behavior remains unchanged: draft is not an access-control
approval gate.

In branch mode, skills use `default_status`; plugins and MCP servers use it unless their manifest
version is a prerelease, in which case they import as `draft`.

Minimum age applies to scheduled and manual syncs only when the provider supplies a timestamp.
Resolve an omitted starting point before applying the age requirement, so a newer version that
is not yet old enough does not cause an older version to become the floor.

Changing or removing a policy does not alter a sync already started: that work uses its policy
snapshot, and the next run uses the new configuration. Disabling prevents future syncs rather than
cancelling in-flight work.

Policy management, source discovery, and manual sync require registry-level management permission
because they initiate server-side access and can create assets. Scheduled writes remain subject
to target and member permissions.

### Branch history

The starting commit is optional and defaults to the branch head resolved during configuration.
The first sync imports its snapshot, then all commits reachable from the pinned head but not from
that starting commit. Subsequent syncs use the last completed head instead.
Include commits introduced through merges in deterministic parent-before-child order, not
timestamp order. The starting point defines the initial backfill; the checkpoint records progress.

At each commit, discover assets within the allowed paths. For each standalone skill, compare its
content with its previously imported snapshot using the existing content digest. Changed content
creates a monotonic version; unchanged content is a no-op. This preserves intermediate changes and
reversions. Plugins and MCP
servers import each previously unseen manifest version; a commit with an existing manifest version
creates nothing. Plugin members are imported only with a new parent plugin version.

Process commits in bounded batches. Persist the pinned head and progress so retries resume without
creating duplicate versions. Stop at failed imports and advance the checkpoint only after the
entire range succeeds. Fail if rewritten history makes the checkpoint unreachable from the branch.

### Source plugin interface

Discover administrator-installed providers through the `mlflow.asset_sync_sources` entry-point
group. `MLFLOW_SYNC_SOURCE_ALLOWED_PLUGINS` is a comma-separated allowlist of provider names,
defaulting to `git`. It controls all provider use, including UI discovery; an empty value disables
all providers.

The interface knows about versions and source files, not asset types or registry writes:

```python
Source = GitSource | OCISource | ZipSource


@dataclass(frozen=True)
class Release:
    version: str
    reference: str | None = None
    timestamp: datetime | None = None


@dataclass(frozen=True)
class SourceRevision:
    revision: str
    source: Source


@dataclass(frozen=True)
class ReferenceChanges:
    head: str
    revisions: list[SourceRevision]


class AbstractAssetSyncSource(ABC):
    default_allowed_host_pattern: str
    supports_reference_tracking: bool = False

    def __init__(self, *, allowed_hosts: str | None = None): ...

    @abstractmethod
    def list_releases(self, url: str) -> list[Release]: ...

    @abstractmethod
    def resolve_release(self, url: str, release: Release) -> SourceRevision: ...

    @abstractmethod
    def list_files(self, source: Source) -> list[str]: ...

    @abstractmethod
    def fetch_files(
        self,
        source: Source,
        paths: list[str],
        *,
        max_bytes: int,
    ) -> dict[str, bytes]: ...

    def list_reference_changes(
        self,
        url: str,
        reference: str,
        *,
        after: str | None = None,
        starting_point: str | None = None,
    ) -> ReferenceChanges:
        raise NotImplementedError
```

`Release` describes a candidate upstream version, including a Git tag. Its optional `reference`
is opaque to MLflow and is only needed when the version alone cannot identify the upstream entry.
MLflow normalizes versions, ignores non-SemVer entries, and determines eligibility.
For eligible releases, `resolve_release` returns the pinned source and its opaque revision
identifier. Branch tracking returns the same `SourceRevision` type through `ReferenceChanges`.

Providers supporting reference tracking return ordered revisions from an inclusive `starting_point`
or after a saved checkpoint; omitting both returns only the current head. The provider owns history
traversal. MLflow processes the revisions and checkpoints the returned `head` after completion,
reusing the pinned history on retries. Other providers reject reference-tracking policies.

`SourceRevision.source` uses the existing pull descriptors: Git pins a commit SHA, OCI pins a
digest, and ZIP uses a version-specific URL. `list_files` returns only paths relative to that
source. Listing, fetching, and pointer registration use the same resolved source and selected
asset path.

The provider handles upstream pagination and fails rather than returning a truncated listing.
Timestamps, when present, are timezone-aware UTC values.

`max_bytes` caps aggregate content bytes, enforced while reading. Return all requested files or
fail. MLflow validates paths, completeness, and size using existing content validation and limits.

MLflow owns asset validation, registration, and canonical content-digest calculation from fetched
files. Provider revision identifiers pin snapshots but do not determine whether asset content changed.

### Built-in Git provider and network access

The Git provider discovers tags and follows branches through Git CLI/GitPython rather than a
forge API, so repositories need not publish forge-specific releases. Reuse existing Git
content-fetching protections against executing repository hooks or filters.

Annotated tags provide an author-controlled tagger timestamp; lightweight tags provide none.
Do not substitute commit timestamps. Minimum age is best-effort filtering, not a publication-age
guarantee.

Each provider declares a default source-host regex, matched against the complete parsed hostname.
Git defaults to `github\.com`. Administrators override it through
`MLFLOW_SYNC_SOURCE_<PLUGIN>_ALLOWED_HOSTS`, where `<PLUGIN>` is the entry-point name uppercased
with hyphens and periods replaced by underscores. Reject provider names that collide under this
mapping.

MLflow reads and validates the regex, then passes it as `allowed_hosts` to the provider constructor.
An unset variable passes `None`, selecting the provider default; an override replaces that default.
For example:

```bash
MLFLOW_SYNC_SOURCE_GIT_ALLOWED_HOSTS='github\.com|git\.example\.com'
```

Allowing a hostname does not permit private-network access, which is configured separately. The
initial Git transport is HTTPS; reject local paths and alternate transports that bypass the
network policy.

By default, reject connections to non-public addresses, including loopback, private, and link-local
IPv4/IPv6 addresses. Enforce this on actual connections, including DNS resolution and redirects,
for both HTTP clients and Git subprocesses. Redirect hosts need not match the source-host regex,
but must pass the address checks without leaking credentials across origins.

Credentials are deployment-managed, not embedded in policies. Providers may access ambient
credentials, so installation alone does not enable them: administrators can exclude providers
that would use capabilities such as the server's S3 credentials. If an administrator disables a
provider, syncs using it fail with an error explaining that the provider is disabled.

Surface transient failures as temporarily unavailable, back off, and honor retry timing when
supplied. Log credential/quota guidance where applicable.

### Import and persistence

Reuse the validation, lifecycle, and atomic registration behavior from
[RFC-0008](../0008-mvp-skill-registry/0008-mvp-skill-registry.md) and
[RFC-0004](../0004-mcp-registry/0004-mcp-registry.md).

The policy's `storage_mode` selects between registering a source pointer (`pointer`, the default)
and copying content into MLflow artifact storage (`artifact`).

Pointer mode records the resolved source without retaining fetched skill/plugin content. Consumers
still need permission to access that source; importing it does not expand upstream access.
Artifact mode is available only when MLflow serves artifacts and snapshots the selected content
through existing upload handling. An agent plugin stores one package tree, with member skills referencing
its subdirectories. MCP imports persist the canonical definition JSON, not the server's packages
or implementation, so the content storage choice does not apply to MCP servers.

Skip existing canonical registry versions, including deleted versions, without comparing content
or changing metadata. For plugins and MCP servers, read the manifest to identify the registry
version before making this check. This also applies when another import registers the version
concurrently. Upstream tag movement or removal does not mutate a registered version.

Record provenance: source version without the leading `v` when present, source location, resolved
ref/digest when available, selected path, and policy identifier. Keep per-policy last-attempt state
and a summary of imported, skipped, deferred, and failed versions by asset. One invalid asset must
not abort unrelated imports; branch checkpoints still wait for all imports in the range to succeed.

### Packaged member skills

For synchronized agent plugins, derive each member's registry name as
`<agent-plugin-name>-<skill-name>` in the plugin's organization. Preserve the skill's original name
and paths inside the package. Members use SemVer and inherit the agent plugin's normalized manifest
version in both tag and branch modes. Each new agent plugin version creates member versions even
when their content is unchanged, preserving their package-derived sources.

Create missing member skills. Reuse an existing skill only if it uses SemVer and was previously
imported with the same agent plugin; otherwise fail the import. Retained deleted history counts,
but assembled references do not, and deleted versions are never restored.

Reject invalid or colliding generated names. Register the plugin and member versions atomically;
an existing member version that cannot be reused under the package-source rules fails the import.

### Scheduling and UI

Initially, one periodic Huey task processes enabled policies with bounded parallelism. Network
fetching runs in the task process or subprocesses; it requires no new container backend. Concurrent
manual requests use the same execution path and are coalesced with work already pending or running
for that policy.

Deployments must run this scheduler on one replica. Existing Huey locking is not a general
cross-replica guarantee when replicas use independent storage; this RFC does not close that gap.
RFC-0002 can later move scheduling, locking, and execution into the shared job framework without
changing source plugins or policies.

Add a Sync sources section to each registry with a form for the provider, source URL, allowed paths,
tag or branch mode, and import settings. In SemVer/tag mode, entering the source URL automatically
calls the provider's `list_releases` through the server and populates a starting-version selector with valid
SemVer candidates, ordered by version. Default to the latest available version; show timestamps
when provided and explain when minimum age cannot apply. Branch mode accepts a branch name and
optional starting commit.

Provide discovery previews, save and sync, enable/disable, and sync now. Show the resolved starting
point and per-asset results and errors; asset pages show sync provenance. Expose equivalent
operations through the registry APIs and SDK.

## Adoption and tradeoffs

SemVer skill references require updated clients; existing monotonic workflows remain compatible.
No existing skill history is converted.

Pointer storage depends on upstream availability and permissions. Artifact storage removes that
dependency but can expand access: users permitted to retrieve the content through MLflow may not
have permission to access its original source.

Fetching now occurs in a server-managed process, unlike ordinary pointer registration. Provider
allowlisting, network restrictions, and resource bounds are required even though imported content
is not executed. Stronger process isolation can follow the remote-executor work.

## Open questions

- Should prefixed member naming also become the default for manual packaged imports? Define how
  existing unprefixed memberships transition to synchronization without silently renaming skills
  or changing their versioning scheme.
