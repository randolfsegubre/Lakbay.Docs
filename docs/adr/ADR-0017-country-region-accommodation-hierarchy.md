# ADR-0017: Country → Region → Destination → Accommodation content hierarchy

**Status:** Accepted — 2026-09-08

## Context

The user's own framing, verbatim: "The content tree doesn't look like ECMS
or PCMS at all. If you remember ECMS, we have Product (Ski Holidays) under
it are Countries, under it are Regions, under it are resorts, under it are
accommodations... We want to have the same structure so details of
location can be highlighted... but in a way that will fit our business
model."

Lakbay's Products tree was flat: `Product` carried `productLine` +
`destination` (Content Picker) + a plain `region: string` field, and — from
ADR-0016's earlier accommodation/activities work — two flat text fields
(`accommodationName`/`accommodationDescription`) instead of a real
accommodation entity. No level below Product could carry its own
highlights independent of a specific holiday package.

## Grounding in the real ECMS structure

Before designing anything, an Explore agent read the real ECMS repo
(`Hotelplan.Inghams.V2.CMS`, cloned locally at `D:\_DEV\HPUK` for read-only
reference — never pushed to). Findings:

- The real tree is `Product → Geography → Country → Region → Resort →
  Accommodation`, built via Umbraco Compositions, and **duplicated per
  product line** — `countrySki`, `countryWalking`, `countryLapland` are
  separate content-type instances, not one shared node. Resort can sit
  directly under Country when a country has no meaningful region split;
  Lapland skips the Region level entirely.
- **Crucially, the actual "highlights" content — descriptions, features,
  images, best-for splits, ratings — does not live in Umbraco fields at
  all.** It lives in a separate external system ("Product CMS" / PCMS),
  joined to each Umbraco node by a `{level}Code` property and fetched at
  render time. Umbraco only holds the addressable page tree plus
  free-form block-grid editorial content.

Lakbay's own [ADR-0001](ADR-0001-unify-ecms-pcms.md) already rejected that
two-system split — ECMS and PCMS were explicitly unified into one Umbraco
solution for cost reasons, the biggest lever vs. the Hotelplan-style
split. The adaptation for Lakbay is therefore: keep ECMS's **tree shape**,
but hold highlights content directly as real Umbraco fields at every
level, the same way `Destination`/`Product` already did.

## Decision

New tree:

```
ProductLine (Islands/Highlands/Festivals/Heritage)     [existing, unchanged]
Country ("Philippines")                                 [NEW — one shared node]
  └─ Region (e.g. "Palawan", "Cordillera Administrative Region")   [NEW — one per ProductLine]
       └─ Destination (Coron, El Nido, Boracay, ...)     [EXISTING — reparented under Region]
            └─ Accommodation (Coron Bayside Inn, ...)    [NEW — promoted off Product's two flat strings]
Product (the bookable curated package)                   [EXISTING — now references Accommodation]
```

**Country is one shared node, not duplicated per ProductLine.** Confirmed
with the user before implementing: unlike ECMS's real multi-country scale,
Lakbay's entire catalog is one country today. Region and Destination stay
scoped one-per-ProductLine, matching how `Destination` already worked and
ECMS's own real per-product duplication — a Region name can legitimately
repeat across lines (e.g. "Cordillera Administrative Region" exists as two
distinct nodes, one for Highlands with its own hiking/climate highlights,
one for Heritage with its own rice-terraces/culture highlights).

**Accommodation is promoted to a real, reusable entity.** This directly
resolves the "revisit only if..." case ADR-0016 explicitly flagged when it
chose flat strings over a separate entity: now that the geography
restructure called for real content nodes throughout, giving
Accommodation the same treatment costs nothing extra and unlocks sharing
one accommodation's copy across multiple products at the same destination
later, without duplicating text.

**ECMS's "Resort" maps onto Lakbay's already-established "Destination"**
— no rename. Every doc, ADR, and repo already uses "Destination"; renaming
it would be churn for no benefit.

**Highlights are a newline-per-line Textarea in Cms**, reusing the exact
`ParseActivityLines` helper ADR-0016 already built for
`includedActivities`/`optionalActivities` — no new parsing mechanism.

**Migration, not backward compatibility.** This is disposable pre-launch
dev data — no real bookings, no real customers. The seeder detects a
pre-restructure database (Region content type missing) and deletes +
recreates `Product`/`Destination`/`ProductLine` in one pass, the same
self-limiting shape ADR-0015 and ADR-0016 already established for their
own content-type migrations. No in-place data transform was written or
needed.

**A real, pre-existing data inconsistency fixed as part of this
migration**: some of the original 14 destinations used Philippine
province names for their `region` field ("Palawan", "Ilocos Sur") while
others used proper regional groupings ("Western Visayas", "Cordillera
Administrative Region") — inconsistent granularity that a flat string
field let slip in silently. The new Region entity forces one consistent,
traveler-recognizable granularity, applied uniformly across all 14
destinations (12 real Region nodes total).

## Consequences

- Every holiday detail page now shows a `Philippines → Region →
  Destination` breadcrumb with the region's own highlights (and the
  country's, dimmed) shown inline — the actual "details of location
  highlighted" the user asked for, without a new page or route.
- The accommodation card is now backed by a real, reusable content node
  with its own highlights, not two flat strings.
- Dedicated `/regions/[id]` or `/countries/[code]` browsing pages —
  closer to how a visitor might explore ECMS's real site — were
  explicitly scoped out of this pass (confirmed with the user) and are a
  natural next step, not silently dropped. The GraphQL `regions`/`country`
  queries already exist server-side for whenever that page gets built.
- `Query.cs` needed no `AddType<>` registration for the new C# records —
  confirmed again this session that HotChocolate infers GraphQL object
  types from plain nested POCOs automatically, same as every earlier
  schema addition this platform has made.
- The next content addition to this schema should check the *existing*
  content type's property list the same way this migration did, on any
  machine where the type might predate the change — this is now the
  third time that exact check has been needed (ADR-0015, ADR-0016, this
  ADR), so it's a genuinely established pattern in this codebase, not a
  one-off.
