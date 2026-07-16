# Current Repository Mapping

The original design documents in this repository describe the ecosystem using repository
names from the `skintwin-ai` organization plan. The ecosystem is now implemented across
the `jax-a11y` and `rzonedevops` accounts. This document maps the design-era names onto
the repositories that actually exist today, and places every active repository in the
architecture. The machine-readable version of this mapping is
[`registry/ecosystem.json`](../registry/ecosystem.json); each repository also carries its
own copy at `.skintwin/manifest.json`.

## Design-name → actual repository

| Design document name | Actual repository | Notes |
|---|---|---|
| `skintwin` (Alchemist Engine spec) | [`jax-a11y/skintwin-workbench`](https://github.com/jax-a11y/skintwin-workbench) | Contract-and-docs hub; carries the Alchemist Engine spec and the MRP/workbench OpenAPI contracts |
| `skintwin-asi` (Shopify App Service) | [`jax-a11y/skintwin-asi`](https://github.com/jax-a11y/skintwin-asi) | Shopify embedded admin app ("Cognitive Alchemist Workbench") |
| `multiskin` | [`rzonedevops/neuro-symbolic-core`](https://github.com/rzonedevops/neuro-symbolic-core) | Reference cognitive-service implementation (nettica + Flask REST API); [`rzonedevops/skin7nn`](https://github.com/rzonedevops/skin7nn) is a heptavertonic variant of the same foundation |
| `skintwin-customer-portal` (Customer & Org Service) | [`jax-a11y/skintwin-customer-portal`](https://github.com/jax-a11y/skintwin-customer-portal) | Name unchanged |
| `regima-training-lms` (LMS Service) | [`rzonedevops/regima-training-lms`](https://github.com/rzonedevops/regima-training-lms) | Name unchanged; [`rzonedevops/regima-train-nnllms`](https://github.com/rzonedevops/regima-train-nnllms) is the nn→llm→lms training-pipeline variant |
| `skintwin-integrations` (E-commerce & Booking Service) | [`jax-a11y/skintwin-integration-hub`](https://github.com/jax-a11y/skintwin-integration-hub) | Contains the `AmazingSalonApp9ragbot3` tree documented in the integration architecture |
| `business-directory-template` / `bus-listing` | [`jax-a11y/skintwin-marketplaces-buyer-app`](https://github.com/jax-a11y/skintwin-marketplaces-buyer-app) (surface), [`jax-a11y/skin-zone`](https://github.com/jax-a11y/skin-zone) (design) | The directory/discovery role is served by the marketplace buyer app; skin-zone documents the marketplace platform |
| `org-skin` (org management) | [`jax-a11y/bolt.ceo`](https://github.com/jax-a11y/bolt.ceo) | Nearest fit: JAX CEO orchestration/dev tooling; no direct successor exists |

## Repositories with no design-era counterpart

| Repository | Layer | Role |
|---|---|---|
| [`jax-a11y/skintwin-salon`](https://github.com/jax-a11y/skintwin-salon) | commerce-surface | Gatsby booking/checkout storefront (Paystack Terminal) |
| [`rzonedevops/regima-salon-suite`](https://github.com/rzonedevops/regima-salon-suite) | commerce-runtime | "Atelier OS" multi-store Shopify operator dashboard |
| [`jax-a11y/skintwin-marketplace-admin-app`](https://github.com/jax-a11y/skintwin-marketplace-admin-app) | commerce-runtime | Marketplace channel admin (GraphQL BFF) |
| [`jax-a11y/skintwin-ordering-agent`](https://github.com/jax-a11y/skintwin-ordering-agent) | cognitive | Conversational ordering agent (Mastra + Paystack) |
| [`rzonedevops/skinsource-pro`](https://github.com/rzonedevops/skinsource-pro) | erp-federation | Procurement platform — concrete procurement domain of the Federated ERP layer |
| [`rzonedevops/pcsdbx1`](https://github.com/rzonedevops/pcsdbx1) | erp-federation | Supplier-intelligence database feeding skinsource-pro |
| [`rzonedevops/regima-cognitive-dashboard`](https://github.com/rzonedevops/regima-cognitive-dashboard) | observability | KSM cognitive-pipeline dashboard |
| [`rzonedevops/af.github.io`](https://github.com/rzonedevops/af.github.io) | marketing | RégimA brand-site archive (regima.site + regimazone.org) |
| [`jax-a11y/regimazone-org-mirror`](https://github.com/jax-a11y/regimazone-org-mirror) | marketing | RegimA Zone site mirror |

## Practical corrections to the design documents

- **Reusable CI workflows** live in `jax-a11y/skintwin-ecosystem-design` (public), so caller
  syntax is `uses: jax-a11y/skintwin-ecosystem-design/.github/workflows/<file>.yml@main` —
  not `skintwin-ai/...` as older examples showed. `ci/README.md` has been updated.
- **The integration hub** (`skintwin-integration-hub`) runs Flask on port 5000
  (`config.example.json`); its health endpoint is `GET /api/integrations/health`.
- **Service discovery** across repos uses the `SKINTWIN_*_URL` environment-variable
  conventions listed in `registry/ecosystem.json` (`serviceEnv`).
