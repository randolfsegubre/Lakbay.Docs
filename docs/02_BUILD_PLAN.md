# Build Plan — Lakbay Platform

**Read this before writing code in any `Lakbay.*` repo.** `01_CLAUDE.md`
is the specification — what to build, and why each decision was made.
This document is the execution plan — what order to build it in, what
"done" means at each step, which repo(s) each step touches, and what gets
documented along the way. If the two ever conflict, `01_CLAUDE.md` wins on
*what*, this file wins on *sequencing*.

## How to use this document

- Work through the phases **in order**. `Lakbay.Web` talks to
  `Lakbay.AvailabilityApi` for all catalog browsing/search from Phase 2 onward
  — and **keeps** talking to it in production; `Lakbay.AvailabilityApi` is real,
  permanently deployed infrastructure (see
  [ADR-0007](adr/ADR-0007-searchapi-is-real-not-mock.md)), not a mock
  that gets swapped out. What changes at Phase 3 is where
  `Lakbay.AvailabilityApi`'s *data* comes from — hand-seeded in Phase 1, synced
  from `Lakbay.Cms` from Phase 3 onward — not which service `Lakbay.Web`
  talks to. This carries the Sphinx-API/Manticore pattern forward
  faithfully: a dedicated search read-model, fed by the CMS, queried by
  the storefront, present in production.
- **Every phase ends with a `05_DEVLOG.md` entry** before moving to the
  next one. One entry per phase minimum — more if a session spans a phase
  boundary or something notable happened mid-phase (a blocked decision, a
  changed assumption, a bug that cost real time).
- A new structural decision made while executing any phase gets its own
  ADR under `docs/adr/` at the moment it's made, not retrofitted later.
- Commit per phase (or per meaningful sub-step within a long phase), not
  as one giant commit at the end.
- Each phase names which repo(s) it touches. A session working in a single
  repo only needs that repo's own `CLAUDE.md` plus the phase section here
  that names it — not the whole document.

## Phase overview

| # | Phase | Repos | Depends on | Produces |
|---|---|---|---|---|
| 0 | Foundation & scaffolding | all six | — | Buildable skeleton in every repo, `Lakbay.Contracts` schema v0, ADR-0001–0007 in place |
| 1 | Search API | Contracts, AvailabilityApi | 0 | Real, permanent GraphQL/Mongo search service, hand-seeded with real PH destination data for now |
| 2 | Storefront against AvailabilityApi | Web | 1 | Browsable catalog site for all four product lines, querying `Lakbay.AvailabilityApi` |
| 3 | Real CMS + sync to AvailabilityApi | Cms, AvailabilityApi | 0, 2 | Umbraco unified content+catalog live; sync mechanism replaces AvailabilityApi's hand-seeded data with real Cms-authored data, zero `Lakbay.Web` code changes |
| 4 | Booking & payments | Booking | 0, 3 | End-to-end bookable holiday in staging, PayMongo sandbox integration |
| 5 | Hosting & go-live | Cms, Booking, Web, AvailabilityApi | 1–4 | Live Philippines-first site, real destination content, IaC-provisioned Azure — all four services deployed |
| 6 | Scale readiness | all | 5 | AKS/Stripe/Azure AI Search evaluated against real traffic, not assumed |

## Phase 0 — Foundation & scaffolding

**Goal:** every repo has a buildable (even if empty) skeleton, and the
shared contract exists before any repo starts consuming it.

**Repos:** all six.

- [x] Create the six repos as git repositories under `Personal_Projects/Lakbay/`
      (done 2026-09-03).
- [x] `Lakbay.Docs` populated: this build plan, `01_CLAUDE.md`,
      `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`,
      `06_SYSTEM_ARCHITECTURE.md`, ADR-0001 through 0007 (done
      2026-09-03 through 2026-09-06).
- [ ] Confirm local environment: `.NET` SDK version for Umbraco 17 (verify
      the exact minimum at scaffold time — Umbraco version support moves
      faster than this doc; this same SDK now also covers `Lakbay.Booking`
      and `Lakbay.AvailabilityApi`, per [ADR-0004](adr/ADR-0004-mockapi-dotnet-not-node.md)),
      Node.js version for `Lakbay.Web` only, and Docker Desktop (for the
      offline-runnable Compose stacks each repo will need — SQL Server for
      Cms/Booking per [ADR-0005](adr/ADR-0005-local-sql-server-not-azure-sql.md),
      MongoDB for AvailabilityApi).
