# Lakbay — Platform AI Operating Manual

This file is the source of truth for any AI coding assistant (Claude Code,
or any other LLM) working anywhere in the Lakbay estate — all six repos
under `Personal_Projects/Lakbay/`. Read it before writing code in any of
them. It supersedes generic defaults — where it's specific, follow it over
a more "standard" pattern you might otherwise reach for.

Full market research, the technology-stack decision matrix (adopt/later/skip
for every option considered), and the reasoning behind every call below:
the published **Lakbay Blueprint** artifact —
https://claude.ai/code/artifact/f3565f50-0b09-46c3-8923-e393f7ac6099
That artifact is the design document this repo executes. This file and
`02_BUILD_PLAN.md` are the on-disk, versioned record of the same decisions,
kept in sync with it — if the artifact is ever revised, fold the change
back into these files rather than letting them drift apart.

## 1. What this is

A Philippines-first holiday and experience platform: curated package
holidays across four themed product lines, booked directly by consumers.
Business-model classification, precisely: a **B2C travel e-commerce
platform / online tour operator** — the same category Inghams and
Hotelplan occupy (curated packages, per-booking revenue), not SaaS
(subscription software) and not PaaS (that term describes the Azure
infrastructure this runs *on*, not what it *is*). It becomes an
OTA/marketplace only if third-party operators are later invited to list
their own packages — an open business decision, not an architectural
default. See the Blueprint's executive summary for the full reasoning,
including the one real SaaS opportunity identified (licensing the CMS +
Booking engine itself to other PH operators later, as a second product).

**Product lines** (working names — Filipino words tied to each cluster,
deliberately not copying Inghams/Santa's Lapland naming conventions;
final branding/trademark is a marketing decision, not a technical one):

- **Alon** ("wave") — islands & water adventure: Palawan, Siargao, Boracay, Cebu, Bohol
- **Amihan** ("cool northeast breeze") — highlands & cool-climate escapes: Baguio, Sagada, Mt. Pulag, Tagaytay
- **Parul** ("lantern", Kapampangan) — festive/light tourism: Pampanga's Giant Lantern Festival, Panagbenga, Sinulog
- **Pamana** ("heritage/legacy") — living heritage & culture: Vigan, Ifugao Rice Terraces, Intramuros

## 2. Repo map

| Repo | Role | Stack |
|---|---|---|
| `Lakbay.Docs` | This repo. Platform plan, architecture guide, ADRs, devlog. No app code. | Markdown |
| `Lakbay.Cms` | Unified editorial CMS + product catalog — ECMS and PCMS merged into one Umbraco solution | Umbraco 18, .NET |
| `Lakbay.Booking` | Orders, basket, availability calendar, payment orchestration — deliberately separate from the CMS | .NET minimal API |
| `Lakbay.Web` | Public storefront: marketing pages, catalog browsing, booking flow | Next.js, Redux Toolkit + RTK Query |
| `Lakbay.AvailabilityApi` | Real, permanently deployed product-search service — denormalized read model synced from `Lakbay.Cms`, modeled on Hotelplan's `api-sphinx`/Manticore. **Not a mock** (renamed from `Lakbay.MockApi`) | ASP.NET Core, HotChocolate, MongoDB.Driver |
| `Lakbay.Contracts` | Shared GraphQL SDL schema + generated TS/C# types, versioned as a package | Schema + codegen |

## 3. The load-bearing architecture decisions

Each has a full Architecture Decision Record under `docs/adr/` — this is
the short version; read the ADR before touching code that the decision
governs.

1. **ECMS and PCMS unify into one Umbraco solution** (`Lakbay.Cms`) —
   content pages and the product catalog live in the same backoffice, one
   App Service, one database, native cross-referencing (Content Picker),
   no inter-service API call. See [ADR-0001](adr/ADR-0001-unify-ecms-pcms.md).
