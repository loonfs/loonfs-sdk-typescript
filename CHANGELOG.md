# SDK 0.5.0

Regenerated from LoonFS main e9ddb9567.

Breaking changes on the wire:

- Path and inode download responses replace `access` with required ordered
  `ranges`. Each `DownloadRange` carries `start_offset`, `length`, and its
  own signed `access`, and together the ranges cover the requested revision
  suffix across immutable extents. Transfer helpers read them in order and
  verify the whole value at the end.
- `create_download_by_inode` no longer takes `revision_no`. It authorizes a
  read of the current revision of a visible file inode, or of the revision a
  live snapshot captured when the request names `snapshot_id`. The new
  `create_revision_download_by_inode` authorizes a read of one retained
  revision. Both take their options in a JSON body: `snapshot_id` and
  `start_offset` for the first, and `start_offset` for the second. The inode
  and revision stay in the path. A request with options in the query string
  or no body returns 400. The client methods are `inodes.createDownload` and
  `inodes.createRevisionDownload`.
- `inodes.content` no longer takes `revision_no`. It calls the new
  `get_file_content_by_inode`, which reads the current revision of a visible
  file inode, or the revision a live snapshot captured when the request names
  `snapshot_id`. `inodes.revisionContent` reads one retained revision
  through `get_file_revision_content_by_inode`. The path read operation is
  `get_file_content`, and `files.content` keeps its name. The request types
  are `LoonFS.GetFileContentRequest`, `LoonFS.GetFileContentByInodeRequest`,
  and `LoonFS.GetFileRevisionContentByInodeRequest`.
- Every open upload session names `checksum_algorithm`, including
  `service_proxied`, and the field is required.
- `ChecksumAlgorithm` allows only `crc64nvme` and `crc32c`; `sha256` is
  gone. Every content reference, including inline writes, uploads, copies,
  restores, and appends, uses the deployment's store algorithm.
- A store shape offers a fixed set of upload modes: S3-shaped stores offer
  direct PUT and multipart, GCS-shaped stores offer direct PUT only, and the
  local store proxies. Helpers select from the advertised capabilities.
- Appends combine the base checksum with the appended bytes; the
  `not_supported` answer for a base without a digest is gone, and a base in
  another algorithm reports `namespace_corrupt`.
- `ListChangesResponse.through_seq` is `head_seq`, and it holds the
  namespace head the page was read at on every page. The old field held the
  head only on the final page. Resume a listing only from `next_after_seq`.
  A client that resumes from `head_seq` skips changes.
- Every cursor, binding version, and content token uses a new encoding. The
  values stay opaque. A value issued before the upgrade is rejected with
  `invalid_request`. Restart the listing, read the entry again, or read the
  upload session again to get a new value.
- The `BindingVersion` pattern is `^[A-Za-z0-9_-]+$` instead of lowercase
  hex.
- The `metadata` maintenance report replaces `wal_flush` and `reorganize`
  with the required `outcome` and `compaction_units` and the optional
  `skipped`. `outcome` is a `MetadataMaintenanceOutcome`, `caught_up` or
  `due`, and `skipped` lists `MetadataSkippedWork` values. The
  `WalFlushStepOutcome` and `ReorganizeStepOutcome` types are removed.
- The `metadata_compaction` and `grep_gc` maintenance kinds are removed,
  together with their request and response types and the
  `MetadataCompactionOutcome` types. A `metadata` run includes compaction,
  and a `gc` run includes grep index collection.
- The metadata maintenance request renames `max_wal_tail_segments` to
  `max_wal_tail_objects`. Namespace diagnostics rename `wal_tail_segments` to
  `wal_tail_objects`. GC reports rename the deleted object count
  `wal_segments` to `wal_objects`.
- The `Namespace` type is `NamespaceMetadata`.
- `ErrorDetails` no longer has `max_writer_sessions`. The server has no
  writer session limit.
- The grep index status types are `GrepIndexStatusActive`,
  `GrepIndexStatusBackfilling`, and `GrepIndexStatusDisabled`. They replace
  the three `GrepIndexLifecycle` types.
- `GrepIndexStatusActive.next_event_index` and
  `ObjectTransferAccessPresignedUrl.headers` are required. An omitted field
  always means the field does not apply.
