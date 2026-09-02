# ADR-0007: Lakbay.SearchApi is a real, permanent search service — not a disposable mock

- **Status:** Accepted — supersedes the framing of `Lakbay.MockApi` in
  ADR-0004, the Lakbay Blueprint, and the Lakbay System Map (all of which
  described this repo as "dev/demo only, never deployed to production")
- **Date:** 2026-09-06
- **Repo(s) affected:** `Lakbay.SearchApi` (renamed from `Lakbay.MockApi`),
  `Lakbay.Cms`, `Lakbay.Web`

## Context

The original plan scoped this repo as a disposable mock, following
Randolf's own first-message description of it as "a dummy database or
mockup database just like our Sphinx-Api and mantincore." On direct
questioning (2026-09-06: "Aren't we going to use it as our API application
for real just like Sphinx-Api?"), checking the actual commit history
behind that precedent showed the premise was wrong: `api-sphinx` at
Hotelplan is real, production, business-critical code — 55 commits,
described in [[technical_playbook]] as "the highest-risk-per-change repo
(direct Manticore/Sphinx search-engine query logic, price/availability
correctness)," called by multiple consumer apps (PIM, AgentsX) for real
ski/Lapland product filtering and travel-dates search. It was never a
mock. "Dummy" in the original framing was Randolf's own shorthand at the
time, not an accurate description of what Sphinx-API actually was.

What `api-sphinx` actually is: a **dedicated search/query microservice**,
built on Manticore (a real full-text/faceted search engine, a fork of
Sphinx — the source of both names), sitting apart from the CMS because
fast, filtered catalog browsing (destination, theme, price, date) is a
different workload from content authoring, and a general-purpose CMS
content-cache index isn't built to serve it well at scale. This is the
exact need already flagged as "later" in the Lakbay Blueprint's roadmap
("consider Azure AI Search if catalog facet complexity outgrows Examine")
— the honest read is that this need doesn't start later, it's exactly
what this repo should be from the start, using MongoDB instead of
Manticore.

## Decision

Rename `Lakbay.MockApi` to **`Lakbay.SearchApi`**. It is a real,
permanently deployed service — ASP.NET Core + HotChocolate +
MongoDB.Driver (stack unchanged from ADR-0004) — that:

- Holds a **read-optimized, denormalized copy** of the product catalog
  (holidays, product lines, destinations) in MongoDB, built for fast
  faceted search and filtering.
- Is kept in sync **from** `Lakbay.Cms` (the authoring source of truth,
  per ADR-0001) via a sync mechanism triggered on publish — the specific
  trigger (Umbraco content-cache-refresher event, a Service Bus message,
  or a scheduled Hangfire job) is an open implementation decision for
  Phase 1, not decided by this ADR.
- Is what `Lakbay.Web` actually queries for **catalog browsing, search,
  and filtering** in production — not a temporary stand-in repointed away
  from once `Lakbay.Cms` exists.
- `Lakbay.Cms`'s own Content Delivery API/GraphQL layer remains the
  right path for content that isn't search-driven (e.g. rendering a
  single known landing page by slug) — this ADR does not remove that
  path, it adds a purpose-built one for search.

This mirrors `Lakbay.Booking`'s CQRS split at the level of the whole
system: `Lakbay.Cms` is the write/authoring side for content and catalog
data, `Lakbay.SearchApi` is a dedicated, denormalized read side optimized
for the query pattern that actually matters to shoppers.

## Alternatives considered

- **Keep the original "mock, thrown away in Phase 3" framing** —
  rejected: it was based on a wrong premise about what the real precedent
  was, confirmed by re-checking the actual commit history rather than
  relying on memory of it.
- **Query `Lakbay.Cms`'s Content Delivery API directly for all storefront
  needs, including search/filtering, and drop this repo entirely** —
  rejected: this is exactly the shape Hotelplan moved *away* from by
  building a dedicated search service; Umbraco's Examine index is a
  general-purpose content index, not built for the same query patterns
  (faceted filtering across price/date/theme/destination) Manticore
  handled at Hotelplan. Reintroducing that limitation for Lakbay when the
  real precedent already shows why not to would be ignoring the evidence.
- **Build the search layer directly inside `Lakbay.Cms`** (e.g. a custom
  Examine index with heavy faceting) — rejected: revisits the same
  reasoning as ADR-0001/ADR-0003's database-separation logic. A
  search-optimized read model has a different shape and update pattern
  than Umbraco's own content cache; forcing it into the same process
  risks the same kind of contention ADR-0003 avoided for booking data.

## Consequences

- `Lakbay.SearchApi` is now a Phase 5 (go-live) deployable, not a
  dev-only tool retired at Phase 3. `02_BUILD_PLAN.md`'s phase table and
  Phase 1/3/5 sections need rewriting to reflect this — done alongside
  this ADR, not deferred.
- A real sync mechanism (Cms → SearchApi) must be designed and built —
  this is genuinely new scope the original "mock" framing didn't have,
  since a throwaway mock only needed hand-seeded data, not a live sync
  pipeline. Flagged as an explicit Phase 1 task, not assumed solved.
- The Lakbay System Map diagram's solid/dashed "production vs. dev-only"
  distinction for this repo no longer applies — it needs redrawing, not
  just relabeling. Done alongside this ADR.
- `Lakbay.Contracts`' schema-diff CI check (originally framed as "keeping
  the mock from drifting from the real backend") is reframed as "keeping
  the search read-model's schema from drifting from `Lakbay.Cms`'s
  write-model schema" — same mechanism, more accurate purpose.
