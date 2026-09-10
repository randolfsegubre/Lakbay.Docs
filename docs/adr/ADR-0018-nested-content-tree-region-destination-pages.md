# ADR-0018: Nested Region/Destination pages in the Content tree

**Status:** Accepted — 2026-09-08

## Context

Following ADR-0017's confirmation that Lakbay's Products tree already
matches PCMS's real flat + Content-Picker pattern, the user asked for the
Content tree (the ECMS-equivalent) to get ECMS's real *depth* too — ECMS
nests `Home → Ski Holidays → Ski Resorts → Andorra → Arinsal → Hotel
Xalet Besoli`, a genuine 6-level parent-child page tree. Lakbay's Content
tree was still shallow: Home plus 4 flat ProductLine landing pages, no
region- or destination-level pages.

Mid-implementation, the user raised a second, related point: "I don't see
accommodation... in Inghams, [hotels] are the types of accommodation
shown. That is what they sell and market... double check Inghams and
Inntravel, I might be wrong." Re-verified both live:

- **Inghams**: accommodation genuinely is the primary sellable unit —
  every search result *is* a named hotel with its own price, board basis,
  and flight bundled in ("Hotel Serre Palas, Les 2 Alpes — From £696pp, 7
  nights, Bed & Breakfast, flight from London Gatwick included").
- **Inntravel**: a different, closer-to-Lakbay shape — the sellable unit
  is the *named holiday* ("Timeless Tuscany"), but its one included
  accommodation gets a full, prominent, dedicated section: its own tab
  ("Overview | Itinerary | **Accommodation** | Extend your stay |
  Prices & travel | Reviews"), a real hotel name, gallery, description.

Lakbay's actual positioning — curated themed packages, not "pick your own
hotel from a list" — matches Inntravel's pattern, not Inghams'. So
ADR-0017's data model (`Product` owns one `Accommodation`) was already
right; what was missing was **presentation weight**. Accommodation existed
only as a small card, easy to miss — exactly what the user reported.

## Decision

**1. Real nested Content-tree pages**, matching ECMS's actual depth:

```
Home
  └─ ProductLine landing page (existing)
       └─ Region landing page (NEW — 12, one per Region)
            └─ Destination landing page (NEW — 14, one per Destination)
```

Created with real Umbraco parent-child nesting (`contentService.Create(name,
PARENT.Id, alias)`), unlike the Products tree's deliberately flat
Country/Region/Destination/Accommodation (ADR-0017) — the two trees now
use genuinely different shapes on purpose, matching their real Hotelplan
counterparts (ECMS nests, PCMS doesn't) rather than being accidentally
inconsistent.

**2. `slug` added to `Region` and `Destination`** (`Lakbay.Contracts`,
mirroring `Product.slug`) — the stable join key a Content-tree page uses
to say "I'm about this Region/Destination," the same "plain text, not a
picker" reasoning `ContentTreeSeeder` already used for `productLineCode`
(no ordering dependency between the two independently-booting seeders).

**3. `getRegionPageContent`/`getDestinationPageContent`** in
`cmsContentApi.ts` filter by content type across the whole tree
(`?filter=contentType:regionLandingPage`) and match by `slug`
client-side — not a direct route lookup, and not a "fetch this specific
parent's children" call either, since Region/Destination pages nest under
a *different* parent per line/region (no single well-known parent ID like
`home` to hardcode). Same "don't rely on Umbraco's own route resolution"
spirit as the existing `getLandingPageContent`, generalized further.

**4. Accommodation given real visual prominence**, everywhere it appears:
the new Destination page shows it as a full section (photo, name,
description, highlights — matching Inntravel's dedicated tab); the
existing holiday detail page's "Where you'll stay" card was upgraded from
text-only to a two-column layout with the same photo. **No fabricated
photography**: `Accommodation.heroImageUrl` reuses the same real
Wikimedia photo already sourced for that destination's Product — never a
photo captioned as if it depicts the specific invented property name
("Coron Bayside Inn" isn't a real business). Honest: it's genuinely a
photo of the place; dishonest would be claiming it shows a building that
doesn't exist.

**5. `collections/[code]/page.tsx` gained an "Explore by region" grid**
above the existing flat product grid — additive, not a replacement; a
visitor can still browse all holidays in a line directly, or drill down
by region first.

## Consequences

- The Content tree now has the same 4-level depth ECMS has, each level a
  real, addressable, editorially-rich page — not just data joined onto a
  product page.
- Accommodation is no longer easy to miss: it has a real photo and its
  own prominent section on two different page types now (destination
  page, holiday detail page).
- Copy on the new Region/Destination pages is intentionally shorter than
  the Products tree's own descriptions/highlights (ADR-0017) — this is
  the editorial overlay, covering the same real facts, not a duplicate of
  the structured data.
- Migration followed the now-four-times-established pattern (ADR-0015,
  0016, 0017, this one): a self-limiting stale-shape check
  (`regionLandingPage` missing) triggers a full delete-and-recreate,
  never an in-place transform, since this remains disposable pre-launch
  dev data.
- Dedicated top-level `/regions`/`/countries` index pages (browsing
  regions independent of any product line) remain out of scope — this
  pass nests everything under its owning ProductLine, matching what was
  actually asked for.
