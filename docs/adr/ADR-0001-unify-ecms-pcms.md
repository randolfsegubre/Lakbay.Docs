# ADR-0001: Unify ECMS and PCMS into a single Umbraco solution

- **Status:** Accepted
- **Date:** 2026-08-29
- **Repo(s) affected:** `Lakbay.Cms`

## Context

The enterprise precedent this platform is modeled on (Hotelplan's ECMS /
PCMS estate) ran editorial content and product-catalog data as two
separately deployed systems, integrated over an internal API — each with
its own App Service, its own release pipeline, its own auth story between
them. That separation earns its cost at large scale with independent
teams owning each side. At Lakbay's scale (a small team, one Umbraco
solution realistically ownable by one or two developers) it is pure
overhead: two things to host, two things to patch, a network hop and an
eventual-consistency window between a landing page and the product it
references.

## Decision

Model both editorial content and the product catalog as Umbraco content
trees in **one** Umbraco 17 solution/startup (`Lakbay.Cms`):

- A "Content" tree — pages, landing pages, inspiration articles,
  navigation, block-list components.
- A "Products" tree — holiday/tour records: itinerary, price bands,
  departure dates, inclusions, media, geo, product-line taxonomy.

Content nodes reference product nodes natively (Content Picker / Block
List element) — an in-database relation, indexed in the same Examine
(Lucene) index. Both trees are exposed together through Umbraco's Content
Delivery API, with a GraphQL layer on top matching `Lakbay.Contracts`.

## Alternatives considered

- **Keep them separate, integrate over REST/GraphQL** (the enterprise
  pattern, as-is) — rejected: doubles hosting cost (two App Services, two
  databases) and introduces an eventual-consistency window between a
  content edit and the product it references, for no benefit at this team
  size.
- **A separate headless commerce module for products, Umbraco for content
  only** — rejected for the same reason; also would have meant learning
  and operating a second platform (e.g. a dedicated PIM) instead of
  extending the one already chosen.

## Consequences

- Faster content editing (no cross-system reference lookups), one deploy
  pipeline, one database to operate.
- Content and product-catalog data now scale together — if catalog write
  volume or query complexity ever needs to scale independently of content
  publishing, this decision should be revisited with a new ADR, not
  silently worked around.
- Umbraco's NuCache is tuned for read-heavy, publish-then-cache workloads.
  This unification is scoped to content and catalog *data* specifically
  because both fit that profile — it does **not** extend to transactional
  booking data. See [ADR-0003](ADR-0003-separate-booking-data.md).
