# Tasks — Current Status

**Current phase:** Phase 0 complete everywhere. Phase 1 done
(`Lakbay.AvailabilityApi`'s query API). Phase 2 done (`Lakbay.Web`'s real
catalog storefront, verified end-to-end in a browser against live seeded
PH data). Next up: Phase 3 (`Lakbay.Cms`'s content + product trees) or
Phase 4 (`Lakbay.Booking` — real checkout, replacing the honestly-disabled
"Book this holiday" button) — see `02_BUILD_PLAN.md`.

## Done

- [x] Market/product research completed; four product lines defined (Alon,
      Amihan, Parul, Pamana) — 2026-08-29
- [x] Full architecture plan drafted and published as the Lakbay Blueprint
      artifact — 2026-08-29, revised 2026-09-01
- [x] Six repos created under `Personal_Projects/Lakbay/` and git-initialized
      (`Lakbay.Docs`, `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web`,
      `Lakbay.MockApi`, `Lakbay.Contracts`) — 2026-09-03
- [x] `Lakbay.Docs` populated: `01_CLAUDE.md`, this file, `02_BUILD_PLAN.md`,
      `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`, ADR-0001–0003 — 2026-09-03
- [x] Each of the five application repos given a thin `CLAUDE.md`, README,
      and stack-appropriate `.gitignore` — 2026-09-03
- [x] `Lakbay.MockApi` switched from Node.js/Apollo to .NET/HotChocolate/
      MongoDB.Driver (ADR-0004); local-dev database story clarified — SQL
      Server in Docker, Azure SQL Database only once live (ADR-0005) —
      2026-09-05
- [x] Headless-CMS decision formalized as ADR-0006 (no Razor/UI code in
      `Lakbay.Cms`, ever) — 2026-09-05
- [x] `Lakbay.MockApi` renamed to `Lakbay.SearchApi` and reframed as a
      real, permanently deployed search service (not a disposable mock),
      modeled on Hotelplan's `api-sphinx`/Manticore (ADR-0007); Build Plan
      Phases 1/3/5 rewritten accordingly; `06_SYSTEM_ARCHITECTURE.md`
      written covering per-repo and whole-platform architecture —
      2026-09-06
- [x] Real-time availability propagation designed (Service Bus →
      `Lakbay.SearchApi` → Azure SignalR Service, no polling) — ADR-0008,
      Phase 4 rewritten to include it, explicitly separated from the
      overselling/concurrency-control concern it does not solve —
      2026-09-06
- [x] `Lakbay.SearchApi` renamed to `Lakbay.AvailabilityApi` and split
      into a pure query-serving GraphQL service plus a separate
      Service-Bus-triggered Azure Function (`Lakbay.AvailabilityApi.Sync`)
      for event consumption — decouples sync/write load from query
      performance (ADR-0009) — 2026-09-06
- [x] Last-write-wins ordering guard designed for the sync function, using
      a source-generated timestamp so out-of-order Service Bus delivery
      can't overwrite newer data with stale data (ADR-0010) — 2026-09-06
- [x] Double-booking prevention designed: an atomic, conditional SQL
      `UPDATE` in `Lakbay.Booking`'s confirm-booking handler, not a
      read-then-write check — the actual fix for two near-simultaneous
      bookings racing for the same slot, explicitly independent of the
      real-time propagation work above (ADR-0011) — 2026-09-06
- [x] Block-rendering pattern designed: Umbraco Block List/Grid JSON →
      a React block-registry in `Lakbay.Web` mapping element-type alias to
      component, matching the standard headless-CMS component-mapping
      pattern (ADR-0012) — 2026-09-06
- [x] **Implementation started — Phase 0 scaffolding, all five repos,
      2026-09-06:**
  - `Lakbay.Contracts`: `schema/lakbay.graphql` (v0 — Product,
    ProductLine, Destination, incl. `sourceUpdatedUtc` per ADR-0010); a
    net10.0 C# class library mirroring it by hand; a `@lakbay/contracts`
    npm package generating TS types via `@graphql-codegen` — both sides
    build/type-check clean.
  - `Lakbay.Booking`: minimal API + xUnit, `/health` endpoint, boots and
    tests green.
  - `Lakbay.AvailabilityApi`: query API (HotChocolate) + a separate
    `Lakbay.AvailabilityApi.Sync` Azure Function project (ADR-0009), both
    build; query API boots, introspection/query test green.
  - `Lakbay.Web`: Next.js 16 + Redux Toolkit + RTK Query, builds and
    lints clean.
  - `Lakbay.Cms`: Umbraco **18.1.1** (not 17 — corrected; see below)
    scaffolded, confirmed booting to the real install wizard via browser
    screenshot.
  - **Verified end-to-end, not just per-repo:** ran `Lakbay.Web` and
    `Lakbay.AvailabilityApi` simultaneously; the homepage's RTK Query call
    round-tripped through a real GraphQL request and rendered the live
    response in the browser (CORS configured on the API side to make this
    work).
  - All five repos' `Docs/DEVELOPER_HANDBOOK.md` written from what was
    actually proven working, not speculatively.
- [x] **Version correction:** every prior document said "Umbraco 17."
      Checking `dotnet new install Umbraco.Templates` on 2026-09-06 showed
      the real latest is **18.1.1** — corrected across all living docs
      (historical ADR-0001 text left as-is, matching the project's own
      "don't rewrite point-in-time records" convention).
- [x] SQL Server (Docker, Developer Edition) Compose file written for
      `Lakbay.Cms` (adapted from the official `dotnet new umbraco-compose`
      template, trimmed to database-only per ADR-0005 — the app runs
      natively, never in the compose file); `Database/setup.sql` creates
      both `umbracoDb` and `lakbayBookingDb` on one shared instance — not
      yet run, Docker Desktop install was still in progress at end of
      session.

## Done — Docker unblocked, 2026-09-08

- [x] Docker Desktop finished installing after the machine restart;
      confirmed running (`docker ps` reachable, daemon up).
- [x] `docker compose up -d` from `Lakbay.Cms` — SQL Server container
      built and healthy (`lakbay_sqlserver`, both `umbracoDb` and
      `lakbayBookingDb` created per `Database/setup.sql`).
- [x] Connection string wired via `dotnet user-secrets`; `Lakbay.Cms.Web`
      boots against the real database — backoffice module bundle loads
      clean, no exceptions, listening on both configured ports.

## Done — Lakbay.Cms Phase 0 fully closed out, 2026-09-08

- [x] Umbraco install wizard's admin-account step completed by Randolf
      through the browser. Credential itself deliberately not recorded
      anywhere in this repo or memory — even for a local-only environment,
      nothing gitignored-adjacent should carry a real password, since a
      repo's local-only status can change later and memory syncs across
      machines. `Lakbay.Cms` now has a working backoffice login.
- [x] Consolidated `07_MANUAL_SETUP_GUIDE.md` written — one linear,
      dependency-ordered walkthrough across all five application repos,
      pulling proven commands from each repo's own
      `Docs/DEVELOPER_HANDBOOK.md` rather than restating from memory.

## Done — Phase 1 (Lakbay.AvailabilityApi query API), 2026-09-08

- [x] Real `productLines`/`destinations`/`products`/`product` resolvers
      against MongoDB, matching schema/lakbay.graphql field-for-field —
      no `[UseFiltering]`, hand-built `FilterDefinition<T>` from the
      explicit `ProductFilter` input so the field signature stays in sync
      with `Lakbay.Contracts`.
- [x] MongoDB via Docker Compose (`lakbay_mongo`, no auth — local-only,
      nothing secret to protect, unlike SQL Server).
- [x] `CatalogSeeder` — real Philippine destinations/products (Coron/Alon,
      Baguio/Amihan, San Fernando Pampanga/Parul, Vigan/Pamana), not
      placeholder data, seeded idempotently on startup.
- [x] 7 xUnit tests via Testcontainers.MongoDb — a real ephemeral
      database per test run, not a mock; each test isolated to its own
      database name.
- [x] Verified live via `curl`, not just via the test suite — including a
      real bug caught and fixed (`ProductLine`'s missing `Id` member
      needed `IgnoreExtraElements` on its Mongo class map; see that
      repo's `Docs/DEVELOPER_HANDBOOK.md` for the full explanation).
- [x] `07_MANUAL_SETUP_GUIDE.md`, `Lakbay.AvailabilityApi`'s own
      handbook, and its README updated to match — including a real,
      proven "adding a new query field" walkthrough (previously deferred
      as "not applicable yet").

## Done — Phase 2 (Lakbay.Web catalog storefront), 2026-09-08

- [x] Real pages: `/` (all four product lines), `/collections/[code]`
      (products in a line), `/holidays/[slug]` (full detail) — all
      querying live `Lakbay.AvailabilityApi` data, no placeholders.
- [x] Typed RTK Query endpoints (`getProductLines`, `getDestinations`,
      `getProducts`, `getProduct`) hand-written against
      `Lakbay.Contracts`' generated types, `transformResponse` throwing on
      GraphQL `errors` rather than silently returning `undefined`.
- [x] Verified in an actual browser (screenshots across three of the four
      collections), not just a clean `npm run build`.
- [x] **A real Turbopack + local-workspace-package gotcha, found and
      fixed**: `@lakbay/contracts` (a `file:` dependency shipping raw
      `.ts`) wouldn't resolve under Next 16's Turbopack even with the
      standard `transpilePackages` fix, because Turbopack infers
      `Lakbay.Web`'s own `package-lock.json` as the workspace root and
      refuses files outside it. Fixed with a `tsconfig.json` `paths`
      alias *plus* a widened `turbopack.root` — neither alone was enough.
- [x] **A real HotChocolate schema-naming bug, found only by a proper
      GraphQL client, not `curl`**: `ProductFilter` was being served as
      `ProductFilterInput` (HotChocolate's default naming convention),
      invisible to every existing `curl` test because they all passed
      `filter` as an inline literal rather than a `$filter: ProductFilter`
      variable — exactly what `Lakbay.Web` does. Fixed with an explicit
      `ProductFilterInputType` descriptor; a regression test now covers
      the real-variable path specifically.
- [x] Booking is an honestly-disabled button ("coming in Phase 4") — not
      a fake/stubbed checkout flow.

## Not done — rest of Phase 0/1

- [ ] Decide the Cms → AvailabilityApi sync trigger mechanism (Umbraco
      event, Service Bus message, or scheduled job) — flagged by ADR-0007,
      genuinely Phase 1/3 work.
- [ ] CI skeleton in every repo — not started.
- [ ] A concurrency test in `Lakbay.Booking` proving the atomic decrement
      actually prevents double-booking under simulated simultaneous
      requests — ADR-0011; needs the real SQL Server connection to be
      meaningful (SQLite/InMemory wouldn't exercise real row-locking
      semantics), so this is genuinely Phase 4 work, not deferred Phase 0.
- [ ] Decide: GitHub remotes for these repos, or stay local-only for now —
      open item, needs an explicit answer, not an assumption.

## Blocked / needs a decision before it can proceed

- Exact GraphQL-on-Umbraco package for `Lakbay.Cms` (see `01_CLAUDE.md`
  §6) — blocks locking `Lakbay.Contracts` schema v0 as final.
- Package registry choice (GitHub Packages vs. Azure Artifacts) for
  publishing `Lakbay.Contracts` — not urgent; all five repos currently
  reference it via local project/file references, which works fine for
  single-machine local dev.
- Production MongoDB hosting for `Lakbay.AvailabilityApi` (Azure Cosmos DB
  for MongoDB vs. MongoDB Atlas) — a Phase 5 decision, not urgent now.