- The capability document no longer has these keys:
  - `filesystem.namespaces.create`, `filesystem.namespaces.delete`, and
    `filesystem.namespaces.fork`
  - `filesystem.snapshots`
  - `query.grep.default_limit` and `query.grep.max_limit`
  - `snapshot.max_lifetime_ms`, `snapshot.max_ttl_ms`, and
    `snapshot.max_live_per_namespace`
- `create_checkpoint` and `delete_checkpoint` are removed, together with
  `maintenance.checkpoints.create`, `maintenance.checkpoints.delete`,
  `CreateCheckpointRequest`, and `DeleteCheckpointResponse`. A snapshot is the
  one kind of user pin, and the checkpoint listing shows a snapshot with the
  `user` owner and the snapshot's name. `CheckpointOwnerSnapshot`,
  `DeletedCheckpointsByOwner.snapshot`, and
  `NamespaceDiagnostics.live_checkpoints` are removed.
- The error codes `checkpoint_not_found`, `snapshot_gone`, and
  `snapshot_quota_exceeded` are removed. A read, fork, or extension of an
  expired snapshot returns `snapshot_not_found`, and so does a retrieve of a
  missing checkpoint. The server has no limit on live snapshots.
- `CreateSnapshotRequest.ttl_ms` is optional, and a snapshot created without
  it doesn't expire. `Snapshot.expires_at_ms` is optional. `ttl_ms` must be at
  least 1 on create and extend, and an extension never moves the expiry
  earlier.
- Live reads default to `eventual` consistency unless the server's
  `consistency` setting is `strong`. An eventual read uses the head that the
  server confirmed last while that confirmation is younger than the maximum
  staleness, so it can miss a recent commit. Set `consistency` to `strong` to
  confirm the head before the read.
- `ContentRef.owner_namespace_id` is removed. A fork family stores its
  content under the family root's prefix, and the required
  `NamespaceMetadata.family_root_id` names that root. A `content_token` is
  valid only in the namespace where its upload session was completed.
- `PathEntryFile.content_ref` is optional. Entry reads by path or by inode
  include it, and directory listings leave it out.
- Inode, content, and upload IDs are the prefix `ino_`, `con_`, or `upl_`
  followed by 26 lowercase Crockford base 32 characters. Inode IDs were
  decimal numbers, and content and upload IDs were 32 hex digits.
- `GrepIndexStatusActive`, `GrepIndexStatusBackfilling`, and
  `GrepIndexStatusDisabled` no longer have `next_run_no`, and the `RunNo`
  type is removed.
- `StoreProbeResponse.checks` holds `StoreProbeCheck` values, a union on
  `outcome` with the members `passed`, `unsupported`, and `failed`. A failed
  check has a `message`. `StoreProbeCheck` replaces `StoreProbeCheckResult`
  and `StoreProbeCheckOutcome`.
- The metadata maintenance request's `max_wal_tail_objects` must be at
  least 1.
- `@loonfs/sdk/client` is removed, together with its `LoonFSClient` export
  and the `namespace_alias` addressing of the `/v0/namespace-aliases/`
  routes. Import `LoonFSClient` from `@loonfs/sdk/server`. The same client
  works in a backend and in a browser. In a browser, set `baseUrl` to the
  application's proxy and `token` to the credential that the proxy's
  `authorize` hook checks. The proxy replaces that credential with the
  server token, so the server token stays in the backend.

Other changes:

- `snapshots.retrieve` (`get_snapshot`) returns one snapshot, including an
  expired one that garbage collection hasn't deleted yet. A missing id
  returns `snapshot_not_found`. `maintenance.checkpoints.retrieve`
  (`get_checkpoint`) returns one pin as the checkpoint listing returns it,
  and a missing id returns `snapshot_not_found`.
- The proxy forwards `GET` requests for one snapshot, so a browser
  application can retrieve a snapshot.
- Commits accept `append_file` (`FilesystemOperationAppendFile`) and
  `append_file_by_inode` (`FilesystemOperationAppendFileByInode`). Each one
  adds 1 byte to 256 KiB of inline content to a file as its next revision.
  An append over 256 KiB returns 413.
- Commits accept four inode-addressed operations:
  - `copy_by_inode` (`FilesystemOperationCopyByInode`)
  - `restore_revision_by_inode` (`FilesystemOperationRestoreRevisionByInode`)
  - `update_access_by_inode` (`FilesystemOperationUpdateAccessByInode`)
  - `update_attributes_by_inode` (`FilesystemOperationUpdateAttributesByInode`)
