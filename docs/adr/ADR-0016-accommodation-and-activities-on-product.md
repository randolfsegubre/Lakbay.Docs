# ADR-0016: Product carries accommodation + included/optional activities

**Status:** Accepted — 2026-09-08

## Context

The user's own framing, verbatim: "I notice that in the collections, it is
just showing the place... If you remember HotelPlan applications, they are
offering accommodations, not just activities but activities along with
it." They pointed at three real Hotelplan-family sites — Inghams,
Inntravel, Santa's Lapland — as reference.

Browsing those three sites (2026-09-08) surfaced a consistent structural
pattern, not just a styling one:

- **The accommodation is a first-class part of the product**, not
  scenery. Inghams shows "Properties available: 13 · From £1,049pp · View
  accommodation" on every destination card — a customer picks a place,
  then a specific hotel/lodge/cabin.
- **Activities split into two buckets**: bundled into the package price,
  and optional add-ons booked on top. Santa's Lapland states this
  explicitly — the "Santa's Magic" package includes a husky ride,
  reindeer sleigh, tobogganing, and elf shows; a separate "Fancy more
  snow fun?" section lists optional extras (Northern Lights excursion,
  longer husky ride, snowmobile safari) as their own cards.

Lakbay's `Product` had neither: no accommodation field at all, and
activities existed only as unstructured prose inside `summary`. A
collection page genuinely was "just showing the place" — the destination
and a price, nothing that names what a guest actually gets.

## Decision

Add four fields to `Product` (schema/lakbay.graphql, Lakbay.Contracts,
Lakbay.Cms, Lakbay.AvailabilityApi.Sync, Lakbay.Web):

```graphql
accommodationName: String
accommodationDescription: String
includedActivities: [String!]!
optionalActivities: [String!]!
```

**Not a separate Accommodation entity.** Inghams' "13 properties, pick
your stay" pattern implies a real one-to-many relationship (one
destination, many bookable accommodations) — a much bigger lift: a new
Cms content type/tree, a picker UI, a new GraphQL type, and a real
per-property price/availability model. Lakbay's catalog is
one-accommodation-per-product today, matching Santa's Lapland's simpler
"package includes this specific stay" shape more closely than Inghams'.
Fields on the existing `Product` type were chosen explicitly over that
bigger model — revisit only if the product genuinely needs multiple
lodging choices per destination, not preemptively.

**No per-activity pricing.** `optionalActivities` is a plain string list,
not `{name, pricePhp}`. Hotelplan sites don't show a price on the
optional-extras teaser cards either — pricing detail lives past that
point in their own booking flow, which Lakbay doesn't have yet (Phase 4).
Designing a pricing shape for optional activities now, before
`Lakbay.Booking` exists to actually charge for them, would be speculative.

**Newline-separated Textarea in Cms, not JSON.** `includedActivities`/
`optionalActivities` are flat string lists with no nested shape to
preserve, unlike `priceBands` (which does need JSON — see
`CatalogContentTypeSeeder`'s own v0-simplifications note). One activity
per line is simpler for a content editor to type and needs no escaping.

**Fields, not `required`, on the C# record.** `AccommodationName`/
`AccommodationDescription` are nullable; `IncludedActivities`/
`OptionalActivities` default to `[]`. Marking these `required` would have
broken every existing construction site (tests, `CatalogSeeder`) the
moment this field was added — the schema comment states this plainly
rather than silently picking `required` and fixing the fallout site by
site.

## A real gap found during rollout, not designed in advance

Adding a property to an *existing* Umbraco content type isn't something
`CatalogContentTypeSeeder`'s original `if (!typesAlreadyExist)` guard
handles — that guard only creates the type once, on a truly empty
schema. On this machine, `ProductLine`/`Destination`/`Product` types
already existed from earlier sessions, so the guard skipped type creation
entirely, and the first reseed attempt crashed: `SetValue("accommodationName", ...)`
threw `No PropertyType exists with the supplied alias`.

Fixed with the same shape ADR-0015 already established for the Content
tree's abandoned Block List cleanup: a one-time, self-limiting migration
gated on a specific tell (`accommodationName` missing from the existing
`Product` type) that deletes and recreates all three Products-tree
content types plus their content. Once recreated, the tell is
permanently false, so this never re-triggers on a normal boot — it isn't
a general "detect any schema drift" mechanism.

**A second real gap, found only by actually reading the synced data**:
after the fix, `accommodationName` still came back `null` over GraphQL.
The Azure Functions Sync host (`func start`) had been running since
before this session's code changes and was never restarted — it was
still executing the old `CatalogSyncFunction.cs` build, which had no
`.Set(p => p.AccommodationName, ...)` in its update. Every long-running
local process this platform depends on (Cms, the query API, *and* the
Sync function host) needs restarting after a schema change that touches
it — not just the two that happen to be started most often.

**A third, mechanical consequence of the above**: each crash-then-fix
cycle recreated Products-tree content with fresh Umbraco UDIs, and
`CatalogSyncFunction` upserts Mongo documents keyed by that UDI — so
every recreation left the previous generation's Mongo documents orphaned
(same issue ADR — no, not a formal ADR, just documented in the 2026-09-08
(8) devlog entry — already found once for the rename/destinations work).
Cleaned up the same way each time: a targeted `deleteMany` on the
pre-cutoff `SourceUpdatedUtc` generation, confirmed via a
`$dateTrunc`-grouped count first, never a collection drop. This is a
known, accepted rough edge of `LAKBAY_FORCE_RESEED_CATALOG` specifically
— a real editor's backoffice edit never regenerates a node's UDI, so
this never happens outside this dev-only escape hatch.

## Also decided in this session, not requiring code

**Lakbay stays Philippines-only, brand unchanged.** The user raised
whether "Lakbay" — a Filipino word — forecloses a future Southeast Asia
expansion, given the original plan considered other Asian markets.
Decision: keep the name and the PH-only scope for now. "Lakbay" means
"journey," not literally "Philippines" — the word itself doesn't block a
future non-PH destination the way a geography-specific name would.
Revisit scope only when there's a real destination to add, not
speculatively; this ADR's schema work (accommodation/activities) is
deliberately geography-agnostic already, so nothing here would need
rework if that decision changes later.

## Consequences

- Every collection card and holiday detail page can now show a named
  stay and a real "what's included" list — the actual gap the user
  pointed at is closed structurally, not just with better copy.
- All 14 existing/new products were authored with real accommodation
  names (plausible, generic — not real hotel brand names, to avoid
  implying an actual business relationship) and activity lists specific
  to that destination, not filler text.
- The next content addition to this schema (a fifth field, say) should
  check the *existing* Product type's property list the same way this
  migration did, on any machine where the type might predate the change
  — not assume a fresh-install code path is the only one that matters.
