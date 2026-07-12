# Ecosystem Position: skintwin-ecosystem-design

> Part of the [SkinTwin-AI ecosystem](https://github.com/jax-a11y/skintwin-ecosystem-design).
> Manifest: [`.skintwin/manifest.json`](./.skintwin/manifest.json) ·
> Registry: [`registry/ecosystem.json`](./registry/ecosystem.json)

**Layer:** governance · **Role:** design-docs

This repository is the hub of the SkinTwin-AI ecosystem. It does not run in production;
every other repository points back here for:

| It provides | Where |
|---|---|
| Ecosystem registry (repos → layers, contracts, events) | [`registry/ecosystem.json`](./registry/ecosystem.json) |
| Manifest schema for per-repo `.skintwin/manifest.json` | [`schemas/ecosystem-manifest.schema.json`](./schemas/ecosystem-manifest.schema.json) |
| Event envelope + topic payload schemas | [`contracts/events/`](./contracts/events/) |
| Reusable CI workflows (`workflow_call`) | [`.github/workflows/`](./.github/workflows/) — caller syntax in [`ci/README.md`](./ci/README.md) |
| Architecture design docs | [`design/`](./design/) |
| Old-name → actual-repo mapping | [`analysis/repository_mapping_current.md`](./analysis/repository_mapping_current.md) |
| E2E test specifications | [`e2e/`](./e2e/) |

## How a repository joins the ecosystem

1. Add `.skintwin/manifest.json` conforming to the manifest schema, declaring layer, role,
   provided/consumed contract ids, and published/subscribed event topics.
2. Add an `ECOSYSTEM.md` describing the repo's position (like this file).
3. Register the repo in `registry/ecosystem.json` here.
4. Wire CI to the reusable workflows:
   `uses: jax-a11y/skintwin-ecosystem-design/.github/workflows/ci-node.yml@main`
   (or `ci-python.yml`; see [`ci/README.md`](./ci/README.md)).
5. Use the `SKINTWIN_*_URL` environment variables from the registry's `serviceEnv` map to
   locate other services — never hardcode service URLs.
