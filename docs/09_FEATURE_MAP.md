# Feature Map — "where do I look?"

A fast-lookup index from a user-facing feature or a bug report straight to
the repo, file, and class/function responsible. Written for two audiences
equally: a developer triaging a bug report, and an AI coding agent that's
been asked to fix or extend one feature without reading six repos cold.

**How to use this file:** find the row closest to what you're
investigating, go straight to the file(s) listed, skim the "Key
class/function" column for where the actual logic lives. Each row also
names the ADR that explains *why* it's built this way, if the "why" isn't
obvious from the code alone.

**Keep this current:** when you add or move a feature, update its row (or
add a new one) in the same commit — this file rots fast if it's treated as
a one-time snapshot instead of a living index.

---

## Storefront (customer-facing)

| Feature | Repo | Key file(s) | Key class/function | Notes / ADR |
|---|---|---|---|---|
| Homepage — four product-line cards | `Lakbay.Web` | `src/app/page.tsx` | `Home()` | Queries `useGetProductLinesQuery` — zero hard-coded lines, driven entirely by API response |
| Collection page — products in one line | `Lakbay.Web` | `src/app/collections/[code]/page.tsx` | `CollectionPage()` | Filters via `useGetProductsQuery({ productLine: code })` |
| Holiday detail page | `Lakbay.Web` | `src/app/holidays/[slug]/page.tsx` | `HolidayPage()` | Price bands table, board basis, availability, hero image, accommodation card, included/optional activities (ADR-0016) |
| Site header / nav | `Lakbay.Web` | `src/app/SiteHeader.tsx` | — | Static list of the four product lines for nav links |
| Product-line visual theme (emoji, accent color) | `Lakbay.Web` | `src/lib/catalog.ts` | `PRODUCT_LINE_META` | Presentation-only metadata keyed by `ProductLineCode` — not from the API on purpose (schema doesn't carry imagery/color) |
| Formatting helpers (PHP currency, dates, board-basis labels, destination display name) | `Lakbay.Web` | `src/lib/catalog.ts` | `formatPhp`, `formatDate`, `boardBasisLabel`, `destinationLocation` | `destinationLocation` specifically avoids "Coron, Palawan, Palawan" — a real bug fixed 2026-09-08 |
| GraphQL client / RTK Query API slice | `Lakbay.Web` | `src/lib/availabilityApi.ts` | `availabilityApi` (RTK Query `createApi`) | Every field selected in each query's GQL string — check here first if a page is missing a field the schema has |
| Hero images (Wikimedia URLs) | `Lakbay.Web` | `next.config.ts` (`images.remotePatterns`), `src/app/collections/[code]/page.tsx`, `src/app/holidays/[slug]/page.tsx` | — | New external image host? Add it to `remotePatterns` first or `next/image` silently 400s |

## Catalog search/query API

| Feature | Repo | Key file(s) | Key class/function | Notes / ADR |
|---|---|---|---|---|
| GraphQL schema entry point | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Api/Program.cs` | `AddGraphQLServer().AddQueryType<Query>()` | |
| All four resolvers (productLines, destinations, products, product) | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Api/Query.cs` | `Query` class | Hand-built Mongo filters, not `[UseFiltering]` — ADR-0007 explains why (schema-shape parity with `Lakbay.Contracts`) |
| `ProductFilter` GraphQL type naming | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Api/ProductFilterInputType.cs` | `ProductFilterInputType` | Fixes a real HotChocolate default-naming bug (`ProductFilterInput` vs `ProductFilter`) — see its own doc comment before touching |
| Mongo collections / access layer | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Shared/CatalogContext.cs` | `CatalogContext` | Shared between the query API and the Sync function (ADR-0009) — one copy, not two |
| Mongo ↔ C# type mapping | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Shared/MongoClassMaps.cs` | `MongoClassMaps.Register()` | `ProductLine`/`Product` both need `SetIgnoreExtraElements(true)` — see ADR-0014 for why on `Product` specifically |
| Test-only seed data (real PH destinations/products) | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Api/CatalogSeeder.cs` | `CatalogSeeder.SeedIfEmptyAsync` | **Not called automatically anymore** (retired 2026-09-08) — only `Lakbay.AvailabilityApi.Tests` calls it now, via `AvailabilityApiFactory.SeedCatalogAsync()` |
| CORS (so `Lakbay.Web` can call this cross-origin) | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Api/Program.cs`, `appsettings.Development.json` | `Cors:AllowedOrigins` config key | A missing/wrong origin here is the #1 cause of "works via curl, fails from the browser" |

## Cms → search sync pipeline (ADR-0013, ADR-0014)

| Feature | Repo | Key file(s) | Key class/function | Notes / ADR |
|---|---|---|---|---|
| Products-tree document types (code-first) | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Catalog/CatalogContentTypeSeeder.cs` | `CatalogContentTypeSeeder.HandleAsync` | Runs on `UmbracoApplicationStartedNotification`; idempotent — no-ops once types exist |
| Real seed content (stand-in for backoffice authoring) | `Lakbay.Cms` | same file | `CatalogContentTypeSeeder.SeedRealContentAsync` | Set `LAKBAY_FORCE_RESEED_CATALOG=true` to force a redo — never the default, see its doc comment |
| Country/Region/Accommodation geography tree (ADR-0017) | `Lakbay.Cms` | same file | `SeedRealContentAsync` (Country/Region/Accommodation blocks) | Country → Region → Destination → Accommodation, matching ECMS's real structure; Region is one-per-ProductLine, Country is shared |
| Publish → Service Bus | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Catalog/CatalogPublishSyncHandler.cs` | `CatalogPublishSyncHandler.HandleAsync` | Fires on real `ContentPublishedNotification` — resolving Content Pickers correctly is the trickiest part here (see `ResolvePickedContent`/`ResolveProductLineCode`) |
| Service Bus transport | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Catalog/ServiceBusCatalogSyncPublisher.cs`, `ICatalogSyncPublisher.cs` | `ServiceBusCatalogSyncPublisher` | Connection string via `ConnectionStrings:ServiceBus` user-secret |
| Wire registration (composer) | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Catalog/CatalogComposer.cs` | `CatalogComposer.Compose` | Auto-discovered via `Program.cs`'s `AddComposers()` — no manual registration needed elsewhere |
| Shared event shape | `Lakbay.Contracts` | `csharp/CatalogSyncEvent.cs`, `CatalogEntityType.cs` | `CatalogSyncEvent` | One event type for all three entity kinds — `EntityType` discriminates which payload field is populated |
| Sync consumer | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Sync/CatalogSyncFunction.cs` | `CatalogSyncFunction.Run` | Field-scoped `$set`, never a whole-document replace — this is what makes ADR-0014's dual-writer split actually safe |
| Sync function DI/startup | `Lakbay.AvailabilityApi` | `src/Lakbay.AvailabilityApi.Sync/Program.cs` | — | Ensures the `ProductLine.Code` unique index exists on every boot (`CatalogContext.EnsureIndexesAsync`) |

## Content tree (Home + landing pages, ADR-0012, ADR-0015)

| Feature | Repo | Key file(s) | Key class/function | Notes / ADR |
|---|---|---|---|---|
| Content tree document types + seeding | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Content/ContentTreeSeeder.cs` | `ContentTreeSeeder.HandleAsync` | Home Page + 4 landing pages; `sections` is JSON-in-Textarea, not a real Block List — ADR-0015 explains why |
| Shared document-type/data-type builder | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Shared/CmsSchemaBuilder.cs` | `CmsSchemaBuilder` | Used by both `CatalogContentTypeSeeder` and `ContentTreeSeeder` — one copy, not two |
| Content Delivery API | `Lakbay.Cms` | `src/Lakbay.Cms.Web/Program.cs` | `.AddDeliveryApi()` | Also needs `Umbraco:CMS:DeliveryApi:Enabled` in appsettings *and* the CORS allow-list — config alone isn't enough, a real bug found live |
| Fetching Cms content | `Lakbay.Web` | `src/lib/cmsContentApi.ts` | `cmsContentApi` (RTK Query) | Landing pages fetched via Home's children + client-side `productLineCode` match, **not** a direct route lookup — see the file's own comment on the ProductLine/landing-page name collision |
| Rendering Cms content (block registry) | `Lakbay.Web` | `src/components/blocks/BlockRegistry.tsx` | `CmsSections`, `HeroSection`, `ImageTextSection` | Add a new section `type` here *and* in `ContentTreeSeeder` together, or it renders as nothing (logged in dev only) |
| Homepage hero + sections | `Lakbay.Web` | `src/app/page.tsx` | `Home()` | Falls back to static copy if `cmsContentApi` fails — page still works with only `Lakbay.AvailabilityApi` running |
| Collection page hero + sections | `Lakbay.Web` | `src/app/collections/[code]/page.tsx` | `CollectionPage()` | Same fallback pattern as the homepage |

## Shared contract

| Feature | Repo | Key file(s) | Notes / ADR |
|---|---|---|---|
| Canonical schema (source of truth for field shapes) | `Lakbay.Contracts` | `schema/lakbay.graphql` | Change a field here → check every implementer (`Lakbay.Cms`'s doc types, `Lakbay.AvailabilityApi`'s resolvers) and every consumer (`Lakbay.Web`'s TS types) |
| C# mirror types | `Lakbay.Contracts` | `csharp/*.cs` | Hand-written to mirror the schema — no codegen on this side yet, keep in sync by hand |
| TS generated types | `Lakbay.Contracts` | `typescript/generated/types.ts` | Generated via `npm run codegen` from the schema — never hand-edit this file |

## Not built yet (so you stop looking for it)

| Feature | Status | Where it'll live | Blocked on |
|---|---|---|---|
| Checkout / booking | Disabled button only (`Lakbay.Web`'s holiday detail page) | `Lakbay.Booking` | PayMongo sandbox account (Phase 4) |
| Live availability push (SignalR) | Designed (ADR-0008), not implemented | `Lakbay.AvailabilityApi.Sync` + `Lakbay.Web` | Phase 4, depends on Booking existing first |
| A real Umbraco Block List (`priceBands` and Content-tree `sections`) | Attempted for `sections`, didn't round-trip through the Delivery API, pivoted to JSON-in-Textarea (ADR-0015) | `Lakbay.Cms`'s `ContentTreeSeeder`/`CatalogContentTypeSeeder` | Root cause not found within budget — worth revisiting with Umbraco's own source open |
| Per-tree container nodes (fixes the ProductLine/landing-page name collision) | Worked around client-side for now, not fixed at the source | `Lakbay.Cms`'s `ContentTreeSeeder` / `CatalogContentTypeSeeder` | Needs each tree under its own non-routable container — a real (small) restructure |
| Product/section images via real media library | Deliberately deferred — currently plain URL fields | `Lakbay.Cms`'s `CatalogContentTypeSeeder` / `ContentTreeSeeder` | Real photography/licensing sourcing (Phase 5) |

## Debugging quick-reference

| Symptom | Most likely file to check first |
|---|---|
| Storefront shows zero products | Is `Lakbay.Cms` running and has anything been published? `CatalogSeeder`'s auto-seed is retired — an empty Mongo is now expected, not a bug, until Cms publishes something |
| A product's data doesn't match what's in Cms | Check `lakbay_servicebus` logs and `Lakbay.AvailabilityApi.Sync`'s console — is the message actually being consumed? See `08_LOCAL_INFRASTRUCTURE.md`'s verification snippet |
| GraphQL query works via `curl` but fails from `Lakbay.Web` | Almost always CORS (`Query.cs`'s neighbor, `appsettings.Development.json`) or the `ProductFilter` vs `ProductFilterInput` naming trap — see `ProductFilterInputType.cs`'s doc comment |
| A Cms field value looks like `umb://document/...` instead of real data | A Content Picker field being read as plain text instead of resolved — see `CatalogPublishSyncHandler.ResolvePickedContent` |
| `dotnet build`/`dotnet run` fails with a file lock on `.exe` | A previous run of the same app is still alive holding the file — find the exact PID the error names and kill *that PID specifically*, never a blanket `taskkill /IM dotnet.exe` (kills unrelated dotnet processes too) |
| Docker container won't start / `docker ps` empty | See `08_LOCAL_INFRASTRUCTURE.md` — also check Docker Desktop itself isn't stuck on its own first-run onboarding dialog (`%APPDATA%\Docker\settings-store.json`'s `DisplayedOnboarding` flag) |
| A `/collections/[code]` landing page shows the wrong content (or none) | `Lakbay.Cms`'s `ProductLine` nodes and Content-tree landing pages share names — a URL-routing collision. Check `cmsContentApi.ts`'s `getLandingPageContent` is still fetching via Home's children, not a direct `/content/item/{code}` route |
| `Lakbay.Web` fetch to Cms's Content Delivery API fails with a CORS error | `Lakbay.Cms`'s `Program.cs` needs both `Umbraco:CMS:DeliveryApi:Enabled` in appsettings *and* an explicit CORS allow-list (`Cors:AllowedOrigins`) — config alone doesn't add the CORS middleware |
| Cms boots but any Delivery API request throws `Unable to resolve service for type IRequestSegmentService` | `Program.cs`'s Umbraco builder chain is missing `.AddDeliveryApi()` — enabling it via config alone doesn't register its DI services |
