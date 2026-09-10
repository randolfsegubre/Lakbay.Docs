# ADR-0019: Room Types, destination-level perks, and a cross-destination Stays search

**Status:** Accepted — 2026-09-08

## Context

The user described a concrete scenario: search for "a budget-friendly
hotel in Boracay for a family holiday," find its room options, and add
activities/perks around that stay — closer to how Inghams actually sells
holidays than how Lakbay currently worked. Asked to verify this directly
rather than take it on faith.

Opened a real Inghams hotel page (Hotel Post, St Anton) and confirmed a
structure Lakbay didn't have yet:

- A real **Room Types** section — "St Anton room" (20-24m², twin bed,
  sleeps 2), "Galzig room" (26-33m², seating area, sleeps 3) — each its
  own size, bed configuration, and occupancy, not a single flat
  description of "the accommodation."
- **"Included in your ski holiday to St Anton"** — a perks list (local
  destination-expert support, transfers) shown on the hotel page but
  genuinely resort-wide, identical regardless of which of that resort's
  hotels is picked.
- **Optional promotional add-ons** (free helmet hire with pre-booked
  skis) — also resort-wide, not hotel-specific.
- Official star rating and filterable tags ("Family Holidays," "Budget")
  shown per hotel, used to filter search results.

Two scope questions were confirmed with the user before building:
included/optional perks move to **Destination** level (shared by any
stay there), while `Product` keeps its own curated-itinerary activities
separately, since those are genuinely trip-specific, not resort-wide; and
this includes a real cross-destination **Stays** search page, not just a
per-destination upgrade — matching the actual "search for a hotel"
scenario, even though most destinations only have one accommodation
today (the structure is right and scales the moment a destination gets a
second option).

`Product` is **not removed**. Per the user's own framing ("should be
offered as packages, or additional option"), curated multi-day holidays
stay exactly as they are — repositioned in the UI as a secondary, optional
path ("or book this as a ready-made package") alongside the new primary
hotel-browsing path, not replaced by it.

## Decision

**1. `RoomType` is a new, top-level synced entity** — `Lakbay.Contracts`,
`Lakbay.Cms` (a flat `roomType` content type with an `accommodation`
Content Picker, matching the Products tree's established flat + picker
convention from ADR-0017, not nested under its Accommodation),
`Lakbay.AvailabilityApi` (own Mongo collection, own
`roomTypes(accommodationId)` query). It carries a plain
`accommodationId: String!` reference rather than an embedded
`Accommodation` object — the same "queried separately by parent id" shape
`ProductFilter.destinationId` already uses for `Product`/`Destination`,
not a new pattern, and it sidesteps a reverse-lookup problem an embedded
`Accommodation.roomTypes` list would otherwise need in the Cms publish
handler. 28 real room types seeded (2 per Accommodation — a
cheaper/simpler option and a pricier/larger one).

**2. `Accommodation` gains `destination`, `tags`, `officialRating`.**
Previously an Accommodation was only reachable indirectly, through a
Product that picked both a Destination and an Accommodation as siblings.
Making it independently browsable (the Stays page) meant it needed to
know its own Destination directly — a real, necessary structural
addition, not a convenience field. `tags` are short filterable labels
("Budget-Friendly," "Family-Friendly," "Beachfront") distinct from the
existing prose `highlights`. `officialRating` is nullable — not every
budget inn in this catalog has a formal star rating.

**3. `Destination` gains `includedPerks`/`optionalAddOns`** — resort-wide
items shared by any stay there, regardless of which Accommodation, mirroring
Inghams' "Included in your ski holiday to X" pattern. Kept separate from
`Product.includedActivities`/`optionalActivities`, which stay
itinerary-specific and curated per package — a confirmed scope decision,
not an oversight; the two lists serve genuinely different purposes and
can legitimately overlap in content without being the same field.

**4. A new `/stays` route** — a cross-destination search/browse page
(destination + tag filters, both client-side over the full ~14-item
accommodation list, the same "fetch small dataset, filter in the UI"
pattern the existing collection pages already use) — and
`/stays/[accommodationId]`, a full detail page: photo, tags, rating, all
its Room Types (via `RoomTypeList`, a new shared component also reused by
the Destination page), the owning Destination's perks/add-ons, and a
"book as a ready-made package" section listing any `Product`s that
reference this Accommodation.

**5. Existing Destination page updated**, not replaced: its "Where you'll
stay" section now also renders that Accommodation's Room Types and links
to its full `/stays/[id]` detail page; the Destination's own perks/add-ons
render once, destination-wide, not duplicated per accommodation; "Holidays
here" is retitled "Or book as a ready-made package," matching the
repositioning decided above.

**6. A real, pre-existing schema gap found and fixed along the way**:
HotChocolate infers a plain C# `string?` parameter as GraphQL `String`,
not `ID` — a bare member literally named `Id` gets `ID` by convention, but
`destinationId`/`accommodationId` parameters do not. The authored
`schema/lakbay.graphql` had declared `accommodations(destinationId: ID
...)`/`roomTypes(accommodationId: ID!)`, which the live server never
actually served that way; a client query declaring `$destinationId: ID`
against the real `String`-typed argument fails GraphQL's variable-usage
compatibility check outright (caught live, in the browser, not by `tsc`
or `dotnet build`). Fixed by matching the SDL and the hand-written
`Lakbay.Web` queries to the real runtime type (`String`). Note:
`ProductFilter.destinationId` has this exact same latent SDL/runtime
mismatch and predates this change — not fixed here (out of scope, and
never triggered, since it's only ever passed inside an object literal,
not as a bare `$var: ID`), but worth a dedicated look if a raw-variable
`destinationId` argument is ever added elsewhere.

## Consequences

- Accommodation is now a first-class, independently searchable/sellable
  unit, matching the concrete "search for a budget-friendly Boracay hotel
  for a family" scenario end-to-end: `/stays` → filter to Boracay +
  Family-Friendly → Station 2 Beachfront Inn → two real room options
  (Garden View Room, ₱2,200/night, sleeps 2; Beachfront Family Room,
  ₱3,800/night, sleeps 4) → destination-wide perks/add-ons → the existing
  curated Product offered as a secondary package.
- `Product` is unchanged in meaning — still a curated, bookable multi-day
  itinerary — only its framing in the UI shifted from "the only path" to
  "an optional path alongside browsing accommodations directly."
- Same migration/verification discipline as every prior ADR this session:
  a self-limiting stale-shape check (`accommodation` missing `destination`)
  triggers delete-and-recreate; all three long-running processes stopped
  and restarted in order before reseeding; `dotnet build`/`test` clean
  (8/8), `tsc --noEmit` clean; a targeted `deleteMany` cleaned up one
  round of orphaned pre-reseed documents (confirmed via a
  `$dateTrunc`-grouped count first); a full real-browser pass through the
  exact Boracay scenario above, plus the Destination-page and holiday
  detail-page regression checks, all confirmed working.
- A per-card `useGetRoomTypesQuery` call on the Stays search grid means
  each of the ~14 result cards fires its own small GraphQL request to
  compute "from ₱X" — accepted for now given the catalog's current size;
  revisit with a bulk accommodation-list price lookup only if the
  catalog grows enough for this to matter.
