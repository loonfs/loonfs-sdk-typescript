# LoonFS TypeScript SDK

One package for LoonFS client, proxy, and server applications.

## Install

```sh
npm install @loonfs/sdk
```

Choose the entry point that matches where your code runs. There is
intentionally no default `@loonfs/sdk` import.

## Server

Use `@loonfs/sdk/server` in trusted server-side code that connects directly to
LoonFS.

```ts
import { LoonFSClient } from "@loonfs/sdk/server";

const client = new LoonFSClient({
    baseUrl: process.env.LOONFS_URL!,
    token: process.env.LOONFS_AUTH_TOKEN!,
});

const capabilities = await client.capabilities.retrieve();
```

`client.files.upload` and `client.files.download` transfer whole files in
memory.

## Client

Use `@loonfs/sdk/client` in untrusted application code. It talks to a LoonFS
proxy in your backend, which maps public namespace aliases and adds the server
credential.

```ts
import { LoonFSClient } from "@loonfs/sdk/client";

const client = new LoonFSClient({
    baseUrl: window.location.origin,
});

const entries = await client.files.list({
    namespace_alias: "team-files",
    path: "/",
});
```

`client.files.upload` and `client.files.download` work the same way through the
proxy. Never send a raw LoonFS server token to client code.

## Proxy

Use `@loonfs/sdk/proxy` in your backend to create a fetch-compatible handler
for client requests.

Set `authorize` to check each request and set `Loonfs-Actor` on forwarded
requests. The `authorize` hook is required; returning `{}` forwards as the token holder.
Here, `authorizedActor` checks the application's session and namespace access.

```ts
import { createProxyHandler } from "@loonfs/sdk/proxy";

const handle = createProxyHandler({
    serverBaseUrl: "https://loonfs.example.com",
    token: process.env.LOONFS_TOKEN!,
    namespaceAliases: {
        "team-files": "namespace_123",
    },
    authorize: async (request, route) => {
        const actorId = await authorizedActor(request, route.namespaceId);
        if (!actorId) {
            return new Response(null, { status: 403 });
        }
        return { actorId };
    },
});

const response = await handle(request);
```

The proxy streams uploads and downloads without retrying, caching, or changing
response bodies.

## Transfer helpers

Both clients expose the same `files` helpers:

| Helper | Behavior |
| --- | --- |
| `prepare` | Prepare bytes for publication. |
| `prepareStream` | Prepare a source once. |
| `upload` | Prepare and publish bytes. |
| `uploadStream` | Prepare and publish a stream. |
| `uploadPrepared` | Publish retained prepared content. |
| `download` | Read and verify a file in memory. |
| `downloadStream` | Read a file with bounded memory and verify at EOF. |
| `append` | Append 1 byte to 256 KiB in one commit. |

Each deployment uses one content checksum algorithm: `crc64nvme` or `crc32c`.
Every open upload session names the required `checksum_algorithm`, including
`service_proxied`. Callers do not choose the algorithm. Store capabilities offer
a fixed set of upload modes. S3 offers PUT and multipart; GCS offers PUT; local
storage uses proxied uploads. Helpers select from the advertised capabilities
and limits. Unknown sizes use multipart when available, otherwise proxied upload.

`prepare` and `upload` accept `Uint8Array`. Their streaming forms accept a
`Blob`, `ReadableStream<Uint8Array>`, or `AsyncIterable<Uint8Array>`.
Pass `size_bytes` when known; a Blob supplies its size. Small content stays
inline when advertised, up to the smaller of the server limit and 64 KiB.
Lookahead retains that limit plus one byte and reuses the prefix during upload.
Multipart retains one provider-sized part and at most 10,000 part descriptors.
Sources should yield bounded chunks; the helper reads at most 64 KiB at a time.
Known single-request bodies up to 8 MiB are buffered for browser support.
Larger or unknown bodies require a runtime that supports streaming requests.

Preparation returns `PreparedFile`. `InlinePreparedContent` holds immutable
`inlineContent`; `PreparedContent` holds `contentRef` and optional `contentToken`.
Preparation does not publish a file or extend the upload lifetime. Publishing
returns the commit and its events. For publication retries, retain the prepared
value and use the same `commit_id`, path, actor, preconditions, and other inputs.
Calling `upload` again prepares content again.

Source and payload failures abort without replaying bytes. A failed completion
response leaves the session available for inspection. Upload sources are consumed
once; cancellation returns their iterator or cancels their stream.

Append uses `append_file` in one commit, without an upload or content token.
Empty content and content over 256 KiB fail before any request. Supply a stable
commit ID and identical inputs for retries. Inode and revision preconditions are
optional; a path revision precondition requires the inode precondition. The
generated commit API also accepts `append_file_by_inode`.

`append` accepts `content`, `commit_id`, `expected_inode_id`, and
`expected_revision_no`, along with the namespace and path.

## Downloads and resume

`downloadStream` returns a `ReadableStream<Uint8Array>` in `content`. Consume it
through successful EOF to verify it. Cancel its reader on early exit. Bytes
already consumed are unverified until EOF succeeds. The request's abort signal
and timeout cover metadata and body reads. The default deadline is 60 seconds;
`uploadStream` keeps that deadline through publication.

Direct grants contain ordered `ranges`. Each range has a revision `start_offset`,
`length`, and signed `access`. Helpers validate the range list before any object
request, read ranges in order, and check each length before opening the next.
They send the signed headers unchanged. The object Range header can differ from
the revision offset. An empty file needs no object request. Successful EOF checks
the total length, then the checksum of the complete file.

The download helpers read complete files. To resume through the generated grant
API, pin `revision_no`, retain the bytes already read, and request a new grant
with `start_offset` equal to that prefix's length. Verify prefix plus suffix
against the content reference. Checking only the suffix cannot verify the file.
A complete local file needs no new grant.

Direct requests carry only signed headers and never follow redirects. Proxied
reads use API authorization and are pinned to the selected revision.

## Retries

The client and server SDKs retry transient failures on operations that are safe
to repeat. Operations that LoonFS classifies as non-idempotent are never
retried automatically.

## Generated code

The client and server SDKs are generated from the LoonFS OpenAPI
specifications. Please report SDK issues in the
[main LoonFS repository](https://github.com/loonfs/loonfs).

## License

Apache-2.0.
