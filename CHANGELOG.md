# SDK 0.4.0

Regenerated from LoonFS `dee06fcc4ad5d3d5122ff9b2d127bf7cd5db8324`. Targets LoonFS v0.4.x.

Breaking changes on the wire:

- Namespace generations are gone. A namespace id has one lifetime, and
  deleting it makes the id unusable. `Namespace.generation`,
  `NamespaceDiagnostics.generation`, `ContentRef.owner_generation`, the
  `NamespaceGeneration` type, and the GC counts for retired generation records
  are removed. Creating a namespace answers `namespace_deleted` (410) for a
  deleted id, and `ErrorDetails.namespace_id` names the namespace.
- The binding token is `binding_version` on entries and change events,
  `expected_binding_version` on inode-addressed moves and deletes and on the
  `path_binding` precondition, and the mismatch error is
  `binding_version_mismatch` with `expected_binding_version` and
  `actual_binding_version` details. Tokens stay opaque.
- GC reports rename `reclaim_after_ms` to `reclaimable_at_ms`, and the
  `deleted_checkpoints_by_owner` counts are `user`, `snapshot`, and `fork`.
  Namespace diagnostics name the writer as `active_writer_id`. A backfilling
  grep index reports `captured_seq`.
- Grep index garbage collection runs through `run_maintenance` with kind
  `grep_gc`. The `gc_grep_index` operation and its request and response
  types are removed.
- Capability limit names changed. `PinId` replaces `CheckpointId` and `SnapshotId`;
  `AttributesRevisionNo` replaces `AttributeRevisionNo`.

Other changes:

- Snapshot and checkpoint deletes are not retried after a transport error,
  since a delete that landed would otherwise report not found. Callers handle
  transport errors on those calls themselves.
- The maintenance job kinds include `recover_administrator`.
- Documentation strings follow the current specification wording.
- Server and browser clients clear request timers after failures and report
  SDK timeouts as `LoonFSTimeoutError`. Caller cancellation remains distinct.

# SDK 0.3.0

Regenerated from LoonFS `6ea950f9f5e0030066e18061efe787dcaf9be4db` (PR #996 plus the
Python compatibility fix in PR #1000). Targets LoonFS API v0.3.x.

Small-file helpers now prepare content inline when advertised, up to the smaller
of the server inline limit and 64 KiB. Streaming detection has bounded lookahead
and preserves bytes when falling back to normal uploads. Publication retries
must reuse the prepared value, commit ID, and all other publication inputs.

## Migration

Preparation now returns `PreparedFile`: inline content or the existing staged
`PreparedContent`. Passing the value directly to prepared publication continues
to work. Callers that annotate the result or inspect staged reference/token
fields must accept the union and narrow to the staged variant first.

This regeneration also includes the current API's access-control models and
subject/principal context support, including proxy header filtering. Existing
clients do not need to configure subject context; when supplied, principal scope
and principals must be configured together.