2. **Transactional booking data stays out of that database, on purpose**
   — Umbraco's NuCache is tuned for read-heavy, publish-then-cache
   workloads, not high-frequency transactional writes. `Lakbay.Booking`
   owns orders/baskets/availability in its own database, communicating
   over Service Bus events. See
   [ADR-0003](adr/ADR-0003-separate-booking-data.md).
3. **`Lakbay.Booking` uses CQRS (MediatR), not a shared service class** —
   commands and queries have different validation and performance needs;
   a single fat service class is the exact shape that left
   `E-Commerse.AI.API`'s controllers as untestable, disconnected stubs.
   See [ADR-0002](adr/ADR-0002-cqrs-booking.md).
4. **`Lakbay.AvailabilityApi` is .NET (HotChocolate + MongoDB.Driver), not
   Node.js/Apollo** — the only place the original plan introduced a second
   backend language without a real requirement behind it. One backend
   language across `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Contracts`, and
   `Lakbay.AvailabilityApi`; only `Lakbay.Web` is genuinely a different stack.
   See [ADR-0004](adr/ADR-0004-mockapi-dotnet-not-node.md).
5. **Local development runs SQL Server in Docker, not Azure SQL Database**
   — Azure SQL Database is cloud-only PaaS with no local edition; it's
   used only once a live/staging environment exists (Phase 5). See
   [ADR-0005](adr/ADR-0005-local-sql-server-not-azure-sql.md).
6. **The frontend is fully headless, not Razor-hosted** — `Lakbay.Cms`
   never renders a page or holds UI code; `Lakbay.Web` is a standalone
   Next.js app owning 100% of presentation, talking to GraphQL only.
   Next.js (SSR/ISR) for SEO-critical pages, RTK Query for
   server-state/caching, plain Redux Toolkit slices only for genuinely
   client-side state (booking wizard, basket). This is the direct
   opposite of the ECMS/Prototype hybrid (Razor page shells in the CMS
   with React embedded inside them). See
   [ADR-0006](adr/ADR-0006-headless-cms-no-razor-ui.md).
7. **`Lakbay.AvailabilityApi` is real, permanently deployed infrastructure, not
   a disposable mock** — checking the actual Hotelplan `api-sphinx`
   commit history (55 commits, described as the highest-risk-per-change
   repo doing real price/availability search) showed the "just a mock"
   framing was wrong from the start. It's a dedicated, denormalized search
   read-model synced from `Lakbay.Cms`, and `Lakbay.Web` queries it in
   production for catalog browsing/search/filtering — it is not repointed
   away from once `Lakbay.Cms` exists. See
   [ADR-0007](adr/ADR-0007-searchapi-is-real-not-mock.md), and
   [06_SYSTEM_ARCHITECTURE.md](06_SYSTEM_ARCHITECTURE.md) for exactly how
   this fits alongside `Lakbay.Cms` and `Lakbay.Booking`.
8. **Availability changes propagate live, no polling** — a confirmed
   booking in `Lakbay.Booking` reaches `Lakbay.AvailabilityApi` via Service Bus,
   which updates its read model and pushes a change notification over
   Azure SignalR Service to any `Lakbay.Web` page currently showing that
   listing. Deliberately not the same thing as preventing overselling
   (that's concurrency control inside `Lakbay.Booking`, unaffected by this
   decision). See
   [ADR-0008](adr/ADR-0008-realtime-availability-propagation.md).
9. **`Lakbay.AvailabilityApi` splits query-serving from event-consumption**
   — a separate Azure Function (`Lakbay.AvailabilityApi.Sync`,
   `[ServiceBusTrigger]`) handles Cms-sync and `AvailabilityChanged`
   events, so the GraphQL query API is never slowed by write/sync load.
   The sync function only applies an update if it's newer than what's
   stored (last-write-wins, guarding against Service Bus's lack of strict
   ordering). See [ADR-0009](adr/ADR-0009-availabilityapi-rename-and-split.md)
   and [ADR-0010](adr/ADR-0010-last-write-wins-sync.md).
