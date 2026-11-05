start_date: 2026-10-29
mlflow_issue: https://github.com/mlflow/mlflow/issues/27154
rfc_pr:

# Summary

Propose an opt-in, read-only viewer for a snapshot of a run's training dataset. A snapshot is stored as bounded, independently readable artifact chunks and associated with one run input. The viewer preserves record order and duplicates, renders recognized SFT conversations, and exposes the original record when rendering fails. This is a proposal for discussion, not an accepted design.

# Basic example

A team fine-tunes a support assistant on JSONL records containing `messages`, including multi-turn tool interactions. After a model behaves unexpectedly, an engineer opens the training run and inspects the examples that were captured when training started. Some examples have an unknown role or missing content; the engineer needs to see the original records, not only the examples that pass a chat parser.

Illustrative Python, with a **proposed interface**, not an existing MLflow API:

```python
with mlflow.start_run():
    mlflow.log_input(training_dataset, context="training")  # existing API
    mlflow.log_input_snapshot(  # proposed API
        dataset=training_dataset,
        records_path="train.jsonl",
        completeness="complete",
    )
```

The proposed call requires an already logged input in the active run. It captures the supplied JSONL in order; the caller is responsible for supplying the actual data consumed by training. The run's dataset drawer offers **View snapshot** when a snapshot is available. The viewer loads one page, shows a completeness label, and opens an individual row as chat or raw JSON.

## Motivation

Today `mlflow.log_input()` logs dataset metadata and a run association, not training records. Its dataset name and digest identify metadata for tracking purposes, but are not a guarantee that the original data can be reconstructed. For example, the pandas digest samples rows and excludes some column types. The source can also refer to a mutable or inaccessible location.

`mlflow.log_table()` can log a JSON or Parquet table artifact and is useful for curated examples. It does not itself associate that table with a particular dataset input, provide this snapshot's completeness and integrity contract, or provide a paginated SFT viewer. GenAI evaluation datasets store records and expose a paginated API, but their inputs, outputs and expectations model serves evaluation; the SQL store deduplicates by input hash. Converting training data to that model can lose the original structure or duplicate examples. Neither workaround covers the request to inspect the original ordered training records for a run.

### Out of scope

- Server-side full-text or field search, filtering, sorting and editing in the first version. Search over an entire large dataset is deferred; a browser search over the current page must be labeled as such.
- Reading arbitrary external sources on the tracking server, generic data registration, and automatic capture by training integrations.
- Proving that a trainer actually consumed the snapshot, or supporting every SFT schema, binary media format and non-JSONL input.

## Detailed design

### Decision matrix

| Approach | Fidelity and reproducibility | Scale and search | Access and operational cost |
|---|---|---|---|
| Opt-in artifact snapshot (proposed) | Captures supplied ordered records independently of a mutable source; completeness depends on the caller. | Bounded chunks support incremental reads. No cross-snapshot index or full-dataset search in v1. | Duplicates data and incurs artifact storage and upload costs; access follows the associated run and artifact policy. |
| Retrieve from source | Avoids a copy, but sources may change, disappear or require a version and source-specific transformations. | Depends on each source's paging and search capabilities. | Adds server credentials, network access and source-specific authorization; local or code sources may be unavailable. |
| Dedicated record store | Could preserve ordered records and immutable versions if designed for them. | Can support indexing and search, at the cost of ingestion and database capacity. | Requires a new data model, migrations, retention and workspace-aware permissions. |

The artifact approach is recommended for a bounded first release because it avoids a new general-purpose record database and server access to arbitrary sources. It does not make large-scale search free. A dedicated store or versioned source adapters could be reconsidered after maintainers agree on scale and search requirements.

### Capture and association

A proposed `log_input_snapshot` uploads a snapshot under a reserved run-relative artifact prefix and registers a manifest for an existing `(run_id, dataset name, dataset digest)` input. The server verifies the input belongs to that run and its workspace before accepting the association. A run-scoped association avoids assuming that the same name and digest across runs implies identical bytes. Initial support is one published snapshot per input; replacement and multiple snapshots are open questions. Reject a second registration rather than silently replacing it.

