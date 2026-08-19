# LoonFS TypeScript SDK

TypeScript client for the LoonFS HTTP API.

## Status

Pre-release. Not yet published to npm. This repository is private until the first release.

## Install

```sh
npm install @loonfs/sdk
```

This command works after the first release.

## Usage

```ts
import { LoonFSClient } from "@loonfs/sdk";

const client = new LoonFSClient({
  environment: "https://your-loonfs-host.example",
  token: "your-api-token",
});

const capabilities = await client.capabilities();
```

## Generated code

This code is generated with Fern from the LoonFS OpenAPI spec (`docs/specs/openapi.json` in `github.com/loonfs/loonfs`). Regeneration runs from the `sdk/fern/` config in that repository (`scripts/generate-sdks.sh`). Do not edit generated files by hand.

## License

Apache-2.0.
