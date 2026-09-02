# Tasks — Current Status

**Current phase:** Phase 0 — Foundation & scaffolding (see `02_BUILD_PLAN.md`)

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

## Not done — rest of Phase 0

- [ ] Confirm exact local toolchain versions (.NET SDK for Umbraco 17/
      Booking/SearchApi, Node.js for Lakbay.Web only, Docker Desktop) on
      this machine
- [ ] `Lakbay.Contracts` schema v0 (Product, ProductLine, Destination
      types) written and committed
- [ ] `Lakbay.Cms` — empty Umbraco 17 solution scaffolded, boots to
      install wizard
- [ ] `Lakbay.Booking` — empty .NET minimal API + xUnit test project
      scaffolded
- [ ] `Lakbay.Web` — empty Next.js + Redux Toolkit project scaffolded
- [ ] `Lakbay.SearchApi` — empty ASP.NET Core + HotChocolate project
      scaffolded, MongoDB via Docker Compose
- [ ] Decide the Cms → SearchApi sync trigger mechanism (Umbraco event,
      Service Bus message, or scheduled job) — flagged by ADR-0007
- [ ] SQL Server (Docker, Developer Edition) Compose service defined for
      `Lakbay.Cms` and `Lakbay.Booking`, per ADR-0005
- [ ] CI skeleton in every repo
- [ ] `Docs/DEVELOPER_HANDBOOK.md` stub in each of the five application
      repos, with real (proven, not assumed) local setup steps
- [ ] Decide: GitHub remotes for these repos, or stay local-only for now —
      open item, needs an explicit answer, not an assumption

## Blocked / needs a decision before it can proceed

- Exact GraphQL-on-Umbraco package for `Lakbay.Cms` (see `01_CLAUDE.md`
  §6) — blocks locking `Lakbay.Contracts` schema v0 as final.
- Package registry choice (GitHub Packages vs. Azure Artifacts) for
  publishing `Lakbay.Contracts`.
- Production MongoDB hosting for `Lakbay.SearchApi` (Azure Cosmos DB for
  MongoDB vs. MongoDB Atlas) — a Phase 5 decision, not urgent now.
