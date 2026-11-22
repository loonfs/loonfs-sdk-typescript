# Unreleased

Regenerated from LoonFS d884e8165. The API version stays 0.6.0.

Breaking changes:

- The store probe types are `ProbeStoreRequest` and `ProbeStoreResponse`, and
  the diff listing type is `ListDiffResponse`. They replace
  `StoreProbeRequest`, `StoreProbeResponse`, and `DiffResponse`, which are
  removed. `maintenance.store.probe` and `diff.list` return the new types. The
  JSON is unchanged.

Other changes:

- `namespaces.list` (`list_namespaces`, `GET /v0/namespaces`) lists the
  store's namespaces in object-key order and returns a
  `ListNamespacesResponse`. Each page lists up to `limit` namespace IDs from
  the store, 100 by default and 1,000 at most, and leaves out deleted
  namespaces and namespaces whose creation hasn't finished, so a page can have
  fewer namespaces than `limit` and still include `next_cursor`.
- `files.listRevisions` and `inodes.listRevisions` take an optional
  `snapshot_id`. The listing then reads the snapshot's state and starts at the
  revision that the snapshot captured. A request can't combine it with
  `consistency`.
- The three `NextMergeRangeResponse` outcomes include the required
  `namespace_id` and `source_namespace_id`.
- `ReplayForkResponse.commit` is required and contains the replay commit. A
  retry with the same commit ID returns the same commit.
- Path parameters use the ID types `LoonFS.NamespaceId`, `LoonFS.InodeId`,
  `LoonFS.SnapshotId`, `LoonFS.UploadId`, and `LoonFS.PoolId`, which are
  aliases for `string`.
- `create_namespace`, `fork_namespace`, `create_snapshot`,
  `create_download_by_inode`, `create_revision_download_by_inode`, and
  `probe_store` accept a request with no body, which uses the defaults.
- A request that repeats `Loonfs-Actor`, `Loonfs-Principals`, or
  `Loonfs-Context` returns 400 `invalid_request`.
- More routes declare their 403, 409, 410, and 501 responses, and the client
  throws the matching typed error for them, such as `LoonFS.ForbiddenError`
  for 403.

# SDK 0.6.0

Regenerated from LoonFS da8a28cba, which sets the API version to 0.6.0. This
release follows SDK 0.4.0, and no SDK 0.5.0 was published.

Breaking changes on the wire:

- Path and inode download responses replace `access` with required ordered
  `ranges`. Each `DownloadRange` includes `start_offset`, `length`, and its
  own presigned URL in `transfer`, and together the ranges cover the requested
  revision suffix across immutable extents. Transfer helpers read them in
  order and verify the whole value at the end.
- `create_download_by_inode` no longer takes `revision_no`. It authorizes a
  read of the current revision of a visible file inode, or of the revision a
  live snapshot captured when the request includes `snapshot_id`. The new
  `create_revision_download_by_inode` authorizes a read of one retained
  revision. Both take their options, `snapshot_id` and `start_offset`, in a
  JSON body. The inode and revision stay in the path. A request with options
  in the query string or no body returns 400. The client methods are
  `inodes.createDownload` and `inodes.createRevisionDownload`.
- `inodes.content` no longer takes `revision_no`. It calls the new
  `get_file_content_by_inode`, which reads the current revision of a visible
  file inode, or the revision a live snapshot captured when the request
  includes `snapshot_id`. `inodes.revisionContent` reads one retained revision
  through `get_file_revision_content_by_inode`. The path read operation is
  `get_file_content`, and `files.content` keeps its name. The request types
  are `LoonFS.GetFileContentRequest`, `LoonFS.GetFileContentByInodeRequest`,
  and `LoonFS.GetFileRevisionContentByInodeRequest`.
- Every open upload session includes `checksum_algorithm`, including
  `service_proxied`, and the field is required.
- `ChecksumAlgorithm` allows `crc64nvme` and `crc32c`, and `sha256` is
  removed. Every content reference, including inline writes, uploads, copies,
  restores, and appends, uses the deployment's store algorithm.
- The upload modes depend on the store shape. S3-shaped stores offer direct
  PUT and multipart, GCS-shaped stores offer direct PUT, and the local store
  proxies uploads. Helpers select from the advertised capabilities.