A manifest, illustrated below, is proposed rather than an existing MLflow format. The checksum is SHA-256 over the exact bytes of each chunk; an aggregate checksum covers the concatenated chunk bytes in manifest order. Each JSONL line is a record. Chunk boundaries occur only between lines, so concatenating chunks reproduces the captured input bytes, including duplicate lines and their order. The capture process rejects invalid UTF-8 or invalid JSON lines with line numbers rather than silently dropping them. Valid JSON with an invalid SFT shape is retained. The writer must preserve line endings and a missing final newline; it must not parse and reserialize records for storage.

```json
{
  "format_version": 1,
  "format": "jsonl-utf8",
  "snapshot_id": "snap-example",
  "input": {"name": "support-sft", "digest": "a1b2c3d4"},
  "completeness": "complete",
  "source_record_count": 500,
  "captured_record_count": 500,
  "sha256": "<sha256 of concatenated chunk bytes>",
  "chunks": [{"path": "chunks/00000.jsonl", "first_record": 0,
              "record_count": 500, "bytes": 120000, "sha256": "<chunk sha256>"}]
}
```

The example lists the entire one-chunk snapshot. A complete snapshot omits sampling; a sampled snapshot states its method and, when known, source count and seed. The UI must never present a sample as complete. Completeness is a caller assertion that the captured records cover the training input. Checksum verification establishes that snapshot bytes match the manifest, not that the assertion is true or the trainer consumed those bytes. Those are three separate claims. Empty snapshots are valid. Capture enforces configurable byte, record, chunk and individual-record limits; exceeding a limit fails explicitly rather than silently truncating. Exact defaults and limits need maintainer agreement.

Publication spans artifact storage and the tracking database; no transaction covers both. The client uses a unique attempt ID and prefix, uploads chunks, verifies their sizes and hashes, then uploads the manifest. A proposed registration request verifies the manifest and referenced chunks and atomically creates the run-input association in the tracking store only after those artifacts are available. The association stores the SHA-256 of the exact manifest bytes. Readers discover snapshots only through that association and verify the manifest against its stored checksum before trusting its chunk hashes.

A retry uses the same attempt ID and identical manifest: an already registered matching attempt returns success; a different snapshot for the input is a conflict. If upload or verification fails, registration is not created. If the registration response is lost, the client retries or queries the association to determine the outcome. An unregistered attempt can be retried with the same bytes or abandoned; a conservative age-based cleanup can remove abandoned prefixes after checking they are unregistered, with coordination to avoid racing active uploads. Cleanup policy and backend consistency guarantees need validation. If registered artifacts later disappear or change, the reader reports an error.

Published paths are unique and the proposed writer does not overwrite them, but underlying artifact storage is not assumed immutable. The reader verifies each requested chunk against the registered manifest and reports missing or changed data. Aggregate verification occurs before registration or during a full verification operation, not by downloading all chunks on every page. The manifest checksum detects artifact-hosted manifest changes as long as the tracking-store association remains trusted. These hashes do not protect against an attacker who can modify the association as well; stronger tamper resistance depends on backend controls.

### Proposed read API and pagination

Proposed endpoints, names and payloads are illustrative:

```text
GET /api/3.0/mlflow/runs/{run_id}/inputs/dataset-snapshot?name=...&digest=...
GET /api/3.0/mlflow/runs/{run_id}/inputs/dataset-snapshot/records?name=...&digest=...&max_results=100&page_token=...
```

The first returns the published snapshot ID, manifest summary and availability. The second returns each record as its original UTF-8 JSON text, including its original line terminator or lack of one, in order, plus its ordinal, a next-page token and integrity status for the chunks read. JSON response escaping is transport encoding: decoding the string recovers the original text. Clients may parse a copy for rendering, but parsed values alone do not preserve whitespace, key order or number spelling. Tokens identify a snapshot version and the next chunk and record position; reject invalid tokens and tokens for a different snapshot. Server-side page-size and response-byte limits bound each response. A record larger than the response limit returns an explicit error with its ordinal, not an endless token loop.

