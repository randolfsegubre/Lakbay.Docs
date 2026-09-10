# ADR-0014: Split Product's sync ordering guard by writer, before a second writer exists

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.AvailabilityApi`

## Context

Found while implementing ADR-0013, not asked for directly. ADR-0010's
last-write-wins guard compares one field, `sourceUpdatedUtc`, against the
stored document before applying an update. That's correct as designed for
its original case — two `AvailabilityChanged` events for the *same* field
group racing each other. It stops being correct the moment a second,
independent writer exists for the *same document* but a *different* field
group, which is exactly `Product`'s actual shape:
`Lakbay.Cms` owns every catalog field (name, slug, itinerary, price bands,
etc.); `Lakbay.Booking` owns `availableCount` alone (ADR-0011).

Concretely: an editor fixes a typo at 10:00:02 in `Lakbay.Cms`, but the
event is still in flight when `Lakbay.Booking` confirms a booking at
10:00:05 and its (future, Phase 4) sync event lands first, setting
`sourceUpdatedUtc = 10:00:05`. The editor's event then arrives with
`sourceUpdatedUtc = 10:00:02` — older than what's stored — and
ADR-0010's guard would silently discard a perfectly valid, unrelated
catalog edit. The two writers were never actually racing on the same
data; a shared single timestamp made it look like they were.

`Lakbay.Booking` doesn't exist yet (Phase 4), so this can't happen today.
But `02_BUILD_PLAN.md`'s own Phase 1 section already warned against
exactly this shape of mistake — "retrofitting an ordering field after
Phase 3's sync job exists is exactly the kind of rework worth avoiding" —
so this is fixed now, while `Lakbay.Cms`'s sync handler is being written,
rather than left for Phase 4 to discover the hard way.

## Decision

`Product`'s Mongo document gets a second, internal-only ordering field,
`availabilityUpdatedUtc`, alongside the existing `sourceUpdatedUtc` (which
becomes, in practice, "catalog fields' own last-updated"). It is **not**
added to `Lakbay.Contracts.Product` and **not** exposed in the GraphQL
schema — nothing outside `Lakbay.AvailabilityApi`'s storage layer needs it,
and the shared contract stays exactly as `Lakbay.Cms`, `Lakbay.Web`, and
`Lakbay.Booking` already know it. It's written via `MongoDB.Driver`'s
untyped `BsonDocument`/`UpdateDefinition` path, not as a property on the
shared record.

- `Lakbay.Cms`'s sync handler `$set`s only catalog fields (`name`, `slug`,
  `productLine`, `destination`, `summary`, `itineraryDays`, `boardBasis`,
  `priceBands`, `heroImageUrl`, `sourceUpdatedUtc`) and gates on
  `sourceUpdatedUtc` alone. It never reads or writes `availableCount` or
  `availabilityUpdatedUtc`.
- `Lakbay.Booking`'s (Phase 4) sync handler will `$set` only
  `availableCount` and `availabilityUpdatedUtc`, gated on
  `availabilityUpdatedUtc` alone. It will never touch the catalog fields
  above.
- First sync of a brand-new product (upsert, not yet in MongoDB) sets
  `availableCount = 0` via `$setOnInsert` — safely "not yet bookable"
  rather than an undefined/missing required field — until
  `Lakbay.Booking` writes a real value in Phase 4.
- `Product` and `Destination`'s `BsonClassMap` entries get
  `SetIgnoreExtraElements(true)` (matching the pattern `ProductLine`
  already uses, `MongoClassMaps.cs`) — defensive, and specifically what
  makes storing `availabilityUpdatedUtc` outside the POCO safe: the query
  API's read path can ignore a Mongo field it doesn't map, the same way it
  already does for `ProductLine`'s auto-assigned `_id`.

`Destination` and `ProductLine` are untouched — both are single-writer
(`Lakbay.Cms` only), so ADR-0010's original single-timestamp guard is
already correct for them.

## Alternatives considered

- **Leave it for Phase 4 to fix** — rejected per the Build Plan's own
  stated reasoning above: the cost of getting this right is a few extra
  lines today; the cost of discovering it via a real bug report ("my price
  update didn't stick") after `Lakbay.Booking` ships is much higher, and a
  fix then would need a data migration for every already-synced product.
- **Add `availabilityUpdatedUtc` to the shared `Lakbay.Contracts.Product`
  record** — rejected: it's a storage/ordering concern internal to how
  `Lakbay.AvailabilityApi` reconciles two writers, not something
  `Lakbay.Cms`, `Lakbay.Web`, or the GraphQL contract need to know about.
  Keeping it out of the shared type keeps the public contract exactly as
  small as ADR-0007/0009 intended.
- **One shared `updatedUtc` with per-writer field-level version numbers
  instead of two timestamps** — rejected as more machinery than the actual
  problem needs; two independent timestamps, one per non-overlapping field
  group, is the simplest thing that's still correct.

## Consequences

- `Lakbay.AvailabilityApi.Sync`'s Mongo write path for `Product` cannot go
  through a simple typed `ReplaceOne(product)` — it must build an explicit
  field-scoped `UpdateDefinition<Product>` (already implied by ADR-0010's
  "single atomic `findOneAndUpdate`," now applied per writer instead of
  per document).
- A future Phase 4 session implementing `Lakbay.Booking`'s sync handler
  should read this ADR before writing any Mongo update code for `Product`
  — the temptation to "just update the whole document" is exactly the bug
  this ADR exists to prevent.
- `CatalogSeeder` (Phase 1's hand-seeded data) already sets
  `availableCount` directly at insert time — this ADR doesn't change that
  path, only the sync-from-events path introduced alongside it in Phase 3.