- Appends combine the base checksum with the appended bytes. An append to a
  base without a digest no longer returns `not_supported`, and a base in
  another algorithm reports `namespace_corrupt`.
- `ListChangesResponse.through_seq` is `head_seq`, and it contains the
  namespace head the page was read at on every page. The old field reported
  the head on the final page alone. Resume a listing only from
  `next_after_seq`. A client that resumes from `head_seq` skips changes.
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
  `MetadataCompactionOutcome` types. A `metadata` job includes compaction, and
  a `gc` job includes grep index collection.
- The metadata maintenance request renames `max_wal_tail_segments` to
  `max_wal_tail_objects`. Namespace diagnostics rename `wal_tail_segments` to
  `wal_tail_objects`. GC reports rename the deleted object count
  `wal_segments` to `wal_objects`.
- The `Namespace` type is `NamespaceMetadata`.
- `ErrorDetails` no longer has `max_writer_sessions`. The server has no writer
  session limit. The `writer_capacity_exceeded` error code is removed.
- The grep index status types are `GrepIndexStatusActive`,
  `GrepIndexStatusBackfilling`, and `GrepIndexStatusDisabled`. They replace
  the three `GrepIndexLifecycle` types.
- `GrepIndexStatusActive.next_event_index` and
  `ObjectTransferPresignedUrl.headers` are required. An omitted field always
  means the field does not apply.
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
  one kind of user pin. `NamespaceDiagnostics.live_checkpoints` is removed.
- The error codes `checkpoint_not_found`, `snapshot_gone`, and
  `snapshot_quota_exceeded` are removed. A read, fork, or extension of an
  expired snapshot returns `snapshot_not_found`, and so does a retrieve of a
  missing snapshot record. The server has no limit on live snapshots.
- `CreateSnapshotRequest.ttl_ms` is optional, and a snapshot created without
  it doesn't expire. `Snapshot.expires_at_ms` is optional. `ttl_ms` must be at
  least 1 on create and extend, and an extension never moves the expiry
  earlier.
- Live reads default to `eventual` consistency unless the server's
  `consistency` setting is `strong`. An eventual read uses the head that the
  server confirmed last while that confirmation is younger than the maximum
  staleness, so it can miss a recent commit. Set `consistency` to `strong` to
  confirm the head before the read.
- `ContentRef.owner_namespace_id` is removed. A namespace and its forks
  share one pool, which stores their content, and the required
  `NamespaceMetadata.pool_id` identifies it. A pool ID (`PoolId`) is `pool_`
  followed by 26 lowercase Crockford base 32 characters. A `content_token` is
  valid only in the namespace where its upload session was completed.
- A namespace's `gc` report no longer covers content. The deleted counts
  `content_objects`, `upload_sessions`, and `retired_content_objects` and the
  retained counts `upload_session_window` and `upload_session_undecided` are
  removed from `DeletedObjectCounts` and `RetainedCandidates`. A pool's `gc`
  job reports content, upload sessions, and temporary objects.
- `PathEntryFile.content_ref` is optional. A directory listing leaves it out
  for a file that the caller can't read.
- Inode, content, and upload IDs are the prefix `ino_`, `con_`, or `upl_`
  followed by 26 lowercase Crockford base 32 characters. Inode IDs were
  decimal numbers, and content and upload IDs were 32 hex digits.
- `GrepIndexStatusActive`, `GrepIndexStatusBackfilling`, and
  `GrepIndexStatusDisabled` no longer have `next_run_no`, and the `RunNo`
  type is removed.
- `StoreProbeResponse.checks` contains `StoreProbeCheck` values, a union on
  `outcome` with the members `passed`, `unsupported`, and `failed`. A failed
  check has a `message`. `StoreProbeCheck` replaces `StoreProbeCheckResult`
  and `StoreProbeCheckOutcome`.
- The metadata maintenance request's `max_wal_tail_objects` must be at
  least 1.
- The maintenance routes end in `/jobs` instead of `/runs`.
  `run_maintenance_job` (`maintenance.jobs.create`,
  `POST /v0/maintenance/namespaces/{namespace_id}/jobs`) replaces
  `run_maintenance` and `maintenance.runs.create`. `RunMaintenanceJobRequest`,
  `RunMaintenanceJobResponse`, and their members, such as
  `RunMaintenanceJobRequestGc`, replace the `RunMaintenanceRequest` and
  `RunMaintenanceResponse` types.
