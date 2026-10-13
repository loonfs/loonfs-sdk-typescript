# SDK 0.5.0

Regenerated from LoonFS main 76ba9db01.

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
- Metadata maintenance reports rename `wal_flush` to `wal_fold` and
  `reorganize` to `compaction`. The types are `WalFoldStepOutcome` and
  `CompactionStepOutcome`. The fold outcome `flushed` is `folded`. The
  compaction outcome `compaction_required` is `metadata_compaction_required`,
  and `unit_published` is `bounded_merge_published`. The type is
  `LoonFS.CompactionStepOutcomeBoundedMergePublished`, and the union member
  is `LoonFS.CompactionStepOutcome.BoundedMergePublished`.
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
- The capability limit `snapshot.max_lifetime_ms` is `snapshot.max_age_ms`.

Other changes:

- `snapshots.retrieve` (`get_snapshot`) returns one live snapshot. A missing
  id returns `snapshot_not_found`, and an expired snapshot returns
  `snapshot_gone`. Both clients provide it. On the server client,
  `maintenance.checkpoints.retrieve` (`get_checkpoint`) returns one pin as
  the checkpoint listing returns it.
- The proxy forwards `GET` requests for one snapshot, so browser clients can
  retrieve a snapshot.
- Namespaces have a naming mode that sets how sibling names compare.
  `NamespaceNaming` is `case_insensitive` or `case_sensitive`, and it is fixed
  when the namespace is created. `CreateNamespaceRequest.naming` is optional
  and defaults to `case_insensitive`. `NamespaceMetadata.naming` is required.
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
  followed by 32 lowercase hex digits.
- The new error code `store_contention` (503) reports that a write to a
  namespace control object used every compare-and-swap attempt it was
  allowed. The response includes `Retry-After: 1`, and the client retries
  it under the same rules as `server_busy`.
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
  `append_file`. Both clients provide it.
- The three languages share helper names, verification order, and error
  wording. SHA-256 code and optional-algorithm branches are removed.
- The browser proxy strips the browser's `Host` header.

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