- [ ] `Lakbay.Contracts`: schema v0 — `Product`, `ProductLine`,
      `Destination` GraphQL types, matching the four product lines and the
      cluster/destination table in the Blueprint's market-research
      section. Publish as a versioned package (GitHub Packages or Azure
      Artifacts — pick one and record the choice as an ADR when it's
      decided).
- [ ] `Lakbay.Cms`: empty Umbraco 17 solution scaffolded, boots to the
      install wizard, nothing customized yet — working baseline before
      customization, same discipline as Ophir Mineral Ventures' Phase 0.
- [ ] `Lakbay.Booking`: empty .NET minimal API solution, xUnit test
      project scaffolded alongside it from the start (not deferred).
- [ ] `Lakbay.Web`: empty Next.js (App Router) + Redux Toolkit project,
      `create-next-app` baseline committed before any real pages.
- [ ] `Lakbay.AvailabilityApi`: empty ASP.NET Core + HotChocolate project,
      MongoDB via Docker Compose, boots and serves an introspection query
      with zero resolvers.
- [ ] CI skeleton in every repo (GitHub Actions or equivalent) — even if
      it only runs `dotnet build`/`npm ci && npm run build` at this stage.
      Real test gates arrive with each phase's own work.
- [ ] Each repo gets its own `Docs/DEVELOPER_HANDBOOK.md` stub with the
      exact local setup commands proven in this phase (not written from
      memory afterward — write it the moment the setup actually works).

**Exit criteria:** `dotnet build`/`npm run build` succeeds in every repo
with no errors; `Lakbay.Contracts` schema v0 is committed and referenced
(even if unused) from both `Lakbay.Cms` and `Lakbay.AvailabilityApi`;
ADR-0001–0007 exist; a DEVLOG entry closes the phase.

## Phase 1 — AvailabilityApi

**Goal:** a running, real GraphQL/MongoDB search service that speaks the
`Lakbay.Contracts` schema, hand-seeded with real Philippine destination
data from the Blueprint's market research (not placeholder lorem) as a
starting point — this hand-seeding is temporary, the *service* is not
(see [ADR-0007](adr/ADR-0007-searchapi-is-real-not-mock.md)).

**Repos:** `Lakbay.Contracts`, `Lakbay.AvailabilityApi`.

- Two projects from the start, per [ADR-0009](adr/ADR-0009-availabilityapi-rename-and-split.md):
  the query API (resolvers only, no Service Bus code) and
  `Lakbay.AvailabilityApi.Sync` (the Azure Function project) — even
  though the Function has nothing to consume yet until Phase 3/4, scaffold
  it now so the split isn't retrofitted later.
- Resolvers for `Product`, `ProductLine`, `Destination` queries against
  seeded MongoDB collections, with the faceted-filtering shape (by
  destination, theme, price, date) this service exists for — not just
  flat lookups.
- Seed data: at minimum one real destination per product line (e.g. Coron
  for Alon, Baguio for Amihan, San Fernando Pampanga for Parul, Vigan for
  Pamana) — real names and descriptions from the Blueprint's research, not
  fabricated placeholders, since this data will be visible in Phase 2's
  storefront screenshots.
- Add the `sourceUpdatedUtc` field ([ADR-0010](adr/ADR-0010-last-write-wins-sync.md))
  to every synced document's schema now, even before there's a real sync
  job to populate it from — retrofitting an ordering field after Phase 3's
  sync job exists is exactly the kind of rework worth avoiding.
- Decide the sync trigger this service will use once `Lakbay.Cms` exists
  (Umbraco content-cache-refresher event, Service Bus message, or
  scheduled Hangfire job) — design the resolver/data-access layer so
  swapping hand-seeding for a real sync job in Phase 3 doesn't require
  reshaping the schema or resolvers, only the data-loading path.
- CI check: schema returned by introspection is diffed against
  `Lakbay.Contracts`' published SDL — the guardrail that keeps this
  service's read-model schema from drifting out of sync with
  `Lakbay.Cms`'s write-model schema.
- xUnit + HotChocolate's testing utilities (`IRequestExecutor` test
  helpers), same test stack shape as `Lakbay.Cms`/`Lakbay.Booking` since
  this repo is .NET too (ADR-0004).

**Exit criteria:** a GraphQL Playground/introspection query against a
locally-run `Lakbay.AvailabilityApi` returns real seeded product-line and
destination data with working facet filters (destination, theme, price,
date); schema-diff CI check is green.

