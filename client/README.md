# LoonFS browser client

`@loonfs/sdk-client` is the browser-safe LoonFS client. It talks to a LoonFS
proxy in your backend, which maps public namespace aliases and adds the server
credential. SDK v0.1.x targets LoonFS API v0.3.x.

## Install

```sh
npm install @loonfs/sdk-client
```

## Usage

```ts
import { LoonFSClient } from "@loonfs/sdk-client";

const client = new LoonFSClient({
    environment: window.location.origin,
});

const entries = await client.filesystem.listPathEntries({
    namespace_alias: "team-files",
    path: "/",
});
```

Upload and download helpers are exported from `@loonfs/sdk-client/transfers`.
Pair this package with [`@loonfs/sdk-proxy`](../proxy); never send the raw
LoonFS server token to the browser.

## Retries

The client retries transient failures on operations that are safe to repeat.
Operations that LoonFS classifies as non-idempotent are never retried
automatically.

## Generated code

This SDK is generated from the LoonFS proxy OpenAPI specification. Please report
SDK issues in the [main LoonFS repository](https://github.com/loonfs/loonfs).

## License

Apache-2.0.