- Commits accept two inode-addressed preconditions:
  - `inode_binding` (`CommitPreconditionInodeBinding`)
  - `name_absence` (`CommitPreconditionNameAbsence`)
- `FilesystemOperationUndelete` can name its destination with the optional
  `destination_parent_inode_id` and `destination_display_name` instead of
  `destination_path`.
- `grep` takes an `inode_id` scope that limits matches to the inode's
  descendants. It cannot be combined with `path_prefix`.
- `RunMaintenanceRequestRetention` takes an optional target, `to_seq` or
  `cutoff_at_ms`, but not both. Without a target, a `retention` run advances
  the floor to the folded manifest head.
- The three download grant requests take an optional `start_offset`. A grant
  covers the revision from `start_offset` to its end.
- GC reports count deleted temporary objects in the required
  `DeletedObjectCounts.temporary_objects`.
- `StoreProbeResponse.run_id` has the type `ProbeRunId`. A run ID is `probe_`
  followed by 26 lowercase Crockford base 32 characters.
- The new error code `store_contention` (503) reports that a write to a
  namespace control object used every compare-and-swap attempt it was
  allowed. The response has no `Retry-After` header, and the client doesn't
  retry it. Back off before retrying the request.
- An error caused by a JSON body field names the field in `param` as a JSON
  Pointer, such as `/start_offset`. An invalid grep pattern reports `param`
  as `pattern`.
- `not_deleted` reports `expected_deletion_seq` and `actual_deletion_seq`
  when a live deletion exists at a different sequence.
- The store probe accepts JSON bodies up to 2 MiB.
- Newer servers may report other event kinds in `FilesystemChange`, and
  clients ignore them. Newer servers may also report other inode kinds in
  `PathEntry` and `TrashEntry`. Only the documentation changed. The generated
  unions are unchanged.
- Forks can be merged and rebased. `merge_fork` (`namespaces.merge`) merges a
  fork's commits into the namespace as one commit. `get_next_merge_range`
  (`namespaces.nextMergeRange`) reports where the fork's next merge starts, as
  a `NextMergeRangeResponse` union on `outcome` with the members `open`,
  `range_start_below_floor`, and `fork_point_below_floor`. `rebase_fork`
  (`namespaces.rebase`) replays a fork's unmerged commits into a new fork and
  reports the conflicting changes as `MergeConflict` values. A merge commit
  has `Commit.merge`, and a namespace that a rebase created has
  `NamespaceForkBasis.rebased_from`.
- `list_diff` (`diff.list`) returns a page of the inodes whose state differs
  between two sequences, as `DiffEntry` values with a `DiffEntryKind`.
- The new error codes are `merge_conflict` (409), `merge_range_conflict`
  (409), `merge_too_large` (413), and `range_not_satisfiable` (416).
  `ErrorDetails` adds `conflict`, `conflict_count`, `merge_bound`,
  `split_at_seq`, `fork_point_seq`, `from_seq`, `source_namespace_id`,
  `source_after_seq`, `expected_after_seq`, `actual_after_seq`, and
  `size_bytes`. The client throws `LoonFS.ForbiddenError` for 403 and
  `LoonFS.RangeNotSatisfiableError` for 416.
- `grep` returns `rebootstrap_required` (409) when a later page's read
  position is below the retention floor.
- Live reads take an optional `consistency` (`ReadConsistency`, `eventual`
  or `strong`) and `max_staleness_ms`. The three download grant requests
  take the same fields in their bodies.
- The three content reads take an optional `Range`, sent as the `Range`
  header, and return one byte range of the file as 206 with `Content-Range`. A
  range that starts at or past the end of the file returns
  `range_not_satisfiable`.
- `get_file_revision_content_by_inode` takes an optional `snapshot_id`.
- `list_path_entries` and `list_inode_children` take an optional
  `recursive`.
- `list_changes` takes `split_commits` and `event_index`, and
  `ListChangesResponse.next_event_index` resumes a page that ended inside a
  commit. The capability key is `filesystem.changes.split_commits`.
- `FilesystemChangeContentChanged.append_offset` reports where an append's
  bytes begin.
- A namespace can have a retention window. `CreateNamespaceRequest` and
  `NamespaceMetadata` have the optional `retention_window_ms`, and the
  `retention_window` maintenance kind sets or clears it.
