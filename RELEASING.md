# Releasing

`@loonfs/sdk` exposes the client, proxy, and server SDKs as subpath exports. A
v0.1.x SDK targets LoonFS API v0.3.x.

Build and inspect the package before every release:

```sh
npm ci
npm run build
npm pack --dry-run
```

For trusted publishing, configure `@loonfs/sdk` to trust
`.github/workflows/release.yml` in this repository. Create and push the
matching `vX.Y.Z` tag, then run the **Publish package** workflow with that tag.
The `npm` GitHub environment should require a maintainer's approval.