Chunks have a maximum byte size and record count. The server reads only chunks needed for a page and verifies each before returning records; a page may span chunks. The artifact repository's capabilities and actual download behavior must be validated for each supported backend. If it cannot fetch a chunk without downloading the whole snapshot, the implementation must not claim scalable paging. Chunk size bounds per-request I/O, but a large snapshot still costs storage and upload time. The UI must request pages lazily, not fetch all pages and slice them in the browser.

Every request resolves the run and association server-side and checks run read, workspace and artifact access. The existing `validate_can_read_run_artifact()` obtains `run_id` or `run_uuid` from query arguments via `_run_id_or_uuid_param()`, not from a route path. The proposed path-based routes therefore need explicit authorization registration and a validator that extracts the route `run_id`, calls the equivalent run authorization, and checks the association and workspace before artifact access. The existing `_get_request_param()` includes route arguments and gives them precedence over query or body values; its applicability to the new routes should be verified. Registration and upload require the corresponding update permission. Tests must verify that the path ID authorized is the same run ID used by the handler. Do not accept arbitrary artifact URIs or paths from the browser. A snapshot may contain sensitive material, so opting in changes who can see copied data: deployment documentation must describe permissions, storage encryption and retention. Run deletion and artifact garbage collection should govern snapshot lifecycle; dangling associations, independently deleted artifacts, retention differences across backends and orphaned uploads need explicit handling. A missing or corrupt snapshot is an error state, not an empty dataset.

### UI and reuse

The existing run dataset drawer provides navigation and metadata. Add a snapshot availability state and a link to a read-only records view. Display completeness, count, capture details and integrity status. The table requests only the current page. A row detail panel can adapt the trace explorer's conversation normalization and renderer for recognized `messages` and prompt/completion forms. Preserve role order, show metadata, and offer the original record text for every record, with an optional parsed JSON view. Unknown roles, missing fields, unsupported content and rendering errors must produce a useful raw view and a diagnostic without preventing other records from loading. Rendering is an interpretation, not a validation that an example is suitable for training.

Existing evaluation dataset record endpoints and their cursor design are useful references, but their IDs and schema are distinct. In particular, the current evaluation v2 UI fetches every page before slicing; that behavior should not be reused for the proposed viewer.

### V1 acceptance criteria

- An opted-in JSONL snapshot can be registered for an existing run input and opened from its drawer; existing inputs without snapshots remain usable.
- Pages retain record order, duplicates and original text, and read only bounded chunks; corrupt or missing chunks produce an error. Recognized conversations render as chat, with original text available for malformed SFT records.
- The UI labels complete versus sampled capture and distinguishes caller assertions from verified bytes. Run and workspace authorization applies to reads and registration.
- Full-dataset search is deferred. Any current-page search is explicitly labeled and does not imply that unloaded records were searched.

### Compatibility and failure handling

`log_input()` remains metadata-only, with no automatic upload or changed entity semantics. Existing runs show their current metadata and an unavailable snapshot state. Older clients continue to log inputs; newer clients should report an unsupported server clearly. The proposal requires new API and association support rather than reinterpreting existing digest or source fields. A partial registration, invalid manifest, checksum mismatch, permission denial or unavailable artifact yields a distinct error. Tokens must not expose credentials or allow a user to select another run's snapshot.

### Verification

Test byte-for-byte round trips, ordering and duplicate lines, missing final newlines, empty input, samples and explicit limit failures. Exercise page boundaries, oversized records, invalid or cross-snapshot tokens, missing and modified manifests and chunks, interrupted publication, retry after a lost registration response, conflicting registrations, cleanup races, and supported artifact backends. Test run and workspace isolation, unauthorized reads, deleted runs and retention behavior. UI tests should cover lazy page requests, conversation and prompt/completion rendering, malformed SFT shapes, raw fallback and legacy runs. Performance tests must measure bounded memory and I/O for late pages on representative artifact stores. Workspace-aware tracking tests are required for any tracking-store changes.

