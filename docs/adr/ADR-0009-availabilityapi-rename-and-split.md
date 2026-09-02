# ADR-0009: Rename to Lakbay.AvailabilityApi; split query-serving from event-consumption

- **Status:** Accepted — renames `Lakbay.SearchApi` (itself a rename of
  `Lakbay.MockApi`, ADR-0007) and refines its internal shape
- **Date:** 2026-09-06
- **Repo(s) affected:** `Lakbay.AvailabilityApi` (renamed from
  `Lakbay.SearchApi`), `Lakbay.Booking`

## Context

Two separate points, raised together:

1. "`Lakbay.SearchApi` is too generic" — fair. The repo's job was never
   just search; ADR-0008 already gave it real-time availability
   propagation, arguably its more defining responsibility now.
2. Concern that consuming Service Bus events, writing to MongoDB, and
   pushing SignalR notifications *inside* the same process that serves
   GraphQL queries would compete with query-serving for CPU/threads —
   "search function won't have so much things to do, will not affect
   performance." Direct analogy drawn to how this memory system's own
   sync hooks work: fire on an event, run isolated, don't live inside the
   main loop.

## Decision

Rename the repo to **`Lakbay.AvailabilityApi`**. Split its runtime into
two deployables that share the same MongoDB database and the same
`Lakbay.Contracts` types, but scale and deploy independently:

1. **`Lakbay.AvailabilityApi`** (the GraphQL/HotChocolate service) — pure
   read path. Answers queries. Never touches Service Bus directly.
2. **`Lakbay.AvailabilityApi.Sync`** (an Azure Function, `[ServiceBusTrigger]`)
   — the event consumer. Subscribes to `Lakbay.Cms` publish-sync events and
   `Lakbay.Booking`'s `AvailabilityChanged` events, applies the update to
   MongoDB (with last-write-wins ordering, see ADR-0010), then pushes the
   Azure SignalR notification. This is the direct .NET/Azure equivalent of
   a webhook — event-triggered, isolated compute, scales by message
   volume rather than query load, and can be redeployed or scaled without
   touching the query API at all.

Both live in the same repo (a solution with two projects) since they
share domain types and the MongoDB access layer — this is a deployment
split, not a repo split.

## Alternatives considered

- **Keep the Service Bus consumer as a `BackgroundService` inside the same
  ASP.NET Core process as the GraphQL API** — rejected: a hosted service
  shares the same process, thread pool, and deploy/scale unit as the query
  API. A burst of availability events (e.g. a flash sale selling out
  several listings at once) would compete for resources with concurrent
  shoppers querying the catalog at the same time — exactly the contention
  this ADR exists to avoid.
- **A fully separate repo for the sync worker** — rejected: it shares
  enough (MongoDB schema, `Lakbay.Contracts` types, the "what does a valid
  availability update look like" logic) that a separate deployable within
  the same repo is the right level of separation — matches the
  Repository/Dependency-Inversion patterns already used elsewhere rather
  than introducing a seventh repo for marginal gain.

## Consequences

- `02_BUILD_PLAN.md` Phase 1 (query API + resolvers) and Phase 4 (sync
  worker, Service Bus, SignalR) now map to two different projects within
  one repo, not one undifferentiated repo.
- CI needs to build and test both projects; the schema-diff check (ADR-0007)
  applies to the query-serving project only, since that's what exposes
  GraphQL.
- Every prior reference to `Lakbay.SearchApi` across `Lakbay.Docs` and the
  other repos is updated to `Lakbay.AvailabilityApi` — done alongside this
  ADR. Earlier ADRs (0004, 0007, 0008) keep their original filenames and
  historical wording; they are point-in-time records, not rewritten.
