# SDK 0.4.1

Regenerated from LoonFS main after v0.4.0.

Breaking changes on the wire:

- Metadata maintenance reports rename `wal_flush` to `wal_fold` and
  `reorganize` to `compaction`. The types are `WalFoldStepOutcome` and
  `CompactionStepOutcome`. The fold outcome `flushed` is `folded`, and the
  compaction outcome `compaction_required` is `metadata_compaction_required`.
- The metadata maintenance request renames `max_wal_tail_segments` to
  `max_wal_tail_objects`. Namespace diagnostics rename `wal_tail_segments` to
  `wal_tail_objects`. GC reports rename the deleted object count
  `wal_segments` to `wal_objects`.
- The `Namespace` type is `NamespaceMetadata`.
- `ErrorDetails` no longer has `max_writer_sessions`. The server has no
  writer session limit.
- `create_download_by_inode` no longer takes `revision_no`. It authorizes a
  read of the current revision of a visible file inode, or of the revision a
  live snapshot captured when the request names `snapshot_id`. The new
  `create_revision_download_by_inode` authorizes a read of one retained
  revision. The client methods are `inodes.createDownload` and
  `inodes.createRevisionDownload`.
- `inodes.content` no longer takes `revision_no`. It calls the new
  `get_file_bytes_by_inode`, which reads the current revision of a visible
  file inode, or the revision a live snapshot captured when the request names
  `snapshot_id`. `inodes.revisionContent` reads one retained revision
  through `get_file_revision_bytes_by_inode`.
- A download grant reads exactly `[start_offset, size_bytes)` of the
  revision. It signs that range, and `access.headers` carries it as
  `range`. A client sends the grant's headers unchanged and adds no `Range`
  of its own, so one grant no longer serves ranged, resumed, or parallel
  reads. A client that resumes asks for a new grant with `start_offset`. A
  grant for a revision of zero bytes signs no range. The download helpers
  already send the grant's headers unchanged.

Other changes:

- Namespaces have a naming mode that sets how sibling names compare.
  `NamespaceNaming` is `case_insensitive` or `case_sensitive`, and it is fixed
  when the namespace is created. `CreateNamespaceRequest.naming` is optional
  and defaults to `case_insensitive`. `NamespaceMetadata.naming` is required.
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
- Commits accept two append operations. Each adds bytes to the end of a file
  as its next revision:
  - `append_file` (`FilesystemOperationAppendFile`)
  - `append_file_by_inode` (`FilesystemOperationAppendFileByInode`)
- `append_file` takes `path` and `inline_content`, and optionally
  `expected_inode_id` and `expected_revision_no`. `append_file_by_inode`
  takes `inode_id` and `inline_content`, and optionally
  `expected_revision_no`. `inline_content` carries 1 byte to 256 KiB as
  base64. A larger append raises `LoonFS.ContentTooLargeError` (413). An
  append to content with no recorded digest to continue raises
  `LoonFS.NotImplementedError` (501).
- `files.append` adds 1 byte to 256 KiB to the end of a file in one
  `create_commit` request with an `append_file` operation, so an append needs
  no upload. Server and browser clients both have it, and
  `@loonfs/sdk/server` and `@loonfs/sdk/client` export its `AppendInput`. It
  refuses empty content and content over 256 KiB before it sends anything.
  Pass `commit_id` explicitly if you may retry. `append_file_by_inode` has no
  helper and goes through `commits.create`.
- Download grants take an optional `start_offset`, the first byte the grant
  reads. It defaults to 0 and must be below the revision's size, except 0
  for a revision of zero bytes. `CreateDownloadRequest.start_offset` is a
  body field, and `create_download_by_inode` and
  `create_revision_download_by_inode` take it as a query parameter. The
  client methods are `files.createDownload`, `inodes.createDownload`, and
  `inodes.createRevisionDownload`.
- The download helpers make no request for a grant of zero bytes. They
  return empty content.
- GC reports add the deleted object count `temporary_objects`, for store
  temporary objects deleted once older than the grace window.
  `content_objects` now counts content objects that no retained view names,
  deleted once older than the grace window. Before, it counted content
  reclaimed through completed upload sessions.
- Newer servers may report other event kinds in `FilesystemChange`, and
  clients ignore them. Newer servers may also report other inode kinds in
  `PathEntry` and `TrashEntry`. Only the documentation changed. The generated
  unions are unchanged.
- Documentation strings follow the current specification wording.

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