## Drawbacks

The proposal copies potentially sensitive training data, increases artifact costs, and introduces a manifest, capture API and reader that users must understand. Chunking and integrity validation add implementation and operational complexity. A user can already log a small table or artifact and inspect it outside MLflow; this proposal is worthwhile only if integrated run navigation and SFT inspection justify that complexity. JSONL-only capture and no full-dataset search limit the first release. Storage permissions and retention may differ from those of the original source.

# Alternatives

Source-based retrieval avoids snapshot duplication but makes fidelity and availability depend on external versions and credentials. A dedicated record store offers a path to search, but is a larger platform commitment. Extending evaluation datasets would reuse existing UI and APIs but imposes evaluation semantics and deduplication on training data. Doing nothing leaves users with table artifacts, external tools or manually mapped evaluation datasets, without a consistent run-input snapshot contract.

# Adoption strategy

After design agreement, introduce the feature as opt-in and document the distinction between lineage metadata, captured samples and complete snapshots. Users can continue existing workflows without migration. Start with explicit JSONL capture; consider integration-driven capture only after size, privacy and fidelity expectations are settled. No changes to existing run inputs or evaluation datasets are required by default.

# Open questions

1. Is a run-scoped snapshot the right identity, or should snapshots be shared across runs as separately versioned objects? How should replacement and multiple snapshots work?
2. What limits and chunk sizes are acceptable, and can each supported artifact backend read individual chunks efficiently and consistently? How should orphan cleanup coordinate with active or retried uploads?
3. What permissions, retention, deletion and audit guarantees should govern copied records, especially when source access is narrower than run access? How should route-aware authorization integrate across supported server configurations?
4. Which SFT shapes and column mappings should be recognized, including multimodal content and tool calls? Should invalid JSON lines be capturable as raw bytes in a later format?
5. What search requirements justify an index, source adapter or dedicated store, and should sampling be deterministic beyond the initial `first_n` example?
6. Should completeness be asserted only by the caller, or can integrations provide stronger evidence that the trainer consumed the captured bytes?

## Code references

These references describe the existing MLflow checkout, not the proposed interfaces:

- [`mlflow/tracking/fluent.py`: `_create_dataset_input`, `log_input`, `log_table`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/tracking/fluent.py); [`mlflow/data/dataset.py`: `to_dict`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/data/dataset.py); [`digest_utils.py`: `compute_pandas_digest`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/data/digest_utils.py).
- [`sqlalchemy_store.py`: `_log_inputs_impl`, `_load_dataset_records`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/store/tracking/sqlalchemy_store.py); [`dbmodels/models.py`: `SqlDataset`, `SqlInput`, `SqlEvaluationDatasetRecord`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/store/tracking/dbmodels/models.py); [`file_store.py`: `log_inputs`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/store/tracking/file_store.py).
- [`service.proto`: `GetDatasetRecords`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/protos/service.proto); [`handlers.py`: `_get_dataset_records_handler`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/server/handlers.py); [`auth/__init__.py`: `validate_can_read_run_artifact`, `validate_can_read_dataset`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/server/auth/__init__.py).
- [`ExperimentViewDatasetDrawer.tsx`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/server/js/src/experiment-tracking/components/experiment-page/components/runs/ExperimentViewDatasetDrawer.tsx); [`useDatasetsQueries.tsx`: `useListDatasetRecordsQuery`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/server/js/src/experiment-tracking/pages/experiment-evaluation-datasets-v2/hooks/useDatasetsQueries.tsx); [`ModelTraceExplorer.utils.tsx`: `normalizeConversation`](https://github.com/mlflow/mlflow/blob/63e255225/mlflow/server/js/src/shared/web-shared/model-trace-explorer/ModelTraceExplorer.utils.tsx).
