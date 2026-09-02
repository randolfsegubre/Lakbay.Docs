# Build Plan — Lakbay Platform

**Read this before writing code in any `Lakbay.*` repo.** `01_CLAUDE.md`
is the specification — what to build, and why each decision was made.
This document is the execution plan — what order to build it in, what
"done" means at each step, which repo(s) each step touches, and what gets
documented along the way. If the two ever conflict, `01_CLAUDE.md` wins on
*what*, this file wins on *sequencing*.

## How to use this document

- Work through the phases **in order**. Lakbay.Web is deliberately built
  against `Lakbay.MockApi` *before* `Lakbay.Cms` exists (Phase 2 before
  Phase 3) — this is not corner-cutting, it's the whole point of carrying
  the Sphinx-API/Mantincore pattern forward: frontend work is never
  blocked on backend availability.
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
| 0 | Foundation & scaffolding | all six | — | Buildable skeleton in every repo, `Lakbay.Contracts` schema v0, ADR-0001–0003 in place |
| 1 | Mock backend | Contracts, MockApi | 0 | GraphQL/Mongo service serving seeded PH destination data |
| 2 | Storefront against the mock | Web | 1 | Browsable catalog site for all four product lines, zero live backend |
| 3 | Real CMS | Cms | 0, 2 | Umbraco unified content+catalog; Web repointed at it with no frontend code changes |
| 4 | Booking & payments | Booking | 0, 3 | End-to-end bookable holiday in staging, PayMongo sandbox integration |
| 5 | Hosting & go-live | Cms, Booking, Web, MockApi(non-prod) | 1–4 | Live Philippines-first site, real destination content, IaC-provisioned Azure |
| 6 | Scale readiness | all | 5 | AKS/Stripe/Azure AI Search evaluated against real traffic, not assumed |

## Phase 0 — Foundation & scaffolding

**Goal:** every repo has a buildable (even if empty) skeleton, and the
shared contract exists before any repo starts consuming it.

**Repos:** all six.

- [x] Create the six repos as git repositories under `Personal_Projects/Lakbay/`
      (done 2026-09-03).
- [x] `Lakbay.Docs` populated: this build plan, `01_CLAUDE.md`,
      `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`, ADR-0001 through 0003
      (done 2026-09-03).
- [ ] Confirm local environment: `.NET` SDK version for Umbraco 17 (verify
      the exact minimum at scaffold time — Umbraco version support moves
      faster than this doc; this same SDK now also covers `Lakbay.Booking`
      and `Lakbay.MockApi`, per [ADR-0004](adr/ADR-0004-mockapi-dotnet-not-node.md)),
      Node.js version for `Lakbay.Web` only, and Docker Desktop (for the
      offline-runnable Compose stacks each repo will need — SQL Server for
      Cms/Booking per [ADR-0005](adr/ADR-0005-local-sql-server-not-azure-sql.md),
      MongoDB for MockApi).
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
- [ ] `Lakbay.MockApi`: empty ASP.NET Core + HotChocolate project, MongoDB
      via Docker Compose, boots and serves an introspection query with
      zero resolvers.
- [ ] CI skeleton in every repo (GitHub Actions or equivalent) — even if
      it only runs `dotnet build`/`npm ci && npm run build` at this stage.
      Real test gates arrive with each phase's own work.
- [ ] Each repo gets its own `Docs/DEVELOPER_HANDBOOK.md` stub with the
      exact local setup commands proven in this phase (not written from
      memory afterward — write it the moment the setup actually works).

**Exit criteria:** `dotnet build`/`npm run build` succeeds in every repo
with no errors; `Lakbay.Contracts` schema v0 is committed and referenced
(even if unused) from both `Lakbay.Cms` and `Lakbay.MockApi`; ADR-0001–0003
exist; a DEVLOG entry closes the phase.

## Phase 1 — Mock backend

**Goal:** a running GraphQL/MongoDB service that speaks the `Lakbay.Contracts`
schema, seeded with real Philippine destination data from the Blueprint's
market research (not placeholder lorem).

**Repos:** `Lakbay.Contracts`, `Lakbay.MockApi`.

- Resolvers for `Product`, `ProductLine`, `Destination` queries against
  seeded MongoDB collections.
- Seed data: at minimum one real destination per product line (e.g. Coron
  for Alon, Baguio for Amihan, San Fernando Pampanga for Parul, Vigan for
  Pamana) — real names and descriptions from the Blueprint's research, not
  fabricated placeholders, since this data will be visible in Phase 2's
  storefront screenshots.
- CI check: schema returned by introspection is diffed against
  `Lakbay.Contracts`' published SDL — this is the guardrail from ADR
  discussions that keeps the mock and the eventual real backend from
  silently drifting apart.
- xUnit + HotChocolate's testing utilities (`IRequestExecutor` test
  helpers), same test stack shape as `Lakbay.Cms`/`Lakbay.Booking` now
  that this repo is .NET too (ADR-0004).