## Phase 2 — Storefront against AvailabilityApi

**Goal:** a browsable, deployable-shaped storefront querying
`Lakbay.AvailabilityApi` for all catalog browsing/search — this is the same
querying path it will use in production, not a temporary stand-in.

**Repos:** `Lakbay.Web`.

- RTK Query API slice wired to `Lakbay.AvailabilityApi`'s GraphQL endpoint.
- Catalog/landing pages for all four product lines (Alon, Amihan, Parul,
  Pamana), rendering the seeded destination data from Phase 1, with real
  facet filtering (destination, theme, price, date) exercised end to end
  — this is the point of having a dedicated search service, so it should
  be visibly working here, not deferred.
- Redux Toolkit slices only where state is genuinely client-side (filter
  UI, basket shell — no real checkout yet, that's Phase 4).
- Jest + React Testing Library + MSW component tests; a Playwright smoke
  test that runs `Lakbay.Web` against `Lakbay.AvailabilityApi` in CI.

**Exit criteria:** the storefront is fully browsable and filterable
end-to-end against `Lakbay.AvailabilityApi` with no manual steps; component and
Playwright tests green in CI.

## Phase 3 — Real CMS + sync to AvailabilityApi

**Goal:** `Lakbay.Cms` goes live as the real content/catalog authoring
system, and a real sync mechanism replaces `Lakbay.AvailabilityApi`'s Phase 1
hand-seeded data with data authored in Umbraco — with **zero
`Lakbay.Web` code changes**, since `Lakbay.Web` never stops talking to
`Lakbay.AvailabilityApi`. This is the actual proof the shared contract held:
not a backend swap, a data-source swap underneath a service whose
interface never moved.

**Repos:** `Lakbay.Cms` (primary), `Lakbay.AvailabilityApi` (sync consumer),
`Lakbay.Web` (no code changes expected — verification only).

- Content tree (pages, landing pages, block-list components) and Products
  tree (holiday records: itinerary, price bands, departure dates,
  inclusions, media, geo, product-line taxonomy) per ADR-0001.
- `Lakbay.Web`'s block registry ([ADR-0012](adr/ADR-0012-block-rendering-in-react.md))
  gets its first real entries here, matched one-for-one against whatever
  block/element types are actually configured in `Lakbay.Cms`'s Block
  List/Grid setup — build both sides of a given block together, not the
  Umbraco side first and the React side "later."
- Content Delivery API enabled on `Lakbay.Cms`; GraphQL layer on top
  matching `Lakbay.Contracts` (see the open item in `01_CLAUDE.md` about
  which package to use). This is the path `Lakbay.Web` may use for
  non-search page content per ADR-0007 — not required for this phase's
  exit criteria, but the layer should exist.
- Build the sync mechanism decided in Phase 1: on publish in `Lakbay.Cms`,
  push the updated product/destination record into `Lakbay.AvailabilityApi`'s
  MongoDB collections in the same shape its resolvers already expect.
- Real content authored for at least the same destinations seeded in
  Phase 1, this time through the actual Umbraco backoffice, and confirmed
  to arrive in `Lakbay.AvailabilityApi` via the sync mechanism.
- xUnit + FluentAssertions + NSubstitute for domain logic; Testcontainers
  (SQL Server) for integration tests against the real Content Delivery
  API; an integration test proving a `Lakbay.Cms` publish results in the
  matching `Lakbay.AvailabilityApi` document updating.

**Exit criteria:** publishing a product change in `Lakbay.Cms`'s
backoffice is reflected in `Lakbay.AvailabilityApi`'s query results without any
`Lakbay.Web` deployment or code change; the storefront shows the newly
synced (real, Umbraco-authored) content in place of Phase 1's hand-seeded
data.

## Phase 4 — Booking & payments

**Goal:** an end-to-end bookable holiday, from catalog page to confirmed
order, in a staging environment.

**Repos:** `Lakbay.Booking` (primary), `Lakbay.AvailabilityApi` (availability
sync + real-time push, ADR-0008), `Lakbay.Web` (checkout UI + live
listing updates), `Lakbay.Cms` (Service Bus event consumption, if
availability affects catalog display).

- CQRS command/query handlers per ADR-0002.
- Basket, availability calendar, PayMongo checkout integration (sandbox).
- **The actual double-booking fix**: `ConfirmBookingCommandHandler` uses a
  single atomic, conditional SQL `UPDATE` (`WHERE AvailableCount > 0`),
  never a separate read-then-write check — see
  [ADR-0011](adr/ADR-0011-atomic-availability-decrement.md). This is the
  correctness guarantee; everything below is a UX/freshness improvement
  and does not substitute for it.
- A concurrency test simulating two simultaneous confirm-booking calls
  against `AvailableCount = 1`, asserting exactly one succeeds — required
  before this phase's exit criteria are considered met, not a nice-to-have.
- Hangfire-scheduled confirmation email/SMS (Twilio) on `BookingConfirmed`.
- Service Bus wiring: `Lakbay.Booking` publishes `AvailabilityChanged`;
  `Lakbay.AvailabilityApi.Sync` (the Azure Function, ADR-0009) consumes it,
  applies it with the last-write-wins timestamp guard (ADR-0010), updates
  MongoDB, and pushes a change notification to Azure SignalR Service;
  `Lakbay.Web` subscribes per listing page and invalidates/refetches just
  that item's RTK Query cache entry on notification (ADR-0008). The
  query-serving `Lakbay.AvailabilityApi` project is untouched by any of
  this. Separately, `Lakbay.Cms` consumes Service Bus events per ADR-0003
  if availability affects catalog display.
- Contract tests run against both the PayMongo sandbox and a fake
  `IPaymentGateway` implementation (Liskov Substitution check, per the
  Architecture & Patterns Guide).

**Exit criteria:** a full booking (browse → basket → PayMongo sandbox
checkout → confirmation email/SMS) succeeds in staging; contract tests
green for both real and fake payment gateway implementations; a second
browser tab showing the same listing reflects a sold-out state within
seconds of a booking being confirmed, with no manual refresh.

## Phase 5 — Hosting & go-live

**Goal:** the platform is live, on real infrastructure, with real content.

**Repos:** `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web`, and
`Lakbay.AvailabilityApi` — all four on Azure Container Apps. `Lakbay.AvailabilityApi`
deploys to production alongside the others; it was never dev-only (see
[ADR-0007](adr/ADR-0007-searchapi-is-real-not-mock.md)).

- Terraform or Bicep IaC (pick one — record the choice as an ADR) for
  Container Apps, Azure SQL, Redis, Service Bus, SignalR, Entra External
  ID, and the MongoDB instance backing `Lakbay.AvailabilityApi` (Azure Cosmos DB
  for MongoDB, or a managed MongoDB Atlas instance — an open choice to
  resolve in this phase, not decided yet).
- Real destination/product content replacing every placeholder from
  earlier phases, authored in `Lakbay.Cms` and confirmed synced into
  `Lakbay.AvailabilityApi`.
- DNS cutover, Application Insights wired, security/QA pass (headers,
  Lighthouse, cross-browser — same discipline as Ophir Mineral Ventures'
  Phase 7).
- Legal/compliance check on DOT accreditation requirements closed or
  explicitly deferred with a named owner.
- `Docs/DEVELOPER_HANDBOOK.md` and `USER_GUIDE.md`-equivalent (for
  whoever operates the CMS day to day) finalized in `Lakbay.Cms`.

**Exit criteria:** the live site is reachable on its real domain, serves
real PH holiday content via `Lakbay.AvailabilityApi`, and a real booking can be
completed with a real (not sandbox) payment method.

## Phase 6 — Scale readiness

**Goal:** revisit every "adopt later" verdict from the Blueprint's
technology-stack table against real traffic data, not assumption.

**Repos:** all, as needed.

- AKS vs. Container Apps: only move if Container Apps' per-second billing
  has genuinely stopped being cheaper than reserved AKS nodes.
- Stripe/Xendit: only add if international-card volume or SEA expansion
  justifies a second payment gateway.
- `Lakbay.AvailabilityApi`'s MongoDB → a managed search-specific engine (Azure
  AI Search, or MongoDB Atlas Search): only if facet/query complexity
  outgrows what MongoDB's own indexing handles well — the same kind of
  scale question Hotelplan answered by moving to Manticore, worth revisiting
  with real query-pattern data rather than assumed on day one.
- Formal ETL (Azure Data Factory): only once real BI/reporting volume
  exists.

**Exit criteria:** each revisited item gets its own ADR recording the
decision made with real data, superseding (not silently replacing) the
original Blueprint verdict.
