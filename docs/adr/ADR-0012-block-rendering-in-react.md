# ADR-0012: Umbraco content blocks render through a React block-registry, not Razor

- **Status:** Accepted
- **Date:** 2026-09-06
- **Repo(s) affected:** `Lakbay.Cms`, `Lakbay.Web`

## Context

ADR-0006 established that `Lakbay.Cms` never renders a page or holds
Razor/UI code. Randolf asked the natural follow-up: Umbraco's Block
List/Block Grid editors are how content editors compose a page out of
reusable pieces (a hero banner, a testimonial, a gallery) — with no
Razor views, how does that arrangement actually reach the screen, and can
React handle it? Short answer: yes, and this is a well-established
headless-CMS pattern (Sanity, Contentful, and Storyblok all do the same
thing under different names — "portable text renderers," "component
mapping"), not something Lakbay needs to invent.

## Decision

Content editors compose pages in `Lakbay.Cms`'s Block List/Block Grid
editors as normal. Umbraco's Content Delivery API serializes each block
as JSON: a type identifier (the element type alias, e.g. `heroBanner`,
`testimonialCard`, `destinationGallery`) plus that block's own property
values — an ordered array per page.

`Lakbay.Web` owns a **block registry**: a single mapping from element-type
alias to a React component.

```
const blockRegistry: Record<string, React.ComponentType<any>> = {
  heroBanner: HeroBanner,
  testimonialCard: TestimonialCard,
  destinationGallery: DestinationGallery,
  // ...
};
```

A `BlockList` (or `BlockGrid`) React component walks the JSON array
returned for a page, looks up each block's component by its type alias,
and renders it with that block's properties as props. An unrecognized
alias renders nothing (logged, not thrown) rather than crashing the page
— a content editor adding a new block type in Umbraco before the matching
React component ships should degrade gracefully, not 500 the page.

Types for each block's properties are generated from `Lakbay.Contracts`
(or from Umbraco's own OpenAPI-generated Content Delivery API types, per
current community practice) so a block's shape is checked at compile
time, not discovered at runtime.

## Alternatives considered

- **Server-render blocks from `Lakbay.Cms`** (reintroducing Razor views
  per block) — rejected outright: this is exactly the hybrid ADR-0006
  already rejected, for the same SEO/hydration-friction reasons.
- **A generic "renderer" that interprets block JSON without a registry**
  (e.g. reflection-driven or schema-driven rendering) — rejected: harder
  to reason about, harder to style deliberately per block, and loses the
  ability to give each block real, deliberate React/TypeScript code —
  Lakbay's product lines (Alon, Amihan, Parul, Pamana) each want visually
  distinct treatment, which a generic renderer works against.

## Consequences

- Every new block type requires coordinated work in two repos: the
  Umbraco element type/Block List config in `Lakbay.Cms`, and the
  matching React component + registry entry in `Lakbay.Web`. This is a
  real coordination cost, not a purely one-sided content change — flagged
  explicitly so it isn't assumed to be "just a CMS change."
- Content editors get Umbraco's structured block editor for composing
  pages (which blocks, in what order, with what property values) but not
  a live visual preview of the final React-rendered page inside the
  Umbraco backoffice — that's a known headless-CMS trade-off. A
  Content-Delivery-API-aware preview package (e.g. the kind that renders
  real HTML inside the backoffice using the same headless setup) is worth
  evaluating in a later phase if editors need it, not assumed necessary
  now.
- `Lakbay.Web`'s `Docs/DEVELOPER_HANDBOOK.md` (once written, Phase 0)
  should include "adding a new block type" as one of its worked
  walkthroughs — this is exactly the kind of repeatable, two-repo task
  that benefits from a documented recipe.