**Exit criteria:** a GraphQL Playground/introspection query against a
locally-run `Lakbay.MockApi` returns real seeded product-line and
destination data; schema-diff CI check is green.

## Phase 2 — Storefront against the mock

**Goal:** a browsable, deployable-shaped storefront with zero live backend
dependency beyond `Lakbay.MockApi`.

**Repos:** `Lakbay.Web`.

- RTK Query API slice wired to `Lakbay.MockApi`'s GraphQL endpoint.
- Catalog/landing pages for all four product lines (Alon, Amihan, Parul,
  Pamana), rendering the seeded destination data from Phase 1.
- Redux Toolkit slices only where state is genuinely client-side (filter
  UI, basket shell — no real checkout yet, that's Phase 4).
- Jest + React Testing Library + MSW component tests; a Playwright smoke
  test that runs `Lakbay.Web` against `Lakbay.MockApi` in CI.

**Exit criteria:** the storefront is fully browsable end-to-end against
`Lakbay.MockApi` with no manual steps; component and Playwright tests
green in CI.

## Phase 3 — Real CMS

**Goal:** `Lakbay.Cms` replaces `Lakbay.MockApi` as `Lakbay.Web`'s backend
with **zero frontend code changes** — this is the proof that the shared
contract actually held.

**Repos:** `Lakbay.Cms` (primary), `Lakbay.Web` (repoint only).

- Content tree (pages, landing pages, block-list components) and Products
  tree (holiday records: itinerary, price bands, departure dates,
  inclusions, media, geo, product-line taxonomy) per ADR-0001.
- Content Delivery API enabled; GraphQL layer on top matching
  `Lakbay.Contracts` (see the open item in `01_CLAUDE.md` about which
  package to use — resolve this before this phase starts in earnest).
- Real content authored for at least the same destinations seeded in
  Phase 1, this time through the actual Umbraco backoffice.
- xUnit + FluentAssertions + NSubstitute for domain logic; Testcontainers
  (SQL Server) for integration tests against the real Content Delivery
  API.

**Exit criteria:** switching `Lakbay.Web`'s GraphQL endpoint from
`Lakbay.MockApi` to `Lakbay.Cms` requires no frontend code changes and the
storefront renders identically (content differences aside).

## Phase 4 — Booking & payments

**Goal:** an end-to-end bookable holiday, from catalog page to confirmed
order, in a staging environment.

**Repos:** `Lakbay.Booking` (primary), `Lakbay.Web` (checkout UI),
`Lakbay.Cms` (Service Bus event consumption, if availability affects
catalog display).

- CQRS command/query handlers per ADR-0002.
- Basket, availability calendar, PayMongo checkout integration (sandbox).
- Hangfire-scheduled confirmation email/SMS (Twilio) on `BookingConfirmed`.
- Service Bus wiring between `Lakbay.Booking` and `Lakbay.Cms`/`Lakbay.Web`
  per ADR-0003.
- Contract tests run against both the PayMongo sandbox and a fake
  `IPaymentGateway` implementation (Liskov Substitution check, per the
  Architecture & Patterns Guide).

**Exit criteria:** a full booking (browse → basket → PayMongo sandbox
checkout → confirmation email/SMS) succeeds in staging; contract tests
green for both real and fake payment gateway implementations.

## Phase 5 — Hosting & go-live

**Goal:** the platform is live, on real infrastructure, with real content.

**Repos:** `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web` on Azure Container
Apps; `Lakbay.MockApi` stays dev/demo-only, not deployed to production.

- Terraform or Bicep IaC (pick one — record the choice as an ADR) for
  Container Apps, Azure SQL, Redis, Service Bus, SignalR, Entra External
  ID.
- Real destination/product content replacing every placeholder from
  earlier phases.
- DNS cutover, Application Insights wired, security/QA pass (headers,
  Lighthouse, cross-browser — same discipline as Ophir Mineral Ventures'
  Phase 7).
- Legal/compliance check on DOT accreditation requirements closed or
  explicitly deferred with a named owner.
- `Docs/DEVELOPER_HANDBOOK.md` and `USER_GUIDE.md`-equivalent (for
  whoever operates the CMS day to day) finalized in `Lakbay.Cms`.

**Exit criteria:** the live site is reachable on its real domain, serves
real PH holiday content, and a real booking can be completed with a real
(not sandbox) payment method.

## Phase 6 — Scale readiness

**Goal:** revisit every "adopt later" verdict from the Blueprint's
technology-stack table against real traffic data, not assumption.

**Repos:** all, as needed.

- AKS vs. Container Apps: only move if Container Apps' per-second billing
  has genuinely stopped being cheaper than reserved AKS nodes.
- Stripe/Xendit: only add if international-card volume or SEA expansion
  justifies a second payment gateway.
- Azure AI Search: only if catalog facet complexity has outgrown Examine.
- Formal ETL (Azure Data Factory): only once real BI/reporting volume
  exists.

**Exit criteria:** each revisited item gets its own ADR recording the
decision made with real data, superseding (not silently replacing) the
original Blueprint verdict.