10. **Double-booking is prevented at the database, not by anything above**
    — `Lakbay.Booking`'s confirm-booking handler uses a single atomic,
    conditional SQL `UPDATE` (never read-then-write) so two
    near-simultaneous bookings for the same slot can't both succeed. This
    is entirely independent of items 8 and 9 above — real-time propagation
    makes the UI accurate, this makes the booking correct. See
    [ADR-0011](adr/ADR-0011-atomic-availability-decrement.md).
11. **Umbraco content blocks render via a React block-registry** —
    `Lakbay.Cms`'s Block List/Grid JSON maps element-type alias to a
    `Lakbay.Web` React component, the standard headless-CMS
    component-mapping pattern. See
    [ADR-0012](adr/ADR-0012-block-rendering-in-react.md).

## 4. Engineering practice — non-negotiable, not aspirational

Every non-obvious structural choice (a new abstraction, a SOLID trade-off,
a design pattern) gets written down where the next developer — junior or
veteran, human or AI — will actually look, not left to be reverse-engineered
from a diff. This is a workflow requirement enforced at PR review, in every
repo, from the first commit. The concrete mechanism —
which OOP pillar, which SOLID principle, which named pattern, mapped to
real Lakbay components with the rejected alternative stated for each —
lives in [03_ARCHITECTURE_AND_PATTERNS_GUIDE.md](03_ARCHITECTURE_AND_PATTERNS_GUIDE.md).

**Every repo must also stay maintainable by a human alone, offline, with
no AI agent available.** Day-to-day building will mostly be AI-assisted,
but nothing in any repo's structure may depend on that being true later.
Concretely, once a repo reaches Phase 0 exit (see `02_BUILD_PLAN.md`), it
must have its own `Docs/DEVELOPER_HANDBOOK.md` with exact offline-runnable
local setup (Docker Compose for SQL Server/Mongo/Redis — pull images once,
run with no internet after) and worked "adding a new X" walkthroughs. The
test for every doc in this system: could a developer follow it on a plane,
no internet, no agent?

TDD applies in every repo: a failing test for a rule or resolver comes
first, before the implementation. Per-repo test stack is specified in the
Blueprint's "Testing & TDD strategy" section and repeated in each repo's
own `CLAUDE.md`.

## 5. Start here — first session on any repo

1. Read this file in full (you're doing that now).
2. Read [04_TASKS.md](04_TASKS.md) for current phase/status.
3. Read [02_BUILD_PLAN.md](02_BUILD_PLAN.md)'s section for the phase
   you're about to work in — don't read the whole build plan cover to
   cover, pull in the phase that's actually active.
4. Read the ADRs listed in that repo's own `CLAUDE.md` under "Decisions
   this repo must honor."
5. Check the most recent [05_DEVLOG.md](05_DEVLOG.md) entries for
   anything that changed since the build plan was last touched.

## 6. Open items (do not silently resolve these — surface them)

- Final platform/product-line naming and trademark search — marketing
  decision, not engineering.
- Exact GraphQL-on-Umbraco package for `Lakbay.Cms` (community package vs.
  a hand-rolled resolver layer over the Content Delivery API) — needs a
  short spike before `Lakbay.Contracts` schema v0 is treated as locked.
- No GitHub remotes exist yet for any of the six repos — local-only as of
  the Phase 0 scaffolding session (2026-09-03). Creating remotes is a
  separate, explicit decision (see [[working-style]] on GitHub token
  scope) — don't assume it's wanted without asking.
- Business model: proprietary tour operator vs. later opening to
  third-party listings (marketplace/OTA) — different unit economics,
  deliberately left open.
- Regulatory: Philippine DOT accreditation requirements for listed
  operators — legal check, can gate launch, not an engineering task.
- Exact sync trigger from `Lakbay.Cms` to `Lakbay.AvailabilityApi` (Umbraco
  content-cache-refresher event, Service Bus message, or scheduled
  Hangfire job) — flagged as genuine new scope by ADR-0007, not yet
  decided. Needs deciding before Phase 1 is considered done.
