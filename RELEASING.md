# Releasing

All three npm packages use the same version. A v0.1.x SDK targets LoonFS API
v0.3.x.

Before the first release, create the `@loonfs` npm organization and confirm the
publisher account has 2FA and write access. Build and inspect all three packages:

```sh
npm ci
npm run build
npm run build --workspaces
npm pack --dry-run
npm pack --dry-run --workspace client
npm pack --dry-run --workspace proxy
```

The first public release must bootstrap each package from an authorized local
npm session:

```sh
npm publish --access public
npm publish --workspace client --access public
npm publish --workspace proxy --access public
```

After that release, configure each package to trust
`.github/workflows/release.yml` in this repository. Create and push the matching
`vX.Y.Z` tag, then run the **Publish packages** workflow with that tag. The
`npm` GitHub environment should require a maintainer's approval.
