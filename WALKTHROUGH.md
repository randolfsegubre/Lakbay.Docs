# Platform Walkthrough — how the Lakbay apps talk to each other

This is the accessible, narrative companion to
[06_SYSTEM_ARCHITECTURE.md](docs/06_SYSTEM_ARCHITECTURE.md) (the precise
technical reference, with diagrams) and
[08_LOCAL_INFRASTRUCTURE.md](docs/08_LOCAL_INFRASTRUCTURE.md) (every Docker
container, what it's for). Read this one first if you're trying to
understand **how the eight repos actually work together as one system**
when they're all running at once - each repo's own `WALKTHROUGH.md` then
goes deeper into that repo specifically.

## The eight repos, in one line each

| Repo | Job |
|---|---|
| `Lakbay.Contracts` | The shared schema every other repo agrees to speak - no running service |
| `Lakbay.Cms` | Where an editor authors the catalog and marketing pages (Umbraco) |
| `Lakbay.AvailabilityApi` | The fast, search-optimized read copy of the catalog (GraphQL + MongoDB) |
| `Lakbay.Booking` | Owns real bookings and the atomic availability-decrement logic |
| `Lakbay.Web` | The public storefront (Next.js) - the only thing customers see |
| `Lakbay.AgentDesktop` | A WPF app a call-center agent uses to book on a customer's behalf |
| `Lakbay.AgentOps` | Backend-for-frontend for AgentDesktop - aggregates offers, proxies bookings, logs calls |
| `Lakbay.Docs` (this repo) | No application code - the platform's shared "why," ADRs, and ops reference |

## Scenario 1: an editor publishes a new holiday package

This is the platform's main data pipeline - the thing that makes
`Lakbay.Cms` the single source of truth without `Lakbay.Web` ever querying
it directly for catalog data.

```
1. Editor clicks Publish on a Product in Lakbay.Cms's backoffice
     -> fires a real Umbraco ContentPublishedNotification

2. Lakbay.Cms's CatalogPublishSyncHandler catches it
     -> resolves every Content Picker field to its published target
        (never a possibly-unpublished draft)
     -> builds a CatalogSyncEvent shaped exactly like Lakbay.Contracts says

3. Lakbay.Cms's ServiceBusCatalogSyncPublisher
     -> sends the event as JSON to the "lakbay-catalog-sync" Service Bus queue

4. Lakbay.AvailabilityApi.Sync (a separate Azure Function process)
     -> CatalogSyncFunction wakes up, consumes the message
     -> applies it to MongoDB via an upsert that only succeeds if the
        incoming data is NEWER than what's already stored (last-write-wins,
        ADR-0010) - a stale or duplicate message is silently, safely discarded

5. Lakbay.AvailabilityApi.Api (the separate, always-running GraphQL query
   service) now serves the updated data - it never talks to Service Bus
   itself, so a burst of sync traffic never slows down a shopper's search

6. Lakbay.Web queries Lakbay.AvailabilityApi.Api on the next page load
     -> the new/updated product appears on the storefront
```

**Why so many hops instead of `Lakbay.Cms` writing to MongoDB directly?**
`Lakbay.Cms` (the write/authoring side) and `Lakbay.AvailabilityApi` (the
read/search side) are two independently deployed, independently scaled
services (ADR-0007, ADR-0009) - neither one calling the other directly
means an outage or slowdown in one never cascades into the other. The
Service Bus queue is the only thing that couples them, and even that
coupling is one-directional and asynchronous.

**The one field this pipeline never touches:** `Product.AvailableCount`.
`Lakbay.Cms` sends a placeholder value for it, and
`CatalogSyncFunction.ApplyProductAsync` deliberately never applies it -
see Scenario 2 for why.

## Scenario 2: a customer (or an agent) books a holiday

`Lakbay.Booking` is the *only* writer of real, live availability -
solving a different problem than Scenario 1's catalog sync (ADR-0014
splits these into two independent concerns on purpose, since `Product` has
two writers now):

