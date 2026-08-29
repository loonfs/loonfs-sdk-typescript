# LoonFS TypeScript SDK

The trusted server-side client for the LoonFS HTTP API. SDK v0.1.x targets
LoonFS API v0.3.x.

## Install

```sh
npm install @loonfs/sdk
```

## Usage

```ts
import { LoonFSClient } from "@loonfs/sdk";

const client = new LoonFSClient({
    environment: process.env.LOONFS_URL!,
    token: process.env.LOONFS_AUTH_TOKEN!,
});

const capabilities = await client.system.getCapabilities();
```

Upload and download helpers are exported from `@loonfs/sdk/transfers`.

Browser applications should use [`@loonfs/sdk-client`](./client) through a
server-side [`@loonfs/sdk-proxy`](./proxy). Never expose a LoonFS server token
to browser code.

## Retries

The SDK retries transient failures on operations that are safe to repeat.
Operations that LoonFS classifies as non-idempotent are never retried
automatically.

## Generated code

This SDK is generated from the LoonFS OpenAPI specification. Please report SDK
issues in the [main LoonFS repository](https://github.com/loonfs/loonfs).

## License

Apache-2.0.
