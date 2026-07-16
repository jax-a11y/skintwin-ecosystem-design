# Ecosystem Contracts

Machine-readable contracts shared across the SkinTwin ecosystem. These are the artifacts
that [`ci-contracts.yml`](../.github/workflows/ci-contracts.yml) validates (JSON Schemas
under a `contracts/` directory, with breaking-change diffs against `*-previous` versions).

## Event contracts (`events/`)

All inter-service events use a common envelope, defined in
[`events/envelope.schema.json`](./events/envelope.schema.json):

```json
{
  "id": "evt_9f2c1b",            // idempotency key — consumers process exactly once
  "type": "user.created",        // dot-separated topic
  "payload": { "userId": "…" },  // camelCase, per-topic schema
  "partitionKey": "user:123",    // ordering preserved per key
  "occurredAt": "2026-07-12T10:00:00Z"
}
```

Naming convention: `entity.pastTenseVerb` (`user.created`, `order.placed`);
service-namespaced for service-emitted outcomes (`lms.user.provisioned`,
`erp.product.synced`). Failed deliveries are retried with `retryCount`, then routed to the
dead-letter queue at `/api/events/dlq`.

Each core topic has a payload schema in `events/<topic>.schema.json`. The authoritative
topic catalog — including which repositories publish and subscribe to each topic — is the
`events.topics` map in [`registry/ecosystem.json`](../registry/ecosystem.json).

| Topic | Publishers | Subscribers |
|---|---|---|
| `user.created` | customer-portal | regima-training-lms, integration-hub |
| `order.placed` | skintwin-salon, ordering-agent | customer-portal, integration-hub |
| `order.created` | integration-hub | customer-portal |
| `order.paid` | integration-hub | customer-portal, skinsource-pro |
| `product.updated` | integration-hub | skintwin-asi, marketplaces-buyer-app |
| `inventory.updated` | skinsource-pro, integration-hub | skintwin-asi, regima-salon-suite |
| `appointment.created` | integration-hub, skintwin-salon | customer-portal |
| `appointment.updated` | integration-hub | customer-portal |
| `course.completed` | regima-training-lms | customer-portal |
| `certification.awarded` | regima-training-lms | customer-portal |

## API contracts

REST/GraphQL contracts are declared in `registry/ecosystem.json` (`contracts[]`) with
provider, consumers, and gateway path prefix. OpenAPI documents live with their provider
repositories (e.g. the MRP/workbench specs in `jax-a11y/skintwin-workbench`); this
directory holds only the cross-cutting schemas.

## Evolving a contract

1. Copy the current schema to `<name>-previous.schema.json` (used by the breaking-change diff).
2. Edit the schema; update `registry/ecosystem.json` if publishers/subscribers change.
3. Run `ci-contracts.yml` from the consuming/providing repo, or open a PR here — the
   contract validation workflow flags breaking changes.