- `ChangeSeq` is `CommitSeq`. `Commit.events` lists `FilesystemEvent` values,
  which replace the `FilesystemChange` types. The `undeleted` event is
  `restored` (`FilesystemEventRestored`), and the `undelete` commit operation
  is `restore_deleted` (`FilesystemOperationRestoreDeleted`).
- `ObjectTransfer` and `ObjectTransferPresignedUrl` replace
  `ObjectTransferAccess` and `ObjectTransferAccessPresignedUrl`.
  `SignedUploadPart` and `UploadSessionStatusOpen` have `transfer` instead of
  `access`.
- `StoreProbeResponse.run_id` is `probe_id`, with the type `ProbeId`, which
  replaces `ProbeRunId`. A probe ID is `probe_` followed by 26 lowercase
  Crockford base 32 characters.
- `PinId` is `SnapshotId`. A snapshot ID is `snp_` followed by 26 lowercase
  Crockford base 32 characters.
- `NamespaceForkBasis.source_head_seq` is `fork_point_seq`.
- The maintenance checkpoint inventory is the snapshot record inventory.
  `list_snapshot_records` (`maintenance.snapshots.list`) and
  `get_snapshot_record` (`maintenance.snapshots.retrieve`) replace
  `list_checkpoints` and `get_checkpoint`. The routes are
  `GET /v0/maintenance/namespaces/{namespace_id}/snapshots` and
  `GET /v0/maintenance/namespaces/{namespace_id}/snapshots/{snapshot_id}`.
  `SnapshotRecord`, `SnapshotRecordOwner`, `SnapshotRecordOwnerUser`,
  `SnapshotRecordOwnerFork`, and `ListSnapshotRecordsResponse` replace the
  `Checkpoint` types and `ListCheckpointsResponse`. The inventory lists a user
  snapshot with the `user` owner, and `CheckpointOwnerSnapshot` is removed. A
  namespace's `gc` report has `deleted_snapshots_by_owner`
  (`DeletedSnapshotsByOwner`) instead of `deleted_checkpoints_by_owner`, and
  `RetainedCandidates.snapshot_not_deletable` replaces
  `checkpoint_not_deletable`. `GrepIndexStatusBackfilling` no longer has
  `checkpoint_id`.
- `InodeKind` and `PathEntryDirectory.inode_kind` spell a directory as
  `directory` instead of `dir`.
- A namespace's `gc` report has `retires_at_ms`, the deletion time of the
  current tombstone plus the grace window, instead of `reclaimable_at_ms`.
- `RunMaintenanceJobRequestGc` no longer takes `grace_window_ms`. Every
  collection uses one grace window of 24 hours.
- `Snapshot.name` and `CreateSnapshotRequest.name` are optional, and a merge
  plan's snapshot has no name.
- A namespace no longer has an access mode. Every request that reads or
  changes a namespace's contents sends exactly one of `Loonfs-Principals`,
  which lists the principals that the request acts for, and
  `Loonfs-Context: service`, which acts in the service context, where the
  server checks no rights. A request with neither or both returns
  `invalid_request`. Set `principals` or `loonfsContext` in the client
  options.
- `CreateNamespaceRequest.access` and `NamespaceMetadata.access` are removed,
  together with the `NamespaceAccess` and `NamespaceAccessMode` types,
  `PrincipalId`, and `PrincipalScope`. A new namespace has no access rows.
  Grant its first administrators with a commit whose one operation is
  `update_access` on `/`, sent in the service context.
- The `Loonfs-Subject` and `Loonfs-Principal-Scope` headers are removed,
  together with the client options `subjectId` and `principalScope`.
  `Loonfs-Actor` alone identifies who made a request.
- Every upload operation requires `Loonfs-Actor`. Set `actorId` on a client
  that uploads. An upload session belongs to the actor that opened it, and a
  request from another actor returns `upload_not_found`.
- The `recover_administrator` maintenance job is removed, together with its
  request and response types.
