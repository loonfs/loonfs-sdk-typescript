# LoonFS TypeScript SDK

One package for LoonFS client, proxy, and server applications. SDK v0.2.x
targets LoonFS API v0.3.x.

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

```ts
import { createProxyHandler } from "@loonfs/sdk/proxy";

const handle = createProxyHandler({
    serverBaseUrl: "https://loonfs.example.com",
    token: process.env.LOONFS_TOKEN!,
    namespaceAliases: {
        "team-files": "namespace_123",
    },
});

const response = await handle(request);
```

The proxy streams uploads and downloads without retrying, caching, or changing
response bodies.

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