- `NamespaceForkBasis.source_pin_id` is required. A read can pass it as
  `snapshot_id` to read the source namespace at the fork point.
- GC reports have the required `listed_bytes` and
  `DeletedObjectCounts.extension_segments`, and the optional `skipped`, a
  list of `GcSkippedWork` values.
- `delete_snapshot` is idempotent, and the client retries it.
- The capability key `filesystem.merges` reports whether the deployment
  serves merges.
- Documentation strings follow the current specification wording.

Handwritten layers:

- Downloads validate the range list before any signed request, read the
  ranges in order, check each range's length, and verify the complete file
  at the end. Resume pins the revision and verifies the retained prefix plus
  the downloaded suffix.
- Downloads keep up to four range responses open: the range being read and
  up to three requested after it. A file stored in many small extents no
  longer waits one round trip per range. Each download can use up to four
  connections. Early exit cancels pending requests and closes held
  responses. A failed request for a later range is reported when the reader
  reaches that range, after the earlier bytes are delivered. A failure in
  the stream aborts the transfer, so range requests in flight reject instead
  of running on.
- `files.append` appends 1 byte to 256 KiB to a file in one commit through
  `append_file`.
- The three languages share helper names, verification order, and error
  wording. SHA-256 code and optional-algorithm branches are removed.
- The proxy strips the browser's `Host` header.
- The proxy forwards `/v0/capabilities` and the namespace routes for files,
  uploads, snapshots, changes, and grep with the path unchanged, and it
  returns 404 for every other route. `namespaceAliases` and the
  `/v0/namespace-aliases/` routes are removed. The `authorize` hook receives
  the decoded namespace ID in `ProxyRouteContext.namespaceId`, and it checks
  that the caller may use that namespace.
- A proxied download whose entry read has no `content_ref` throws instead of
  reading.

# SDK 0.4.0

Regenerated from LoonFS v0.4.0. Targets LoonFS API v0.4.x.

Breaking changes on the wire:

- The binding token is `binding_version` on entries and change events,
  `expected_binding_version` on inode-addressed moves and deletes and on the
  `path_binding` precondition, and the mismatch error is
  `binding_version_mismatch` with `expected_binding_version` and
  `actual_binding_version` details. Tokens stay opaque.
- GC reports rename `reclaim_after_ms` to `reclaimable_at_ms`, and the
  `deleted_checkpoints_by_owner` counts are `user`, `snapshot`, and `fork`.
  `ErrorDetails` renames `active_writer` to `active_writer_id`. A backfilling
  grep index reports `captured_seq`.
- Grep index garbage collection runs through `run_maintenance` with kind
  `grep_gc`. The `gc_grep_index` operation and its request and response
  types are removed.
- The capability document renames these limit keys:
  - `upload.max_content_bytes` to `upload.service_proxied.max_content_bytes`
  - `upload.direct_put_max_content_bytes` to `upload.direct_put.max_content_bytes`
  - `upload.completion_max_body_bytes` to `upload.complete.max_request_body_bytes`
  - `upload.max_concurrent` to `upload.service_proxied.max_concurrent_requests`
  - `download.max_content_bytes` to `download.service_proxied.max_content_bytes`
  - `download.max_concurrent` to `download.service_proxied.max_concurrent_requests`
  - `commit.max_inline_content_bytes` to `commit.max_inline_content_bytes_per_operation`
  - `access.max_principals` to `access.max_principals_per_request`
- `PinId` replaces `CheckpointId` and `SnapshotId`. `AttributesRevisionNo`
  replaces `AttributeRevisionNo`. `BindingVersion` replaces
  `BindingGeneration`.
- `CheckpointOwner` replaces `CheckpointOwnerSummary`.

Breaking changes in the proxy:

- `createProxyHandler` requires the `authorize` hook and throws `TypeError`
  without it. Returning `{}` forwards as the token holder.

Other changes:

- Creating or forking into a deleted namespace id answers `namespace_deleted`
  (410), as it did in LoonFS 0.3.1. The error names the deleted namespace in
  `ErrorDetails.namespace_id`. For a fork, that is the source or the target.
- The nine operations that take a JSON body raise
  `LoonFS.ContentTooLargeError` (413) when the body exceeds 2 MiB.
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
