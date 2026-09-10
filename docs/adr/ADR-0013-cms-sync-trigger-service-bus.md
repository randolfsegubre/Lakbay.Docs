# ADR-0013: Lakbay.Cms → Lakbay.AvailabilityApi sync trigger is Service Bus, reusing the existing Sync function

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.Cms`, `Lakbay.AvailabilityApi` (`Lakbay.AvailabilityApi.Sync`)

## Context

`01_CLAUDE.md` §6 and `02_BUILD_PLAN.md`'s Phase 1 section both flagged this
as a genuine open item, not to be silently resolved: how does `Lakbay.Cms`
tell `Lakbay.AvailabilityApi` that a product/destination changed? Three
options were on the table — an Umbraco content-cache-refresher event
consumed some other way, a scheduled polling job, or a message on the
Service Bus infrastructure ADR-0008/0009 already designed for
`Lakbay.Booking`'s real-time availability push.

## Decision

`Lakbay.Cms` publishes to Azure Service Bus on content publish (an
`INotificationHandler<ContentPublishedNotification>`, Umbraco's own
extension point — not a cache-refresher hack). `Lakbay.AvailabilityApi.Sync`
— the same Azure Function ADR-0009 already built for
`Lakbay.Booking`'s events — gets a second trigger function subscribed to a
dedicated queue, `lakbay-catalog-sync`, separate from the (not yet built)
`lakbay-availability-changed` queue Phase 4 will add for `Lakbay.Booking`.
One event-consumption mechanism for the whole platform, not two.

Locally, both repos point at the official Azure Service Bus Emulator
(Docker: `mcr.microsoft.com/azure-messaging/servicebus-emulator` + its
required `azure-sql-edge` companion) via the well-known
`UseDevelopmentEmulator=true` connection string — no real Azure Service Bus
namespace needed for local dev, matching this platform's standing
offline-first rule (same reasoning as ADR-0005 for SQL Server).

## Alternatives considered

- **Scheduled polling (Hangfire job) reading Cms's Content Delivery API** —
  rejected: adds latency (the Blueprint's whole point with real-time
  availability was pushing away from polling, ADR-0008), and introduces a
  second sync paradigm alongside the event-driven Booking path for no
  benefit — nothing about content publishing is naturally polling-shaped.
- **Direct Umbraco event → synchronous HTTP call into
  `Lakbay.AvailabilityApi`** — rejected for the same reason ADR-0009 chose
  a separate deployable over an in-process `BackgroundService`: it would
  couple `Lakbay.Cms`'s publish path to `Lakbay.AvailabilityApi`'s uptime.
  An editor publishing a typo fix should never fail because the search API
  happens to be redeploying.
- **A separate Azure Function / consumer just for Cms events** — rejected:
  it already shares the exact job (validate ordering, write to MongoDB,
  same `Lakbay.Contracts` types) that `Lakbay.AvailabilityApi.Sync` exists
  for. A second trigger function in the same project, not a second
  project.

## Consequences

- `Lakbay.Cms` takes a new dependency on `Azure.Messaging.ServiceBus` and a
  small `ICatalogSyncPublisher` service, composed at startup per this
  repo's Dependency-Inversion convention (`03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`).
- `Lakbay.AvailabilityApi.Sync` takes on its first real trigger function
  (it was an empty scaffold before this). `Lakbay.Booking`'s
  `lakbay-availability-changed` consumer (Phase 4) is a sibling function
  added later, not a rewrite of this one.
- Local dev for either repo now requires the Service Bus emulator
  container running — documented in both repos' `Docs/DEVELOPER_HANDBOOK.md`
  and `07_MANUAL_SETUP_GUIDE.md`.
- See ADR-0014 for a related but distinct problem this decision surfaced:
  `Product` has two independent writers (`Lakbay.Cms` for catalog fields,
  `Lakbay.Booking` for `availableCount`, Phase 4), and ADR-0010's
  single-timestamp last-write-wins guard needs a small correction before a
  second writer exists, to avoid the two colliding.