- Each error code has one remedy, and the optional `ErrorDetails.cause`
  (`ErrorCause`) reports which condition occurred when two conditions share a
  code. These codes changed:
  - `below_retention_floor` replaces `rebootstrap_required`.
  - `outcome_unknown` replaces `commit_outcome_unknown` and
    `deadline_exceeded`. Its cause is `unconfirmed` or `deadline`.
  - `checkpoint_unavailable` and `namespace_unrestricted` are removed. A
    snapshot whose manifest is gone returns `snapshot_not_found` with the
    cause `manifest_missing`.
  - `wrong_kind` (409) reports a directory where the operation needs a file,
    or a file where the operation needs a directory.
  - `commit_too_large` (413) reports a commit that is larger than one WAL
    object.
  - `store_contention` with the cause `publication_budget` reports a metadata
    publication that stopped at its time budget. `maintenance_required` now
    reports a namespace at its write stop and no other condition.
- `grep`'s `path_prefix` parameter is `path`.
- `include_attributes` defaults to `false` on `get_path_entry` and
  `get_inode`. It defaulted to `true`.
- `@loonfs/sdk/client` is removed, together with its `LoonFSClient` export
  and the `namespace_alias` addressing of the `/v0/namespace-aliases/`
  routes. Import `LoonFSClient` from `@loonfs/sdk/server`. The same client
  works in a backend and in a browser. In a browser, set `baseUrl` to the
  application's proxy and `token` to the credential that the proxy's
  `authorize` hook checks. The proxy replaces that credential with the
  server token, so the server token stays in the backend.

Other changes:

- `snapshots.retrieve` (`get_snapshot`) returns one snapshot, as the snapshot
  listing returns it. A missing ID, the ID of an expired or deleted snapshot,
  or the ID of a fork snapshot returns `snapshot_not_found`. The snapshot
  listing leaves out expired and deleted snapshots.
  `maintenance.snapshots.retrieve` (`get_snapshot_record`) returns one
  snapshot record of any owner, including an expired or deleted snapshot that
  garbage collection hasn't deleted, and a missing ID returns
  `snapshot_not_found`.
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
- `FilesystemOperationRestoreDeleted` can identify its destination with the
  optional `destination_parent_inode_id` and `destination_display_name`
  instead of `destination_path`.
- `grep` takes an `inode_id` scope that limits matches to the inode's
  descendants. It cannot be combined with `path`.
- `RunMaintenanceJobRequestRetention` takes an optional target, `to_seq` or
  `cutoff_at_ms`, but not both. Without a target, a `retention` job advances
  the floor to the folded manifest head.
- The three download grant requests take an optional `start_offset`. A grant
  covers the revision from `start_offset` to its end.
- The new error code `store_contention` (503) reports that a write to a
  namespace control object used every compare-and-swap attempt it was
  allowed. The response has no `Retry-After` header, and the client doesn't
  retry it. Back off before retrying the request.
- An error caused by a JSON body field reports the field in `param` as a JSON
  Pointer, such as `/start_offset`. An invalid grep pattern reports `param` as
  `pattern`.
- `not_deleted` reports `expected_deletion_seq` and `actual_deletion_seq`
  when a live deletion exists at a different sequence.
- The store probe accepts JSON bodies up to 2 MiB.
- Newer servers may report other event kinds in `FilesystemEvent`, and clients
  ignore them. Newer servers may also report other inode kinds in `PathEntry`
  and `TrashEntry`. Only the documentation changed. The generated unions are
  unchanged.
- Forks can be merged and rebased. `merge_fork` (`namespaces.merge`) merges a
  fork's commits into the namespace as one commit. `get_next_merge_range`
  (`namespaces.nextMergeRange`) reports where the fork's next merge starts, as
  a `NextMergeRangeResponse` union on `outcome` with the members `open`,
  `range_start_below_floor`, and `fork_point_below_floor`. A rebase takes two
  calls. `fork_namespace_for_rebase` (`namespaces.forkForRebase`,
  `POST /v0/namespaces/{namespace_id}/rebases`) creates the rebase's new fork
  from the namespace's head. `replay_fork` (`namespaces.replay`,
  `POST /v0/namespaces/{namespace_id}/replays`) replays the fork's unmerged
  commits into the new fork and reports the conflicting changes as
  `MergeConflict` values. A merge commit has `Commit.merge`, and a namespace
  that a rebase created has `NamespaceForkBasis.rebased_from`.