```
1. Lakbay.Web's checkout (or Lakbay.AgentDesktop, via AgentOps - see
   Scenario 3) sends a confirm-booking request to Lakbay.Booking

2. Lakbay.Booking.ConfirmBookingCommandHandler runs ONE atomic SQL
   statement: UPDATE AvailabilitySlots SET AvailableCount = AvailableCount - 1
   WHERE ... AND AvailableCount > 0
     -> if 0 rows changed, nothing was available - no partial state to
        roll back, ADR-0011's whole reason for existing (never a
        read-then-write race between two simultaneous bookings for the
        last slot)
     -> if it succeeds, a real BookingReservation row is inserted in the
        same database transaction

3. (Designed, not yet built - Phase 4/ADR-0008): Lakbay.Booking would
   publish an AvailabilityChanged event so Lakbay.AvailabilityApi.Sync
   can reflect the new count without Lakbay.Web needing to poll or reload
```

## Scenario 3: a call-center agent books on a customer's behalf

The Agent Channel (built to demonstrate WPF/WCF/ABP/Hangfire/Redis/SignalR
patterns from real job-posting requirements) reuses Scenario 1 and 2's
same backends through a different front door:

```
1. A simulated phone call arrives at Lakbay.AgentDesktop (WPF) via a WCF
   duplex service (Lakbay.AgentDesktop.TelephonyBridge) - the screen-pop

2. The agent browses live offers: AgentDesktop calls
   GET /api/agent-offer/{destinationCode} on Lakbay.AgentOps

3. Lakbay.AgentOps aggregates that view by querying Lakbay.AvailabilityApi's
   GraphQL API directly (accommodations + activities + products for that
   destination), caching the result in Redis for 10 minutes so a busy call
   center doesn't hammer AvailabilityApi with the same query repeatedly

4. The agent confirms a booking: AgentDesktop calls
   POST /api/bookings/confirm on Lakbay.AgentOps, which proxies the
   request to Lakbay.Booking (the BFF/Backend-for-Frontend pattern,
   ADR-0021) - Lakbay.Booking has no idea it's talking to an agent tool
   versus the public website, it's the exact same ConfirmBookingCommandHandler
   from Scenario 2

5. Lakbay.AgentOps logs the call outcome via a Hangfire background job to
   an Oracle database (ADR-0024 - a deliberately separate, narrow-scope
   store, framed as integrating with a pre-existing on-prem CRM system)
```

**Payment is genuinely different on this channel** (ADR-0026): an
agent-booked reservation never touches a payment gateway at confirm time -
it goes straight to `PaymentStatus.PendingInvoice`, since there's no
PayMongo sandbox account to test a real charge against yet. This was a
deliberate design choice to get a real, working end-to-end booking flow
proven live without waiting on that external blocker.

## Where to look when something's not working

- **A product doesn't show up / shows stale data on the storefront** →
  check `lakbay_servicebus` logs and `Lakbay.AvailabilityApi.Sync`'s
  console first (Scenario 1, step 4) - is the message actually being
  consumed? See `08_LOCAL_INFRASTRUCTURE.md`'s verification commands.
- **A booking succeeds but availability doesn't visibly change on the
  storefront** → expected today - the AvailabilityChanged propagation in
  Scenario 2's step 3 is designed (ADR-0008) but not yet built (Phase 4).
- **AgentDesktop's offer screen is empty/errors** → check whether
  `Lakbay.AvailabilityApi.Api` is actually running; `Lakbay.AgentOps`
  fails cleanly with a real connection exception when it isn't, not a
  confusing generic error.
- **A local run needs to know which of the four running processes to
  start, in what order** → `07_MANUAL_SETUP_GUIDE.md` has the full,
  proven sequence.

## What's designed but not real yet

Two things worth knowing before assuming they work: **real-time
availability push to the storefront** (ADR-0008 - SignalR, so a shopper
sees a slot disappear without refreshing) and **live PayMongo payment
processing** (ADR-0026 - currently only the Agent channel's
deferred-invoice path is real end to end). Check `04_TASKS.md` for the
current, up-to-date status of both before building on top of either.
