# Hash-Ledger Merge Strategy Design

## Summary

Add a new `merge` strategy named `hash-ledger` for resources that should preserve an append-only row-content ledger instead of maintaining a key-based current-state table. `hash-ledger` inserts new live events for unseen row hashes, appends tombstone events during authoritative full-snapshot runs, and never mutates or deletes existing ledger rows.

This strategy is intended for users who want content-addressed history first and will derive their own downstream current-state views if needed.

## Goals

- Add a first-class `merge` strategy named `hash-ledger`
- Keep the destination table physically append-only
- Represent row lifecycle with live events and tombstone events
- Support mixed cadence resources that run incrementally most days and as authoritative full snapshots on selected runs
- Preserve `A > B > A` history where the same row content hash may reappear later as a new live event
- Keep v1 scoped to root tables only

## Non-Goals

- Providing merge-like current-state table semantics
- Supporting child-table history ledgers in v1
- Supporting multi-run authoritative snapshots in v1
- Adding a generic pipeline-wide full-snapshot mode
- Auto-creating helper views or materialized current-state projections

## User Model

Users configure a resource with:

- `write_disposition={"disposition": "merge", "strategy": "hash-ledger"}`

`dlt` computes `_dlt_hash` automatically from row content. Users do not provide it manually.

Some runs are ordinary incremental runs. Some runs are explicitly marked by the resource invocation as authoritative full snapshots. Only authoritative full-snapshot runs may emit tombstones.

## Table Semantics

Each physical row is a ledger event.

Required strategy metadata columns:

- `_dlt_hash`: content hash of the user row
- `_dlt_is_deleted`: boolean tombstone marker
- existing system columns, including `_dlt_load_id` and `_dlt_id`, remain available for event identity and lineage

Ledger events:

- Live event: `_dlt_hash=<hash>`, `_dlt_is_deleted=false`
- Tombstone event: full prior payload copied from the latest live event for that hash, `_dlt_hash=<same hash>`, `_dlt_is_deleted=true`

`_dlt_hash` is not unique across history. It identifies row content, not the event row.

Example `A > B > A` lifecycle:

1. insert live `A`
2. insert tombstone `A`
3. insert live `B`
4. insert tombstone `B`
5. insert live `A`

At most one event per `_dlt_hash` per run is allowed.

## Hash Contract

`_dlt_hash` is computed from a canonical JSON representation of the user row.

Rules:

- include user columns only
- exclude `_dlt_*` system columns and `hash-ledger` metadata columns
- include nested content when hashing root rows
- treat `null` and missing as distinct
- use deterministic key ordering in the serialized representation

Implications:

- changing a value changes the hash
- adding, removing, or renaming a column changes the hash
- schema evolution that changes row shape changes the hash
- the same logical row content produces the same `_dlt_hash` even if it reappears later in history

## Run Classification

`hash-ledger` owns its run classification and does not introduce a new generic pipeline concept.

There are two run classes for a `hash-ledger` resource:

- ordinary run: insert newly live hashes only, no tombstones
- authoritative full-snapshot run: insert newly live hashes and append tombstones for hashes that are currently live but missing from the completed snapshot

The authoritative full-snapshot signal is passed as a resource argument at invocation time.

This keeps the signal:

- run-scoped, not schema-scoped
- local to resources using `hash-ledger`
- explicit enough to avoid accidental tombstoning from inferred partial runs

## Batching and Snapshot Boundaries

One `pipeline.run(...)` may stream many yielded batches, pages, or files and still count as one authoritative full snapshot. In that case, reconciliation happens only after the run completes.

Separate `pipeline.run(...)` invocations are separate runs with separate `_dlt_load_id` values. They must not be stitched together into one authoritative snapshot in v1.

Therefore:

- batched delivery inside one run is supported
- multi-run snapshots are unsupported in v1
- if the user splits one logical snapshot across multiple runs, that usage is unsupported in v1 and must not be treated as an authoritative full snapshot

## In-Run Processing

During a run:

- deduplicate identical hashes within the run
- allow at most one event per `_dlt_hash` for the run

For authoritative full-snapshot runs:

- collect the set of hashes seen during the run
- do not emit tombstones batch-by-batch
- reconcile only after the final batch is loaded for the run

If an authoritative full-snapshot run fails before completion, discard its in-progress seen-hash set. Only completed full snapshots may drive tombstoning.

Retries should be idempotent at the ledger-event level.

## Reconciliation Rules

### Ordinary Runs

Insert a live event for each incoming `_dlt_hash` that is not currently live.

Do not:

- append tombstones
- append duplicate live events for hashes that are already live

If a previously tombstoned hash reappears on an ordinary run, insert a new live event.

### Authoritative Full-Snapshot Runs

After the run finishes:

1. insert live events for snapshot hashes that are not currently live
2. append tombstones for hashes that are currently live but absent from the completed snapshot

If the authoritative snapshot is empty, tombstone every currently live hash.

If the tombstone wave is unusually large, warn in logs/trace but do not block the load.

## Source of Truth for Liveness

The destination ledger table is the source of truth for current liveness. v1 should not persist the full live-hash set in resource state.

Current live set derivation:

1. group ledger rows by `_dlt_hash`
2. take the latest event for each hash by maximum `_dlt_load_id`
3. keep hashes whose latest event has `_dlt_is_deleted=false`

This model handles repeated content hashes naturally, including `A > B > A`.

`_dlt_load_id` ordering is sufficient because `hash-ledger` guarantees at most one event per hash per run.

## Schema Evolution

Schema evolution is part of row identity for `hash-ledger` because the hash is based on canonical row JSON.

Examples:

- `{a: 1, b: 1}` -> `{a: 1, b: 1, c: 1}` produces a new hash
- `{a: 1, b: 1}` -> `{a: 1, c: 1}` produces a new hash
- `{a: 1, b: null}` -> `{a: 1}` produces a new hash

This is intended behavior.

## Root-Table Scope

v1 supports root tables only.

Rationale:

- child-table history ledgers require additional design around normalized nested tables
- partial history on child tables is one of the confusing aspects of SCD2 that this design should avoid inheriting by accident
- users who need whole-document history can choose to avoid unnesting upstream

## Validation and Error Handling

Validation rules:

- `hash-ledger` is a valid merge strategy value
- the strategy must inject and manage `_dlt_hash` and `_dlt_is_deleted`
- authoritative full-snapshot mode does not support multi-run stitched snapshots in v1
- duplicate same-hash events in a single run are deduplicated before reconciliation

Operational behavior:

- failed authoritative snapshot runs do not tombstone
- retries should not append duplicate events for the same logical run outcome
- suspiciously large tombstone waves emit warnings only

## Querying Current Live Rows

`hash-ledger` remains a raw ledger strategy.

v1 will not add a built-in helper API for current-state queries. A documentation example should show how to derive the current live set from the ledger by selecting the latest event per `_dlt_hash` and filtering to `_dlt_is_deleted=false`.

## Testing Focus

Tests should cover at least:

- strategy resolution and validation
- automatic `_dlt_hash` generation
- deduplication of identical rows within one run
- ordinary incremental runs inserting only newly live hashes
- authoritative full snapshots emitting tombstones for missing hashes
- empty authoritative full snapshot tombstoning all live hashes
- `A > B > A` lifecycle
- idempotent retry behavior
- batched full snapshot within one run
- rejection of multi-run authoritative snapshots in v1
- root-table-only scope

## Open Questions Deferred From v1

- child-table ledger semantics
- support for authoritative snapshots stitched across multiple runs
- storage/performance optimization for very large ledger tables
- optional helper APIs for current-state projections
