# Manual Setup Guide — the whole Lakbay stack, start to finish

One linear walkthrough, in dependency order, consolidating what's already
proven working in each repo's own `Docs/DEVELOPER_HANDBOOK.md`. Written so
you can set this up **without an AI agent** — every command below has
actually been run on this machine and its real output is quoted, not
assumed.

**If this document and a repo's own handbook ever disagree, the repo's own
handbook wins** — this file is a consolidation, not a second source of
truth. Update both in the same commit if you change a setup step.

**Current status:** Phase 0 is complete for every repo. `Lakbay.Cms` has a
live database connection and a created admin account. `Lakbay.AvailabilityApi`
is further ahead — Phase 1 is done for its query API (real resolvers, real
seeded MongoDB data, 7 passing tests). See `04_TASKS.md` for exactly
what's left.

## 1. Prerequisites

| Tool | Version confirmed on this machine | Needed for |
|---|---|---|
| .NET SDK | 10.0.400 | `Lakbay.Contracts` (C# side), `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.AvailabilityApi` |
| Node.js | 24.18.0 (any 20+ should work) | `Lakbay.Contracts` (TS side), `Lakbay.Web` |
| npm | 11.16.0 | same as Node.js |
| Docker Desktop | 29.7.2 | SQL Server (`Lakbay.Cms`/`Lakbay.Booking`, ADR-0005, §4) and MongoDB (`Lakbay.AvailabilityApi`, §6) — daemon must actually be running, not just installed (`docker ps` should return a table, not a connection error) |
| Azure Functions Core Tools (`func`) | **not installed** as of 2026-09-08 | `Lakbay.AvailabilityApi.Sync` only, and only once Phase 4 gives it a real trigger to run — not needed for anything below |

Check what you actually have before starting:

```bash
dotnet --version && node --version && npm --version && docker --version && docker ps
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
and runs on. Without it, `npm install`/`npm run dev` don't exist.

```powershell
winget install OpenJS.NodeJS
```

Verify: `node --version` and `npm --version`. Reopen your terminal first
if either command isn't found.

**Docker Desktop** — runs SQL Server and MongoDB as disposable containers
so you never install a database server directly onto your machine, and so
"start the database" is one command instead of a multi-step native
install. This is what ADR-0005 means by "local dev runs SQL Server in
Docker."

```powershell
winget install -e --id Docker.DockerDesktop
```

**This one needs a machine restart to finish** (it installs a Windows
virtualization feature) — that's the exact restart this project's own
history already went through. After restarting, open Docker Desktop once
from the Start menu and wait for it to say "Docker Desktop is running"
before trying `docker` commands from a terminal. Verify: `docker ps`
should print an empty table, not a connection error.

**Azure Functions Core Tools** (`func`) — not needed for anything in this
guide yet. When `Lakbay.AvailabilityApi.Sync` gets a real trigger (Phase
4), install it via npm (the officially documented method, more reliable
than guessing a package manager ID from memory):

```powershell
npm install -g azure-functions-core-tools@4 --unsafe-perm true
```

### 1b. A one-sentence job description for the CLI verbs you'll actually type

If you're newer to these tools, this is what each recurring command is
doing — not a full tutorial, just enough to know why you're typing it:

- **`dotnet build`** — compiles a C# project/solution without running it;
  the fastest way to check "does this still compile" after an edit.
- **`dotnet run --project <path>`** — compiles (if needed) and starts a
  C# project as a running process; used for anything that's a server
  (`Lakbay.Cms.Web`, the two `.Api` projects) rather than a library.
- **`dotnet test`** — runs a project's xUnit tests and reports pass/fail;
  every `Lakbay.*.Tests` project in this platform is run this way.
- **`dotnet user-secrets set <key> <value>`** — writes a key/value pair to
  a per-project store *outside* the repo (on Windows:
  `%APPDATA%\Microsoft\UserSecrets\<a GUID from the .csproj>\secrets.json`)
  so a connection string with a real password never has a chance to be
  committed, even by accident. It's read automatically at startup in the
  Development environment — no code change needed to use it.
- **`npm install`** — reads `package.json` and downloads every dependency
  it lists into `node_modules/` (gitignored — never committed, always
  regenerated from `package.json` + `package-lock.json`).
- **`npm run <script>`** — runs a named script from `package.json`'s
  `"scripts"` block (e.g. `npm run codegen`, `npm run dev`, `npm run
  build`) — a project-specific shortcut, not a built-in npm command.
- **`docker compose up -d`** — reads a `docker-compose.yml` in the current
  directory and starts every container it describes, in the background
  (`-d` = detached, so it doesn't tie up your terminal). Every
  `docker-compose.yml` in this platform starts a database only — the
  applications themselves always run natively via `dotnet run`/`npm run
  dev`, never inside the compose file, so you still get fast hot-reload.
- **`docker ps`** — lists currently running containers; the fastest way to
  confirm "is the database actually up" before assuming a connection
  failure is a code problem.

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
# wait for it to report healthy:
docker inspect --format='{{.State.Health.Status}}' lakbay_sqlserver
# creates BOTH umbracoDb (this repo) and lakbayBookingDb (Lakbay.Booking)
# on one shared local SQL Server instance — still two fully separate
# databases (ADR-0003), one container purely for local-dev convenience.

# 3. Point the app at it via .NET user-secrets — NEVER appsettings.json,
#    so the password never lands in a committed file:
cd src/Lakbay.Cms.Web
dotnet user-secrets set "ConnectionStrings:umbracoDbDSN" \
  "Server=localhost,1433;Database=umbracoDb;User Id=sa;Password=<your DB_PASSWORD from .env>;TrustServerCertificate=true"
dotnet user-secrets set "ConnectionStrings:umbracoDbDSN_ProviderName" "Microsoft.Data.SqlClient"

# 4. Run it:
dotnet run --project src/Lakbay.Cms.Web
```

Open `https://localhost:44330/umbraco` (check the console output for the
actual port — it can vary) and complete the install wizard's admin-account
step yourself: real email, real password, your choice. This is the one
step in the whole setup that's deliberately manual — nobody should script
or hand you a value for your own CMS login.

That finishes Phase 0 for this repo and is the actual starting point for
Phase 3 (building the Content + Products trees, ADR-0001).

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
`lakbayBookingDb` is already created on it — and point this repo's own
connection string at `Server=localhost,1433;Database=lakbayBookingDb;...`
via `dotnet user-secrets`, same reasoning as `Lakbay.Cms`.

## 6. Lakbay.AvailabilityApi — the query API half (real resolvers, real MongoDB)

Two deployables in one repo (ADR-0009) — you're setting up the query API
here; the `Sync` Azure Function needs `func` and a real trigger, neither
of which exist yet (Phase 4), so there's nothing to run there today.

```bash
cd ../../Lakbay.AvailabilityApi

# Start MongoDB — no auth, local-only:
docker compose up -d
docker inspect --format='{{.State.Health.Status}}' lakbay_mongo   # wait for "healthy"

dotnet build     # 0 Warning(s), 0 Error(s) across all 3 projects
dotnet test tests/Lakbay.AvailabilityApi.Tests/Lakbay.AvailabilityApi.Tests.csproj
# 7 passed — the test suite spins up its OWN ephemeral MongoDB via
# Testcontainers, separate from the one you just started above

dotnet run --project src/Lakbay.AvailabilityApi.Api
# runs on :5170 in this setup (check console output — launchSettings.json
# can reassign this). First run against an empty database seeds four
# ProductLine records and one real destination/product per line (Coron,
# Baguio, San Fernando Pampanga, Vigan) automatically.
```

Verify it's really serving real data, not just booting:

```bash
curl -s http://localhost:5170/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ productLines { code name } }"}'
```

**Important:** run it with `ASPNETCORE_ENVIRONMENT=Development` set (plain
`dotnet run` does this automatically via `launchSettings.json`; if you use
`--no-launch-profile` you must set it yourself) — the CORS allow-list that
lets `Lakbay.Web` call this locally only loads from
`appsettings.Development.json`. Without it, the browser preflight from
`Lakbay.Web` 404s and nothing obviously tells you why.

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
`http://localhost:5000`.

**Don't delete `AGENTS.md` in this repo** — Next.js 16 generates/re-adds it
itself and it documents real breaking changes from older Next.js versions.
Read it before writing App Router code here.

## 8. Verify it end-to-end, not just repo-by-repo

Run `Lakbay.AvailabilityApi`'s query API (§6) and `Lakbay.Web`'s dev server
(§7) **at the same time**. Open `http://localhost:3000` — the homepage's
`useGetStatusQuery()` call should round-trip through RTK Query, hit the
real GraphQL API, and render its live response on the page. If that works,
the entire chain (Redux Toolkit → RTK Query → GraphQL → HotChocolate) is
proven, not just individually compiling pieces.

## 9. What's genuinely not set up yet on this machine

Don't go looking for these — they don't exist yet, and nothing above needs
them:

- **Azure Functions Core Tools** (`func` CLI) — needed only to actually run
  `Lakbay.AvailabilityApi.Sync` locally (it builds fine without it). Install
  when Phase 4 gives that Function a real trigger to fire on.
- **An Azure Service Bus emulator** (Phase 4, real-time availability
  propagation, ADR-0008) — can run in Docker for fully offline dev once
  that phase starts.
- **CI** — no pipeline exists in any repo yet.
- **GitHub remotes** — every repo above is local-only right now, on
  purpose (open decision, see `04_TASKS.md`).

## 10. If something doesn't match this document

This guide is only as current as the day it was written. Each repo's own
`Docs/DEVELOPER_HANDBOOK.md` is updated the moment its setup actually
changes — check there first if a command above stops working, and update
both files together once you've confirmed the fix, so the next person
(human or AI) doesn't hit the same gap.