- `list_diff` (`diff.list`) returns a page of the inodes whose state differs
  between two sequences, as `DiffEntry` values with a `DiffEntryKind`.
- The new error codes are `merge_conflict` (409), `merge_range_conflict`
  (409), `merge_too_large` (413), and `range_not_satisfiable` (416).
  `ErrorDetails` adds `conflict`, `conflict_count`, `merge_bound`,
  `split_at_seq`, `fork_point_seq`, `from_seq`, `source_namespace_id`,
  `source_after_seq`, `expected_after_seq`, `actual_after_seq`, and
  `size_bytes`. The client throws `LoonFS.ForbiddenError` for 403 and
  `LoonFS.RangeNotSatisfiableError` for 416.
- `grep` returns `below_retention_floor` (409) when a later page's read
  position is below the retention floor.
- Live reads take an optional `consistency` (`ReadConsistency`, `eventual` or
  `strong`) and `max_staleness_ms`, which is at most 1200000 (20 minutes). The
  three download grant requests take the same fields in their bodies. `grep`,
  `get_namespace`, and `get_next_merge_range` take them as query parameters.
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
- `FilesystemEventContentChanged.append_offset` reports where an append's
  bytes begin.
- A namespace can have a retention window. `CreateNamespaceRequest` and
  `NamespaceMetadata` have the optional `retention_window_ms`, and the
  `retention_window` maintenance job sets or clears it. The window must be at
  least the grace window (86400000, which is 24 hours). The capability key
  `namespace.default_retention_window_ms` reports the window that a namespace
  created without one receives, and a deployment without a default window
  leaves the key out.
- `NamespaceForkBasis.fork_point_snapshot_id` is required. A read of the fork
  can pass it as `snapshot_id` to read the source's state at the fork point.
- A namespace's `gc` report has the required `listed_bytes` and
  `DeletedObjectCounts.extension_segments`.
- `run_pool_maintenance_job` (`maintenance.pools.jobs.create`) runs one
  maintenance job on a pool. The route is
  `POST /v0/maintenance/pools/{pool_id}/jobs`. The one kind is `gc`
  (`RunPoolMaintenanceJobRequestGc`). Its result,
  `RunPoolMaintenanceJobResponseGc`, reports the deleted content objects,
  upload sessions, and temporary objects (`PoolDeletedObjectCounts`), the
  retained candidates (`PoolRetainedCandidates`), `listed_bytes`, the optional
  `next_reclamation_at_ms`, and the optional `skipped`, a list of
  `GcSkippedWork` values (`content_deletion_pool_budget` or
  `content_deletion_unreadable_member`).
- The new error code `pool_not_found` (404) reports a pool that has no pool
  record.
- `delete_snapshot` records the server's current time as the snapshot's delete
  time. From then on, the snapshot can't be read, forked, extended, or
  retrieved. `SnapshotRecord.deleted_at_ms` reports the delete time, and
  garbage collection deletes the record one grace window later. The request is
  idempotent, and the client retries it.
- The capability key `filesystem.merges` reports whether the deployment
  serves merges.
- `CreateNamespaceRequest.namespace_id` and
  `ForkNamespaceRequest.new_namespace_id` are optional. Without one, LoonFS
  generates the new namespace's ID, and the response's `namespace_id` reports
  the generated ID.
- A namespace's `gc` report has the required
  `RetainedCandidates.fences_and_floor_objects`, the number of fences and
  floor objects that the collection kept. No collection deletes these objects,
  and the collection tells them apart from data WAL objects by their listed
  size.
- `get_snapshot` and `list_snapshots` return `namespace_deleted` (410) for a
  deleted namespace. The client throws `LoonFS.GoneError`.
- Reads and `grep` no longer return `stale_head`. When garbage collection
  removes the segments that a read started from, the server runs the read
  again from the current manifest, and it returns `store_contention` after 8
  attempts.
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
- The proxy's `authorize` hook returns an authorization object with
  `principals`, to act for a user, or with `service: true`, to act in the
  service context. The object no longer has `subjectId` or `principalScope`.
  An authorization with neither choice no longer forwards as the token holder,
  and LoonFS rejects the forwarded request with `invalid_request`. The proxy
  removes the browser's `Loonfs-Actor`, `Loonfs-Principals`, and
  `Loonfs-Context` headers.

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
