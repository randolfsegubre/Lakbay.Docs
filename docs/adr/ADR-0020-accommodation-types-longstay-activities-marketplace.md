# ADR-0020: Real accommodation types, long-stay pricing, room photos, and a fair-price Activities marketplace

**Status:** Accepted — 2026-09-08

## Context

Four related gaps raised in one message, right after ADR-0019 shipped:

1. **Room Types had no photo at all.** Accommodation already had one, but
   the room cards were text-only.
2. **Every accommodation read as "hotel-shaped."** The user pointed out
   that real destinations offer apartments, Airbnbs, and more — Lakbay's
   `Accommodation` had no field for this.
3. **No option for open-ended/long-stay travelers.** Foreigners who
   extend indefinitely (the Philippines allows in-country visa
   extensions up to 36 months) need a different price shape than a dated
   seasonal `PriceBand`.
4. **No way to book activities independently at a fair, fixed price.**
   The user's own framing: buying directly from a local risks getting
   scammed or overpriced; they want flexibility without a rigid Product
   package's schedule.

Researched all four live before designing anything, matching this
session's established practice:

- **Real Philippine accommodation categories** (DOT-accredited terms plus
  what's actually listed on Airbnb in these exact destinations today):
  Hotel, Resort, Apartel (independent furnished apartments leased
  longer-term), Pension House (private/family-run boarding house),
  Hostel, Homestay/Guesthouse, Vacation Rental (Airbnb-style private
  home) — Coron alone has real listings across apartelles, transient
  houses, and Airbnb houses, not just hotels.
- **Long-stay pricing**: real Philippine long-term listings commonly
  discount 30-50% off nightly rate for a monthly stay, no fixed end date
  required.
- **The fair-price activity-booking model** (Klook/GetYourGuide/Viator):
  the fix for "named the price only after you're committed" is a fixed
  price agreed before you go, with every inclusion (guide, gear, fees,
  meals) itemized in writing — a vetted ₱1,500-2,500 island-hopping trip
  versus a ₱500 sidewalk pitch with no accountability.

Confirmed with the user via `AskUserQuestion` before designing: (1) the
real DOT-based category list above, as a new `AccommodationType` field;
(2) long-stay modeled as a monthly rate on individual Room Types, not an
Accommodation-level flag; (3) a full new top-level `Activity` entity,
independently browsable and bookable per destination, across all 14
destinations — not a smaller upgrade to the existing text-only
`optionalAddOns`.

## Decision

**1. `RoomType` gains `heroImageUrl` and `monthlyRatePhp`** (both
nullable). Photos reuse the owning Accommodation's own real photo — never
a fabricated distinct-room interior, extending ADR-0018's photo-honesty
rule one level deeper. Monthly rates are seeded only where a longer stay
is realistic (Baguio, Cebu, Siargao, La Union, Boracay — real Philippine
digital-nomad-friendly spots per research), computed as roughly 35% off
the nightly rate over 30 nights, matching the real discount pattern
found.

**2. `Accommodation` gains `type: AccommodationType!`** — a new 7-value
enum (`HOTEL`, `RESORT`, `APARTEL`, `PENSION_HOUSE`, `HOSTEL`,
`HOMESTAY`, `VACATION_RENTAL`). All 14 existing accommodations were
reassigned a real, description-grounded type rather than left as
`HOTEL` by default — e.g. Coron Bayside Inn → Pension House, Calle
Crisologo Heritage House → Vacation Rental (guests get the whole
restored ancestral house, not individual rooms), Session Road Pine
House → Apartel (Baguio's cool climate and remote-work reputation make
it the property reframed for long-stay guests).

**3. New top-level `Activity` entity** — independently bookable at a
fixed, agreed-upfront price, embedding a full `Destination` (like
Product/Accommodation, not RoomType's thin id reference) since Activities
are cross-destination browsable the same way Accommodation now is. 14
real activities seeded (one per destination), each with a realistic
fixed PHP price (₱500-1,900/person) and itemized inclusions grounded in
the Klook/GetYourGuide research — e.g. Coron's "Ultimate Island Hopping
Tour" (₱1,800, full-day, includes licensed boatman, snorkeling gear,
lunch, entrance fees, life jackets).

**4. New `/activities` route** (destination filter, client-side over the
full ~14-item list — same pattern `/stays` already uses) plus compact
"Activities near your stay" / "Book activities here" teaser sections on
the Accommodation detail page and the Destination page, linking into the
full list. This is what actually closes the loop on "give them
flexibility, no rigid package schedule": pick a room (Stays, ADR-0019),
then add real, fixed-price activities on your own terms — `Product`
stays available as a secondary "ready-made package" alongside it,
unchanged in meaning.

**5. A real, pre-existing schema gap found and fixed twice over**:
HotChocolate serves a plain C# `string?` parameter as GraphQL `String`,
not `ID` — the exact ADR-0019 trap. `Query.activities(destinationId:
String)` was declared `String` from the start this time, avoiding a
repeat of that bug.

**6. A real orphaned-generation bug found during this reseed, distinct
from every prior one**: the established `$dateTrunc`-at-hour-granularity
cleanup check silently merged two genuinely different generations into
one bucket, because ADR-0019's reseed and this one both ran within the
same clock hour — the hour-level check reported "one generation, no
orphans" when there were actually two, and a GraphQL query surfaced 28
accommodations (14 duplicated with a stale default `HOTEL` type) instead
of 14. Fixed by re-running the grouping at minute granularity, which
correctly separated the two generations for a clean targeted `deleteMany`.
Noted here as a real gap in the "hour-bucket" convenience this session
had settled into — minute-level grouping is the safer default whenever
multiple reseeds might land inside the same hour.

## Consequences

- Accommodation now has a real, varied category (not a hidden "every
  stay is a Hotel" assumption), a genuine long-stay price option on
  select rooms, and photos on every room card.
- A fair-price, independently bookable activities marketplace exists
  end-to-end, directly answering the "don't get scammed buying from a
  local" concern with the same fixed-price-and-itemized-inclusions
  pattern real platforms use.
- Same migration/verification discipline as every prior ADR this
  session: stale-shape migration triggered a real delete-and-recreate;
  all three long-running processes restarted in order before reseeding;
  `dotnet build`/`test` clean (8/8), `tsc --noEmit` clean; orphaned
  pre-reseed documents cleaned up (this time requiring minute-level
  grouping, see above); a full real-browser pass confirming room photos,
  the long-stay callout, the Type filter, and the Activities marketplace
  all render and link correctly.
- `ProductFilter.destinationId`'s ID/String SDL mismatch (flagged in
  ADR-0019) remains unfixed and out of scope — a separate suggested task
  already exists for it.
