# LoonFS browser client

Browser client for the LoonFS HTTP API, spoken through a LoonFS proxy in your
backend. Paths are mount-scoped: the client calls `/v0/mounts/{mount}/...` on
your proxy origin, and the proxy maps mounts to namespaces and adds
credentials.

## Status

This package is pre-release. It is not yet published to npm.

## Install

After the first release, install it with:

```sh
npm install @loonfs/sdk-client
```

## Generated code

This code is generated with Fern from the LoonFS proxy document
(`docs/specs/openapi-proxy.json` in `github.com/loonfs/loonfs`). Regeneration
runs from the `sdk/fern-client/` workspace in that repository
(`scripts/generate-sdks.sh typescript-client`). Do not edit generated files by
hand. The package has no runtime dependencies; `@types/node` is a types-only
development dependency for a runtime-guarded stack-trace call.

## License

Apache-2.0.
