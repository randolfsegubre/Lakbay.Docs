# Manual Setup Guide — the whole Lakbay stack, start to finish

One linear walkthrough, in dependency order, consolidating what's already
proven working in each repo's own `Docs/DEVELOPER_HANDBOOK.md`. Written so
you can set this up **without an AI agent** — every command below has
actually been run on this machine and its real output is quoted, not
assumed.

**If this document and a repo's own handbook ever disagree, the repo's own
handbook wins** — this file is a consolidation, not a second source of
truth. Update both in the same commit if you change a setup step.

**Current status (2026-09-08):** Phase 0-2 complete everywhere. Phase 3's
Products-tree sync (`Lakbay.Cms` → `Lakbay.AvailabilityApi`, ADR-0013/0014)
is proven live end-to-end. `Lakbay.Cms` is now the **sole source** of
catalog data — `Lakbay.AvailabilityApi`'s old auto-seed is retired. See
`04_TASKS.md` for exactly what's left. New to this repo? Also read
`08_LOCAL_INFRASTRUCTURE.md` (what each Docker container is for) and
`09_FEATURE_MAP.md` (where a given feature's code actually lives).

## 1. Prerequisites

| Tool | Version confirmed on this machine | Needed for |
|---|---|---|
| .NET SDK | 10.0.400 | `Lakbay.Contracts` (C# side), `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.AvailabilityApi` |
| Node.js | 24.18.0 (any 20+ should work) | `Lakbay.Contracts` (TS side), `Lakbay.Web` |
| npm | 11.16.0 | same as Node.js |
| Docker Desktop | 29.7.2 | SQL Server, MongoDB, the Service Bus emulator — daemon must actually be running, not just installed (`docker ps` should return a table, not a connection error) |
| Azure Functions Core Tools (`func`) | 4.14.0 | `Lakbay.AvailabilityApi.Sync` — required now (Phase 3 gave it a real trigger) |

Check what you actually have before starting:

```bash
dotnet --version && node --version && npm --version && docker --version && docker ps && func --version
```

### 1a. Installing whatever's missing (Windows — this machine)

Each tool below is genuinely doing one job in this project; the note says
what that job is so a command isn't just something to copy blindly.

**.NET SDK** — compiles and runs every C# project in this platform
(`Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.AvailabilityApi`, and the C#
half of `Lakbay.Contracts`). Without it, `dotnet build`/`dotnet run` don't
exist as commands at all.

```powershell
winget install Microsoft.DotNet.SDK.10
```

Close and reopen your terminal afterward — `PATH` changes from an
installer don't apply to an already-open shell. Verify: `dotnet --version`.

**Node.js** (npm comes bundled with it) — runs the TypeScript codegen for
`Lakbay.Contracts` and is the entire runtime `Lakbay.Web` (Next.js) builds
and runs on.

```powershell
winget install OpenJS.NodeJS
```

**Docker Desktop** — runs SQL Server, MongoDB, and the Service Bus
emulator as disposable containers so you never install any of them
directly onto your machine. See `08_LOCAL_INFRASTRUCTURE.md` for what
each container actually does.

```powershell
winget install -e --id Docker.DockerDesktop
```

**This one needs a machine restart to finish** (it installs a Windows
virtualization feature). After restarting, open Docker Desktop once from
the Start menu and **complete its first-run onboarding screen** (accept
the terms, skip sign-in if you don't want an account) before trying
`docker` commands — if you skip this, the daemon never actually starts
even though the process looks like it's running. Verify: `docker ps`
should print an empty table, not a connection error. If it hangs for more
than a couple minutes with `docker ps` still failing, check
`%APPDATA%\Docker\settings-store.json` for `"DisplayedOnboarding": false`
— that's the tell that the onboarding screen, not a slow backend, is the
actual blocker.

**Azure Functions Core Tools** (`func`) — runs `Lakbay.AvailabilityApi.Sync`
locally; required as of Phase 3 (it has a real Service Bus trigger now).

```powershell
npm install -g azure-functions-core-tools@4 --unsafe-perm true
```

### 1b. A one-sentence job description for the CLI verbs you'll actually type

If you're newer to these tools, this is what each recurring command is
doing — not a full tutorial, just enough to know why you're typing it:

- **`dotnet build`** — compiles a C# project/solution without running it.
- **`dotnet run --project <path>`** — compiles (if needed) and starts a
  C# project as a running process; used for anything that's a server.
- **`dotnet test`** — runs a project's xUnit tests and reports pass/fail.
- **`dotnet user-secrets set <key> <value>`** — writes a key/value pair to
  a per-project store *outside* the repo (on Windows:
  `%APPDATA%\Microsoft\UserSecrets\<a GUID from the .csproj>\secrets.json`)
  so a connection string with a real password never has a chance to be
  committed. Read automatically at startup in the Development environment.
- **`npm install`** — reads `package.json` and downloads dependencies into
  `node_modules/` (gitignored).
- **`npm run <script>`** — runs a named script from `package.json`.
- **`docker compose up -d`** — starts every container a `docker-compose.yml`
  describes, in the background. Every compose file in this platform starts
  infrastructure only — applications always run natively.
- **`docker ps`** — lists running containers; confirms "is the database
  actually up" before assuming a connection failure is a code problem.
- **`func start`** — builds (if needed) and starts the Azure Functions
  host for `Lakbay.AvailabilityApi.Sync`; keeps running, listening for
  Service Bus messages, until you `Ctrl+C` it.

## 2. Repo layout

All six repos are siblings under one folder, not nested inside each other:

```
Personal_Projects/Lakbay/
  Lakbay.Docs/            this repo — no application code
  Lakbay.Contracts/       set up FIRST — everything else references it
  Lakbay.Cms/
  Lakbay.Booking/
  Lakbay.AvailabilityApi/
  Lakbay.Web/
```

Set them up in that order. `Lakbay.Cms`, `Lakbay.Booking`, and
`Lakbay.AvailabilityApi` all take a **project reference** (not a package)
to `../Lakbay.Contracts/csharp/Lakbay.Contracts.csproj`, and `Lakbay.Web`
takes a `file:` dependency on `../Lakbay.Contracts/typescript` — none of
them will build until `Lakbay.Contracts` exists on disk in the right
relative position.

## 3. Lakbay.Contracts — the shared schema (no dependencies, no database)

```bash
cd Lakbay.Contracts

# TypeScript side
cd typescript
npm install
npm run codegen        # writes generated/types.ts from ../schema/lakbay.graphql
npx tsc --noEmit        # should report zero errors

# C# side
cd ../csharp
dotnet build            # should report 0 Warning(s), 0 Error(s)
```

No Docker, no database — this repo is schema and generated types only.

## 4. Lakbay.Cms — Umbraco 18, unified content + product catalog

```bash
cd ../../Lakbay.Cms

# 1. Local-only SQL Server password (gitignored — never a real secret):
cp .env.example .env
# edit .env, set DB_PASSWORD to whatever you want locally

# 2. Start SQL Server (Developer Edition, free, in Docker):
docker compose up -d
docker inspect --format='{{.State.Health.Status}}' lakbay_sqlserver
# creates BOTH umbracoDb (this repo) and lakbayBookingDb (Lakbay.Booking)
# on one shared local SQL Server instance (ADR-0003 — still two fully
# separate databases; one container is purely local-dev convenience).

# 3. Point the app at it via .NET user-secrets — NEVER appsettings.json:
cd src/Lakbay.Cms.Web
dotnet user-secrets set "ConnectionStrings:umbracoDbDSN" \
  "Server=localhost,1433;Database=umbracoDb;User Id=sa;Password=<your DB_PASSWORD from .env>;TrustServerCertificate=true"
dotnet user-secrets set "ConnectionStrings:umbracoDbDSN_ProviderName" "Microsoft.Data.SqlClient"

# 4. Service Bus connection (ADR-0013 — needed so publishing content
#    actually reaches Lakbay.AvailabilityApi; see §6 for starting the
#    emulator itself, and 08_LOCAL_INFRASTRUCTURE.md for what this is):
dotnet user-secrets set "ConnectionStrings:ServiceBus" \
  "Endpoint=sb://localhost;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true;"

# 5. Run it:
dotnet run --project src/Lakbay.Cms.Web
```

**First-ever run**: Umbraco's install wizard appears at
`https://localhost:44330/umbraco` (check console output — the port can
vary). Complete the admin-account step yourself — real email, real
password, your choice. This is deliberately the one manual step in the
whole setup; nobody should script or hand you a value for your own CMS
login, and the credential is never recorded anywhere in this repo or in
memory, even locally.

**After that first run**, two seeders run automatically on every boot:
`CatalogContentTypeSeeder` creates the Products-tree document types and
seeds/publishes four real Philippine destinations/products;
`ContentTreeSeeder` creates a Home Page + one landing page per product
line with real copy and photos. Both publish immediately on first
creation — which exercises the full sync pipeline to
`Lakbay.AvailabilityApi` (so start that and the Service Bus emulator
first — see §6 — or the publish events just queue harmlessly until the
Sync function is running to consume them).

`Lakbay.Web` reads the Content tree via Umbraco's built-in Content
Delivery API (`Program.cs`'s `.AddDeliveryApi()`,
`Umbraco:CMS:DeliveryApi:Enabled` in appsettings, plus the same
`Cors:AllowedOrigins` pattern the query API uses) — all already wired up
in this repo, nothing extra to configure for a normal `dotnet run`.

## 5. Lakbay.Booking — orders, basket, availability (no database yet)

```bash
cd ../../../Lakbay.Booking
dotnet build     # 0 Warning(s), 0 Error(s)
dotnet test      # 1 passed — the /health endpoint boots
dotnet run --project src/Lakbay.Booking.Api
# GET http://localhost:<port>/health → {"status":"ok","service":"Lakbay.Booking"}
```

No database needed for this much — Phase 0 only proves the service boots.
**When real work starts here (Phase 4):** don't stand up a second SQL
Server container. Reuse the one `Lakbay.Cms` already starts (§4 above) —
`lakbayBookingDb` is already created on it.

## 6. Lakbay.AvailabilityApi — the read model + sync consumer

Two deployables in one repo (ADR-0009): a GraphQL query API, and a
separate Azure Function (`Lakbay.AvailabilityApi.Sync`) that's the only
thing allowed to write to this repo's MongoDB.

```bash
cd ../../Lakbay.AvailabilityApi

# 1. Local-only Service Bus emulator password (gitignored):
cp .env.example .env
# edit .env, set DB_PASSWORD to whatever you want locally (used by the
# emulator's required sqledge companion — see 08_LOCAL_INFRASTRUCTURE.md)

# 2. Start MongoDB + the Service Bus emulator:
docker compose up -d
docker inspect --format='{{.State.Health.Status}}' lakbay_mongo   # wait for "healthy"
docker logs lakbay_servicebus --tail 20
# look for "Emulator Service is Successfully Up!" and confirmation that
# the "lakbay-catalog-sync" queue was created

dotnet build     # 0 Warning(s), 0 Error(s) across all 4 projects
dotnet test tests/Lakbay.AvailabilityApi.Tests
# 8 passed — each test spins up its OWN ephemeral MongoDB via
# Testcontainers and calls CatalogSeeder explicitly (Program.cs no
# longer auto-seeds — see §6a below)
```

**Start the Sync function** (needs `func`, §1a):

```bash
cd src/Lakbay.AvailabilityApi.Sync
func start
# Functions:
#   CatalogSyncFunction: serviceBusTrigger
```

Leave this running in its own terminal — it's what actually applies
`Lakbay.Cms`'s published content to MongoDB.

**Start the query API** (separate terminal):

```bash
cd ../Lakbay.AvailabilityApi.Api
dotnet run
# runs on :5170 in this setup (check console output — launchSettings.json
# can reassign this)
```

Verify it's serving real data:

```bash
curl -s http://localhost:5170/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ productLines { code name } products(filter: {}) { name heroImageUrl } }"}'
```

If `products` comes back empty, that's expected until `Lakbay.Cms` has
published something — see §6a.

**Important:** run it with `ASPNETCORE_ENVIRONMENT=Development` set (plain
`dotnet run` does this automatically via `launchSettings.json`) — the CORS
allow-list that lets `Lakbay.Web` call this locally only loads from
`appsettings.Development.json`.

### 6a. Where the catalog data actually comes from now

`Lakbay.AvailabilityApi.Api`'s `CatalogSeeder` (Phase 1's hand-seeded PH
data) **no longer runs automatically** — retired 2026-09-08 once
`Lakbay.Cms` became the real source of truth (running both left duplicate
documents for the same holidays under different IDs). It's kept only as a
test-support utility now, called explicitly by
`Lakbay.AvailabilityApi.Tests`.

**In practice this means:** to see real products in `Lakbay.Web`, you now
need `Lakbay.Cms` running with its own database populated (§4 handles
this automatically via `CatalogContentTypeSeeder`), plus the Service Bus
emulator and the Sync function running to carry that data across. If you
only want to poke at the GraphQL API in isolation with no Cms running,
MongoDB will simply be empty — that's expected, not a setup failure.

## 7. Lakbay.Web — the storefront

```bash
cd ../Lakbay.Web
npm install
npm run build    # compiles clean
npm run lint     # zero errors
npm run dev      # http://localhost:3000
```

Optional: set `NEXT_PUBLIC_AVAILABILITY_API_URL` in a gitignored
`.env.local` if `Lakbay.AvailabilityApi` isn't running on the default
`http://localhost:5170`.

**Don't delete `AGENTS.md` in this repo** — Next.js 16 generates/re-adds it
itself and it documents real breaking changes from older Next.js versions.

**If you ever see "Module not found: Can't resolve '@lakbay/contracts'"**:
a known Turbopack + sibling-repo gotcha, already fixed in `next.config.ts`
(`transpilePackages` + a widened `turbopack.root`) and `tsconfig.json` (a
`paths` alias) — see `Lakbay.Web`'s own `Docs/DEVELOPER_HANDBOOK.md`.

**If a hero image doesn't render**: the external host needs to be in
`next.config.ts`'s `images.remotePatterns` — `next/image` silently
refuses any host not explicitly allowed.

## 8. Verify it end-to-end, not just repo-by-repo

With `lakbay_sqlserver`, `lakbay_mongo`, `lakbay_servicebus` (Docker),
the Sync function (`func start`), `Lakbay.Cms` (`dotnet run`),
`Lakbay.AvailabilityApi.Api` (`dotnet run`), and `Lakbay.Web`
(`npm run dev`) **all running at the same time**:

1. Open `http://localhost:3000` — the homepage should render all four
   product lines (Alon, Amihan, Parul, Pamana) with real names/taglines.
2. Click into `/collections/alon` — you should see "Coron Island Hopping,
   3 Days 2 Nights" from ₱12,500, with a real photo.
3. Click into it for the full detail page (itinerary, board basis,
   availability, price bands, hero image banner).
4. **The actual Phase 3 proof, not just Phase 2's:** open
   `https://localhost:44330/umbraco`, edit that product's summary, publish
   it. Within a second or two, reload the holiday detail page in
   `Lakbay.Web` — the new summary should appear, with **no code change
   and no redeploy of anything**. If that works, the full chain — Umbraco
   publish → `CatalogPublishSyncHandler` → Service Bus →
   `CatalogSyncFunction` → MongoDB → GraphQL → RTK Query → React — is
   proven end-to-end, not just individually compiling pieces.

(A GraphQL query that only works via `curl` with inline literal arguments
is *not* proof it works for a real client — see the `ProductFilterInput`
gotcha in `Lakbay.AvailabilityApi`'s handbook / `09_FEATURE_MAP.md` for
exactly why that distinction mattered here.)

## 9. What's genuinely not set up yet on this machine

Don't go looking for these — they don't exist yet, and nothing above needs
them:

- **The Content tree** (pages, Block List components, ADR-0012's block
  registry) — only the Products tree exists in `Lakbay.Cms` so far.
- **A real Block List for `priceBands`** — currently a JSON-in-Textarea v0
  simplification; deliberately deferred to the same session that builds
  the Content tree above.
- **Real product media** (Media Picker / uploaded photos) — hero images
  are currently plain URL text fields pointing at Wikimedia Commons.
- **CI** — no pipeline exists in any repo yet.
- **GitHub remotes** — every repo above is local-only right now, on
  purpose (open decision, see `04_TASKS.md`).
- **Lakbay.Booking's real logic, real-time SignalR push** — Phase 4,
  blocked on a PayMongo sandbox account.

## 10. If something doesn't match this document

This guide is only as current as the day it was written. Each repo's own
`Docs/DEVELOPER_HANDBOOK.md` is updated the moment its setup actually
changes — check there first if a command above stops working, and update
both files together once you've confirmed the fix, so the next person
(human or AI) doesn't hit the same gap.
