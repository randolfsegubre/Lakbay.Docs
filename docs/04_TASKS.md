# Tasks — Current Status

**2026-09-08 — Phase 7 (Agent Channel) built and verified live, without Docker.**
Two new repos (`Lakbay.AgentDesktop`, `Lakbay.AgentOps`) plus real
`Lakbay.Booking` domain code now exist — see ADR-0021 through ADR-0026 and
the 2026-09-08 devlog entry for the full build and verification trail.
Highlights: a real WPF+WCF screen-pop demo running three simultaneous
processes, a real ABP+Hangfire+Redis+SignalR backend, and a real
end-to-end booking confirmation proxied through to `Lakbay.Booking`'s
atomic-decrement logic — a fresh slot confirms, an exhausted one
correctly rejects, both proven live via curl, not just by reading code.
Local Redis via a portable `redis-server.exe` (no Docker, no admin
install) and a native Oracle installer prepared but not yet run (needs
one elevated command — see `Lakbay.AgentOps/README.md`) worked around the
still-unresolved Docker blocker below entirely for this phase.

**2026-09-07 status check:** all five non-`Lakbay.Booking` repos' work
described below was confirmed real but had never been committed — now
committed locally (no push; see 05_DEVLOG.md's 2026-09-07 entry). All
repos build/test clean without Docker. **Full E2E verification, still
pending:** Docker Desktop's backend is currently crash-looping on this
machine on a stuck `sailor-ingest.sock` reparse point unrelated to the
earlier onboarding issue — needs a machine restart (the same fix that
cleared the prior Docker blocker) before Cms/AvailabilityApi's
containers can be brought up again to re-prove the sync pipe live.

**Current phase:** Phase 3 is **functionally complete and verified live**
— both trees. Products-tree sync (`Lakbay.Cms` → `Lakbay.AvailabilityApi`,
ADR-0013/0014) is proven end-to-end, `Lakbay.Cms` is the sole source of
catalog data (Phase 1's `CatalogSeeder` auto-seed retired). Content tree
(Home + 4 product-line landing pages, real copy and real photos) is live
and rendering in `Lakbay.Web` via a real block registry (ADR-0012) — see
the 2026-09-08 (7) devlog entry for the full story, including a real
Umbraco Block List attempt that didn't pan out (ADR-0015) and a real
URL-routing collision between the two trees, both found and worked around
live. The four product lines were then renamed to English (Islands/
Highlands/Festivals/Heritage — codes unchanged) and the catalog grew from
4 to 12 real Philippine destinations/products, three per line — see the
2026-09-08 (8) devlog entry. `Product` then gained accommodation +
included/optional-activities fields (ADR-0016, matching the Inghams/
Inntravel/Santa's Lapland research the user asked for) and grew to 14
products with 2 new surfing offers (Siargao, La Union) — see the
2026-09-08 (9) devlog entry. Brand decision: Lakbay stays Philippines-only
under its current name (ADR-0016's "also decided" section). The Products
tree was then restructured into a real Country → Region → Destination →
Accommodation hierarchy (ADR-0017), matching ECMS's real geography tree
shape (confirmed against the actual repo, `D:\_DEV\HPUK`, read-only
reference) adapted for Lakbay's single-country business model — see the
2026-09-08 (10) devlog entry. The Content tree then gained real nested
Region/Destination pages (ADR-0018), matching ECMS's actual page-tree
depth, and accommodation was given genuine visual prominence (a real
photo, a full section) everywhere it appears — prompted by the user
re-verifying Inghams/Inntravel's real sales models live and confirming
Lakbay's data shape (Inntravel-style, one curated holiday with one
featured stay) was already right, only the presentation needed fixing —
see the 2026-09-08 (11) devlog entry. Accommodation was then made a
first-class, independently searchable unit: real Room Types (own size/
bed-config/occupancy/price), Destination-level shared perks/add-ons, and
a new cross-destination `/stays` search flow, matching a concrete
"budget-friendly family hotel in Boracay" scenario the user described
and asked to be verified against Inghams directly (ADR-0019) — see the
2026-09-08 (12) devlog entry. Accommodation then gained a real Philippine
category (Hotel/Resort/Apartel/Pension House/Hostel/Homestay/Vacation
Rental — DOT-based, not every stay is a Hotel), Room Types gained photos
and an optional long-stay monthly rate, and a new fair-price `Activity`
marketplace (independently bookable per destination, fixed price,
itemized inclusions) shipped as the flexibility-without-a-rigid-package
answer the user asked for (ADR-0020) — see the 2026-09-08 (13) devlog
entry. Two things explicitly deferred, not
silently dropped: the GraphQL-on-Umbraco package decision (`01_CLAUDE.md`
§6) and a proper Block List backoffice-editing experience (ADR-0015).
Phase 4 (`Lakbay.Booking`) still blocked on a
PayMongo sandbox account.

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

## Done — Phase 3 sync pipeline proven live end-to-end, 2026-09-08

- [x] Sync trigger decided: Service Bus (ADR-0013), reusing the
      `Lakbay.AvailabilityApi.Sync` function ADR-0009 had scaffolded but
      left empty.
- [x] A real gap in ADR-0010 found and fixed before it could bite Phase 4:
      `Product`'s single-timestamp last-write-wins guard breaks once it has
      two independent writers (Cms owns catalog fields, Booking will own
      `availableCount`) — ADR-0014 splits the ordering guard so a stale
      Cms-only edit can never be rejected by a Booking-timed write, or vice
      versa.
- [x] `Lakbay.AvailabilityApi.Shared` extracted (Mongo access layer shared
      between the query API and the Sync function, per ADR-0009's original
      intent, not built until now).
- [x] `Lakbay.Cms`'s Products tree (ProductLine, Destination, Product)
      created code-first via `CatalogContentTypeSeeder` — **confirmed for
      real against a live SQL Server database**, all 17 property fields
      present with the correct editor per field, including a genuine bug
      found and fixed live (a fresh Umbraco 18.1.1 install doesn't
      pre-create a data type for every built-in editor — `Umbraco.Decimal`
      had none — fixed with a create-on-demand fallback).
- [x] Real content authored and published programmatically (no backoffice
      login credentials exist in any session, deliberately — see
      2026-09-08 (2) below) via the same seeder, exercising a genuine
      `ContentPublishedNotification` per node — proves the pipe works the
      same way real backoffice authoring would.
- [x] **Full pipe verified live, not just "it compiles":** Cms publish →
      Service Bus (`lakbay-catalog-sync` queue, real emulator) →
      `Lakbay.AvailabilityApi.Sync` (12 messages consumed, 0 failures) →
      MongoDB → `Lakbay.AvailabilityApi`'s GraphQL query returns the
      Cms-authored data, correctly resolved (ProductLineCode via Content
      Picker → picked node's `code`, not the raw picker UDI — a real bug
      found and fixed live).
- [x] Real photos added to all four seeded products (Wikimedia Commons,
      verified resolvable) and rendered in `Lakbay.Web` — collection cards
      and the holiday detail hero — confirmed in a real browser across all
      four collections.
- [x] **Phase 1 vs. Phase 3 data duplication resolved** — Randolf decided:
      `Lakbay.Cms` is now the sole source of catalog data.
      `Lakbay.AvailabilityApi.Api`'s `CatalogSeeder` no longer runs
      automatically on boot (`Program.cs`); it's kept only as a test-support
      utility, called explicitly by `Lakbay.AvailabilityApi.Tests`. The 4
      stale hand-seeded documents in the real local MongoDB were deleted;
      confirmed via GraphQL and a real browser that exactly 4 products
      (the Cms-authored ones) now show, no duplicates.
- [x] **A real, previously-invisible test bug found and fixed as a side
      effect**: `Lakbay.AvailabilityApi.Api`'s `Program.cs` read Mongo
      configuration *eagerly* (`builder.Configuration.Get<MongoDbSettings>()`
      before `builder.Build()`), which meant `WebApplicationFactory`'s
      per-test database-name override (used by
      `Lakbay.AvailabilityApi.Tests`) never actually applied — every test
      had silently been running against the real local MongoDB the whole
      time, not an isolated Testcontainers database. Never caught before
      because normal local dev has no such override to diverge from. Fixed
      by binding lazily inside each DI factory delegate instead. Fixing it
      exposed a second, also-real bug it had been masking: three test
      method names, once prefixed, exceeded MongoDB's 63-character database
      name limit — fixed with a truncate-plus-hash helper. All 8 tests now
      pass against genuinely isolated data.

## Done — Content tree live end-to-end, 2026-09-08

- [x] `Lakbay.Cms`: Home Page + one landing page per product line
      (`ContentTreeSeeder`), real copy and real photos (Wikimedia Commons,
      each verified resolvable), seeded and published the same
      programmatic-authoring way the Products tree is.
- [x] A real Umbraco Block List was attempted first (element types,
      `DataType.ConfigurationData` hand-built, stored property value
      hand-built to match `BlockListValue`'s shape) — the stored JSON was
      structurally sound (confirmed via raw SQL) but Umbraco's own
      Content Delivery API silently returned zero items reading it back,
      root cause not found within budget. Pivoted to the same proven
      JSON-in-Textarea pattern already used for `Product.priceBands`,
      documented as ADR-0015 — ADR-0012's actual intent (a `type` string
      mapping to a React component) is unaffected, only the storage/read
      mechanism changed.
- [x] `Lakbay.Cms`'s Content Delivery API enabled (`AddDeliveryApi()` was
      missing from `Program.cs` — config alone doesn't compose its DI
      services, a real error found live) and given the same CORS
      allow-list pattern `Lakbay.AvailabilityApi` already uses, so
      `Lakbay.Web` can call it cross-origin.
- [x] `Lakbay.Web`: a real block registry (`BlockRegistry.tsx`, ADR-0012)
      mapping `"hero"`/`"imageText"` to components; a new RTK Query slice
      (`cmsContentApi`) alongside the existing `availabilityApi` — two
      backends, each queried the way ADR-0007 always intended. Homepage
      and all four `/collections/[code]` pages now render real Cms hero
      images/copy above the existing AvailabilityApi-driven product grid,
      with a graceful static fallback if Cms isn't reachable.
- [x] **A real URL-routing collision found and fixed live:** the Products
      tree's `ProductLine` nodes ("Alon", "Amihan"...) and the Content
      tree's landing pages share the same names, so Umbraco resolved
      both to the same `/alon` route — and picked the wrong one.
      Sidestepped by fetching Home's children and matching by
      `productLineCode` client-side rather than a direct route lookup,
      instead of restructuring either tree's URLs under time pressure;
      a real container-node-per-tree fix is still open if the Content
      tree's IA gets revisited.
- [x] Verified in a real browser, not just via the Delivery API directly:
      homepage hero + "why Lakbay" section, and all four collection
      pages' heroes + "on the ground" sections, all rendering real Cms
      copy and real photos above the existing (still-working) product
      grids.

## Done — English rename + expanded destination catalog, 2026-09-08 (8)

- [x] The four product lines renamed to English display names for a
      foreign-plus-domestic audience: Alon → Islands, Amihan → Highlands,
      Parul → Festivals, Pamana → Heritage. GraphQL enum codes unchanged
      (`ALON`/`AMIHAN`/`PARUL`/`PAMANA` still the stable identifier) —
      "Lakbay" itself was explicitly kept as-is (platform brand, not a
      product line).
- [x] `Lakbay.Web`: found and fixed four spots deriving nav/heading text
      directly from the enum code string (`code[0] + code.slice(1)...`)
      instead of reading it from anywhere — a Cms-side rename alone would
      not have fixed the nav bar. Added `PRODUCT_LINE_META[code].label`
      as the single source of truth (`lib/catalog.ts`), used in
      `SiteHeader.tsx`, `collections/[code]/page.tsx` (×2), and
      `holidays/[slug]/page.tsx`.
- [x] `Lakbay.Cms`: `CatalogContentTypeSeeder`'s ProductLine `Name` values
      updated to English. `ContentTreeSeeder`'s landing-page `Name` values
      changed to `"{English name} Landing Page"` rather than matching the
      ProductLine name exactly — reusing the same plain name on both
      trees would have recreated the exact URL-collision bug the
      2026-09-08 (7) session had just worked around.
- [x] Catalog expanded from 1 to 3 destinations/products per line (12
      total, up from 4) — real, well-known Philippine destinations, each
      with a verified Wikimedia Commons photo: Islands adds Boracay and
      El Nido; Highlands adds Sagada and Tagaytay; Festivals adds Cebu
      (Sinulog) and Iloilo (Dinagyang); Heritage adds Banaue and
      Intramuros.
- [x] **A real gap found in the reseed flow, not just this session's
      content:** `LAKBAY_FORCE_RESEED_CATALOG=true` deletes and recreates
      Cms content nodes, which get new Umbraco UDIs — since
      `CatalogSyncFunction` upserts Mongo documents keyed by that UDI, the
      old Mongo documents from the previous run were never cleaned up,
      producing duplicate products/destinations under the old and new
      IDs. Not a data-loss risk (last-write-wins still applies per
      product), but a real duplicate-listing bug. Fixed for this run with
      a targeted `deleteMany` on the stale (pre-reseed timestamp)
      documents — the same scoped-delete approach used for the earlier
      Phase 1 cleanup, never a full collection drop. The underlying gap
      (reseed doesn't garbage-collect orphaned synced documents) is not
      fixed at the code level — noted here as a known rough edge in the
      `LAKBAY_FORCE_RESEED_CATALOG` dev-only escape hatch, not something
      any real editor flow can hit (a backoffice edit never changes a
      node's UDI).
- [x] Verified live in a real browser: nav bar, homepage collection
      cards, and all four `/collections/[code]` pages (Islands showing
      Coron/Boracay/El Nido; Heritage showing Vigan/Banaue), plus a
      product detail page's "← Back to Islands" link.

## Done — accommodation + activities on Product, 2 new surf offers, 2026-09-08 (9)

- [x] Researched Inghams, Inntravel, and Santa's Lapland (user's own
      pointer) to ground the "collections just show the place" complaint
      in a concrete pattern: the accommodation is a first-class part of
      the product, and activities split into included-in-the-price vs.
      optional add-ons. Full writeup in ADR-0016.
- [x] `Product` gained `accommodationName`, `accommodationDescription`,
      `includedActivities`, `optionalActivities` — across
      `Lakbay.Contracts` (SDL + C# record + regenerated TS types),
      `Lakbay.Cms` (new Product properties, seed content, publish-sync
      mapping), `Lakbay.AvailabilityApi.Sync` (field-scoped `$set`), and
      `Lakbay.Web` (GraphQL fragment + holiday detail page + collection
      cards' "Stay: X" line).
- [x] All 14 products (the existing 12 plus 2 new) authored with a real,
      plausible (not real-brand) accommodation name/description and
      destination-specific included/optional activity lists — not filler.
- [x] **Two new surfing products added** (the user's explicit ask),
      under Islands (ALON): Siargao Surf Camp (Cloud 9, the country's
      surf capital) and La Union Surf Weekend (San Juan, the accessible
      beginner surf town three hours from Manila) — real destinations,
      each with a verified Wikimedia Commons image.
- [x] **Three real gaps found during rollout, not anticipated going in**
      (full detail in ADR-0016): (1) adding a property to an *existing*
      Umbraco content type isn't handled by the seeder's create-once
      guard — fixed with a one-time migration matching ADR-0015's own
      pattern; (2) the Sync Azure Function host had been running since
      before this session's code changes and needed an explicit restart
      — every long-running local process this platform depends on needs
      restarting after a schema change that touches it, not just the
      ones restarted most often; (3) each crash-then-fix cycle left a
      generation of orphaned Mongo documents (same class of issue as the
      2026-09-08 (8) entry) — cleaned up the same way, a targeted
      `deleteMany` on the pre-cutoff generation, confirmed via a
      `$dateTrunc`-grouped count first.
- [x] Brand/scope decision made explicit (user asked, not assumed):
      Lakbay stays Philippines-only under its current name — "Lakbay"
      means "journey," not literally "Philippines," so the name itself
      doesn't block a future non-PH destination if that's ever decided.
      Documented in ADR-0016 rather than left as an unrecorded verbal
      call.
- [x] Verified live in the browser: a surf product's detail page ("Where
      you'll stay" card, "What's included" / "Optional extras" two-column
      list), and the Islands collection page showing all 5 products
      (Coron, Boracay, El Nido, Siargao, La Union) each with a "Stay: X"
      line on its card.
- [x] `dotnet test` re-run after the `Lakbay.Contracts`/`CatalogSeeder`
      changes — all 8 tests still green.

## Done — Country/Region/Accommodation geography hierarchy (ADR-0017), 2026-09-08 (10)

- [x] Grounded the redesign in the real ECMS structure before building
      anything — an Explore agent read `D:\_DEV\HPUK\Hotelplan.Inghams.V2.CMS`
      (read-only reference, never pushed to) and confirmed the real tree
      (`Product → Geography → Country → Region → Resort → Accommodation`,
      duplicated per product line) and — the key finding — that ECMS's
      actual "highlights" content lives in a separate external PCMS
      system, not Umbraco fields, which Lakbay's own ADR-0001 already
      rejected. Adapted accordingly: same tree shape, highlights as real
      Umbraco fields throughout.
- [x] Used `EnterPlanMode`/`ExitPlanMode` for this one — cross-repo schema
      redesign with real judgment calls (shared vs. per-line Country;
      how far to take the Web-side scope) — and confirmed both via
      `AskUserQuestion` before writing any code.
- [x] New `Country`, `Region`, `Accommodation` types across
      `Lakbay.Contracts` (SDL + C# records + regenerated TS types),
      `Lakbay.Cms` (3 new content types, `Destination.region` and
      `Product.accommodation` converted from flat text to Content
      Pickers, full seed-data rewrite), `Lakbay.AvailabilityApi` (3 new
      Mongo collections/class maps, 3 new `Apply*Async` sync handlers,
      `country`/`regions` GraphQL queries), and `Lakbay.Web` (nested
      GraphQL fragments, a Country → Region → Destination breadcrumb with
      inline highlights on the holiday detail page, accommodation card
      now backed by real content).
- [x] Fixed a real, pre-existing data inconsistency while rewriting the
      seed data: the original 14 destinations mixed province-level region
      names ("Palawan", "Ilocos Sur") with proper regional groupings
      ("Western Visayas", "Cordillera Administrative Region") — the new
      Region entity forces one consistent granularity, applied uniformly
      (12 real Region nodes across the 14 destinations).
- [x] Same migration discipline as ADR-0015/0016: a self-limiting
      stale-shape check (Region content type missing) triggers a full
      delete-and-recreate of `Product`/`Destination`/`ProductLine` — no
      in-place data transform needed for disposable pre-launch dev data.
- [x] All three long-running processes stopped and restarted in the
      right order before reseeding (Cms, `Lakbay.AvailabilityApi.Api`,
      **and** the Sync Azure Function host) — the lesson from ADR-0016's
      rollout applied proactively this time instead of rediscovered.
- [x] `dotnet build` clean across all three .NET repos, `dotnet test`
      still 8/8 green, `npx tsc --noEmit` clean, then verified live:
      GraphQL returning the full nested chain (`product.accommodation`,
      `product.destination.region.country`), MongoDB collection counts
      matching expectations after cleaning up one round of orphaned
      documents from the recreate cycle (same targeted-`deleteMany`
      discipline as every prior reseed this session), and a real browser
      pass showing the breadcrumb, region/country highlights, and
      accommodation card on a holiday detail page.
- [x] Scope deliberately held to enriching the existing detail page —
      dedicated `/regions/[id]`/`/countries/[code]` browsing pages are an
      explicit next step (confirmed with the user), not built now. The
      `regions`/`country` GraphQL queries already exist for whenever that
      page gets built.

## Done — nested Content tree + accommodation prominence (ADR-0018), 2026-09-08 (11)

- [x] Real nested Region/Destination pages in the Content tree (26 new
      nodes: 12 Region landing pages, 14 Destination landing pages),
      matching ECMS's actual `Home → ProductLine → Region → Resort →
      Accommodation` page-tree depth — genuine Umbraco parent-child
      nesting this time, deliberately different from the Products tree's
      flat + Content-Picker shape (ADR-0017), matching how ECMS and PCMS
      really differ.
- [x] Used `EnterPlanMode` a second time this session — cross-repo,
      multi-file build with real design decisions (how Content-tree pages
      reference varying-parent Region/Destination data without a
      hardcodable parent ID) warranted it again.
- [x] Mid-implementation, the user asked to re-verify Inghams/Inntravel's
      real sales model — re-browsed both live. Inghams: accommodation
      genuinely is the primary sellable unit (every search result is a
      named, individually-priced hotel). Inntravel: closer to Lakbay's own
      shape — one curated holiday, one accommodation, given a full
      dedicated page section. Confirmed Lakbay's existing data model
      (Product owns one Accommodation, ADR-0017) already matches
      Inntravel's real pattern — the actual gap was presentation, not
      structure.
- [x] Accommodation given genuine visual prominence: a real photo (reusing
      the destination's own already-sourced Wikimedia image — never a
      fabricated photo of the invented property name) plus a full,
      unmissable section on both the new Destination page and the
      existing holiday detail page.
- [x] `slug` added to `Region`/`Destination` (`Lakbay.Contracts`,
      mirroring `Product.slug`) as the stable join key between the
      Products tree's flat data and the Content tree's nested pages.
- [x] `collections/[code]/page.tsx` gained an "Explore by region" grid,
      additive to the existing flat product listing.
- [x] Same migration/verification discipline as every prior ADR this
      session: `dotnet build`/`test` clean (8/8), `tsc --noEmit` clean,
      all three long-running processes stopped and restarted in order
      before reseeding, orphaned Mongo generation cleaned up with a
      targeted `deleteMany` (confirmed via `$dateTrunc` grouping first),
      and a full real-browser pass through Islands → Palawan → Coron
      confirming the nested pages, real highlights, and the new
      accommodation section all render correctly.

## Done — Room Types, destination perks, Stays search (ADR-0019), 2026-09-08 (12)

- [x] Verified the real Inghams "Room Types" pattern live (Hotel Post, St
      Anton) before building: per-room size/bed-configuration/occupancy/
      price, resort-wide included perks and optional add-ons shown on the
      hotel page but identical across that resort's hotels.
- [x] Two scope questions confirmed with the user via `AskUserQuestion`
      before writing code: perks move to `Destination` (Product keeps its
      own curated activities separately), and a new cross-destination
      Stays page gets built (not just a per-destination upgrade).
- [x] New `RoomType` entity across all four repos — synced as its own
      top-level entity with a plain `accommodationId` reference (not
      embedded), matching the existing `ProductFilter.destinationId`
      convention. 28 real room types seeded, 2 per Accommodation.
- [x] `Accommodation` gained `destination` (previously only reachable via
      Product), `tags`, `officialRating`; `Destination` gained
      `includedPerks`/`optionalAddOns`.
- [x] New `Lakbay.Web` routes: `/stays` (destination + tag filters, client-
      side over the full list) and `/stays/[accommodationId]` (full detail:
      Room Types, Destination perks/add-ons, linked Product package).
      Existing Destination page updated to show Room Types too and
      retitled "Holidays here" → "Or book as a ready-made package."
- [x] **A real pre-existing SDL/runtime schema gap found live in the
      browser**: HotChocolate serves a plain `string?` parameter as
      GraphQL `String`, not `ID` (only a member literally named `Id` gets
      `ID` by convention) — the authored SDL had said `ID` for the new
      `destinationId`/`accommodationId` arguments, which broke a
      raw-variable client query outright. Fixed by matching SDL and
      client to the real runtime type. `ProductFilter.destinationId` has
      the identical latent mismatch and predates this session — left
      alone (never triggered, out of scope), flagged for a future look.
- [x] Same discipline as every prior ADR this session: stale-shape
      migration triggered a real delete-and-recreate; all three
      long-running processes restarted in order before reseeding;
      `dotnet build`/`test` clean (8/8), `tsc --noEmit` clean; one round
      of orphaned pre-reseed Mongo documents cleaned up via targeted
      `deleteMany` (confirmed via `$dateTrunc` grouping first); a full
      real-browser pass through the exact Boracay scenario plus
      Destination-page and holiday-page regression checks.

## Done — ProductFilter.destinationId SDL/runtime mismatch closed out, 2026-09-08 (13)

- [x] `schema/lakbay.graphql`'s `ProductFilter.destinationId` changed `ID` →
      `String`, matching what `Lakbay.AvailabilityApi` actually serves (the
      same direction chosen for `accommodations`/`roomTypes` in ADR-0019,
      over annotating the C# resolver to make the server serve `ID`
      instead). No C# change needed — `ProductFilter.DestinationId` was
      already a plain `string?`.
- [x] `Lakbay.Contracts` TypeScript types regenerated (`npm run codegen`).
- [ ] Live GraphQL introspection against a running `Lakbay.AvailabilityApi.Api`
      not yet done — blocked by an unrelated, pre-existing build break
      (`CatalogSeeder.cs` not yet updated for `Accommodation.Type`, likely
      ADR-0020 in-progress work). Verified statically instead (no `[ID]`
      override anywhere on this field); worth a real introspection pass
      once that build break is fixed.

## Done — Accommodation types, room photos, long-stay pricing, Activities marketplace (ADR-0020), 2026-09-08 (13)

- [x] Researched real Philippine accommodation categories (DOT-accredited
      terms plus live Airbnb listings in these exact destinations),
      real long-stay/digital-nomad pricing patterns, and the real Klook/
      GetYourGuide fair-price anti-scam model before designing anything.
- [x] Three scope questions confirmed with the user via `AskUserQuestion`:
      the DOT-based `AccommodationType` category list; long-stay modeled
      as a monthly rate on individual Room Types (not an Accommodation-
      level flag); a full new top-level `Activity` entity across all 14
      destinations (not a smaller upgrade to `optionalAddOns`).
- [x] `RoomType` gained `heroImageUrl` (reuses the owning Accommodation's
      own real photo — never a fabricated distinct-room interior) and
      `monthlyRatePhp` (nullable, seeded only in 5 realistic long-stay
      destinations: Baguio, Cebu, Siargao, La Union, Boracay).
- [x] `Accommodation` gained `type: AccommodationType!` — all 14 existing
      accommodations reassigned a real, description-grounded category
      across all 7 enum values (not left defaulting to Hotel).
- [x] New `Activity` entity across all four repos — 14 real activities
      seeded (one per destination), each with a realistic fixed PHP
      price and itemized inclusions grounded in the Klook/GetYourGuide
      research.
- [x] New `Lakbay.Web` `/activities` route plus "Activities near your
      stay" / "Book activities here" teasers on the Accommodation detail
      and Destination pages — the actual flexibility-without-a-package
      answer: pick a room (Stays), add fixed-price activities on your
      own terms, `Product` still available as a secondary option.
- [x] Avoided repeating ADR-0019's HotChocolate `String`-vs-`ID` schema
      trap on the new `activities(destinationId: String)` field —
      declared `String` from the start this time.
- [x] **A real orphaned-generation bug found, distinct from every prior
      one**: the established `$dateTrunc`-at-hour cleanup check silently
      merged two genuinely different reseed generations into one bucket
      because both ran within the same clock hour, reporting "no
      orphans" when 28 accommodation documents (14 stale, defaulting to
      `HOTEL`) actually existed. Fixed by re-running the grouping at
      minute granularity — noted as the safer default going forward.
- [x] Same discipline as every prior ADR this session: `dotnet build`/
      `test` clean (8/8), `tsc --noEmit` clean, all three long-running
      processes restarted in order before reseeding, and a full
      real-browser pass confirming room photos, the long-stay callout,
      the new accommodation-type filter, and the Activities marketplace
      all render and link correctly.

## Not done — rest of Phase 0/1

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
  §6) — blocks locking `Lakbay.Contracts` schema v0 as final. Still open:
  `Lakbay.Web` reaches the Content tree via Umbraco's built-in REST
  Content Delivery API (2026-09-08), not a GraphQL layer — a real,
  working interim path, but this decision is about the GraphQL layer
  specifically and hasn't been resolved by that.
- Package registry choice (GitHub Packages vs. Azure Artifacts) for
  publishing `Lakbay.Contracts` — not urgent; all five repos currently
  reference it via local project/file references, which works fine for
  single-machine local dev.
- Production MongoDB hosting for `Lakbay.AvailabilityApi` (Azure Cosmos DB
  for MongoDB vs. MongoDB Atlas) — a Phase 5 decision, not urgent now.
