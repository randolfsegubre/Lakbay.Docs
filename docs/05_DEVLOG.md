# Devlog

Reverse-chronological. One entry per session (or per phase boundary within
a long session) that changed a plan, wrote code, or made a decision.
Format: date, what was asked, what changed and why, what's next.

---

## 2026-09-08 (2) — Consolidated manual setup guide; Lakbay.Cms admin account created

**Asked:** a single guide for setting up the whole stack manually, without
relying on an AI agent to run the commands each time.

**What changed:**

- Wrote `07_MANUAL_SETUP_GUIDE.md`: one linear, dependency-ordered
  walkthrough (Contracts → Cms → Booking → AvailabilityApi → Web) built
  from what each repo's own `Docs/DEVELOPER_HANDBOOK.md` already had
  proven working, plus a fresh tool-version check on this machine
  (`dotnet 10.0.400`, `node 24.18.0`, `docker 29.7.2`; `func` and `mongod`
  confirmed still absent) so nothing in it is stated from memory.
- Randolf completed the Umbraco install wizard's admin-account step in the
  browser and offered the credential directly, since the environment is
  local-only. Declined to record the actual password anywhere — not this
  devlog, not `04_TASKS.md`, not memory — on the reasoning that a repo's
  "local-only" status is a current fact, not a permanent guarantee (the
  GitHub-remotes decision is still open), and credentials shouldn't
  normalize into version-controlled or cross-machine-synced text. Recorded
  only that the step is done.

**What's next:** unchanged from the prior entry — Phase 3 content/product
trees in `Lakbay.Cms`, or Phase 1 real resolvers in `Lakbay.AvailabilityApi`,
whichever Randolf wants to pick up first. MongoDB compose, CI, and the
GitHub-remotes decision are still open and still non-blocking.

---

## 2026-09-08 — Docker unblocked by machine restart; SQL Server live, Umbraco boots against a real DB

**Asked:** "I have restarted the machine. What's next?" — the previous
session had left exactly one blocker recorded in `04_TASKS.md`: Docker
Desktop's install needed a restart to finish.

**What changed:**

- Confirmed Docker Desktop actually finished installing and the daemon is
  reachable (`docker ps`).
- Ran `Lakbay.Cms`'s proven Phase 0 sequence from its own
  `Docs/DEVELOPER_HANDBOOK.md`: created `.env` with a local-only SQL
  Server password (gitignored, confirmed via `git check-ignore`),
  `docker compose up -d`, polled until `lakbay_sqlserver` reported
  `healthy`, wired the connection string via `dotnet user-secrets`
  (never `appsettings`), and started `Lakbay.Cms.Web`.
- Verified for real, not assumed: the app is listening on both configured
  ports, the backoffice module bundle loads with no exceptions in the
  log, and `curl` against `/umbraco` returns 200 with no error/exception
  markers in the response — the install wizard is live against a real
  database connection.
- Deliberately did **not** complete the wizard's admin-account step
  (email/password/name) — that's a real credential choice for whoever
  owns this CMS, not something to script or invent a value for. Left the
  server running; the browser step is the one thing still manual.
- Updated `Lakbay.Cms/Docs/DEVELOPER_HANDBOOK.md` and this platform's
  `04_TASKS.md` to move these steps from "pending Docker" to "proven
  working," matching the project's own rule that handbooks record what
  was actually verified, not what was planned.

**What's next:** open `https://localhost:44330/umbraco` in a browser and
finish the install wizard's admin-account step — that closes out Phase 0
for `Lakbay.Cms` entirely and is the actual start of Phase 3 (real content
+ product trees, per ADR-0001). Separately, still open: MongoDB compose
for `Lakbay.AvailabilityApi`, CI skeletons in every repo, and the
GitHub-remotes decision — none blocked on anything now.

---

## 2026-09-06 (4) — Phase 0 implementation: all five repos scaffolded and verified

**Asked:** "Let's start implementing. Go ahead and do the implementations
from top to bottom... complete everything that can be done in local."
Also asked whether to continue in this session or start a new one.

**Decision on session structure:** recommended separate sessions per repo
going forward (each repo's `CLAUDE.md` is self-contained for exactly this
reason), but did the one genuinely shared, foundational piece
(`Lakbay.Contracts`) plus a first pass at the rest here, since Phase 0
scaffolding is mechanical and benefited from staying in one thread this
first time.

**What changed — real, verified, not just planned:**

- **Toolchain check:** .NET SDK 10.0.400, Node 24.18.0 confirmed present.
  Docker Desktop was **not** installed — flagged immediately rather than
  assumed; user chose to install it (`winget install -e --id
  Docker.DockerDesktop`) while everything Docker-independent proceeded in
  parallel.
- **`Lakbay.Contracts`:** wrote `schema/lakbay.graphql` v0, a hand-written
  C# class library mirroring it, and a TypeScript package generating real
  types via `@graphql-codegen` — ran the generator, verified `tsc
  --noEmit` clean. Committed the generated `types.ts` on purpose (it's
  shipped output, not a disposable build artifact — updated `.gitignore`'s
  reasoning accordingly).
- **`Lakbay.Booking`:** scaffolded minimal API + xUnit, `dotnet test`
  green (1 passed).
- **`Lakbay.AvailabilityApi`:** scaffolded per ADR-0009's two-project
  split — query API (HotChocolate) and a separate
  `Lakbay.AvailabilityApi.Sync` Azure Function project (installed
  `Microsoft.Azure.Functions.Worker.ProjectTemplates` fresh, swapped its
  default HTTP extension for the Service Bus one actually needed). Fixed
  a `FunctionsApplicationBuilder`/`ConfigureFunctionsWorkerDefaults` API
  mismatch the newer minimal builder pattern doesn't need. `dotnet test`
  green.
- **`Lakbay.Web`:** `create-next-app@latest` (Next.js 16.3.4, React
  19.2.8) — hit an npm package-name validation error scaffolding directly
  into the capital-letter `Lakbay.Web` folder, worked around by scaffolding
  into a temp lowercase subfolder and moving contents up, merging the
  three files that already existed (`.gitignore`, `CLAUDE.md`, `README.md`)
  by hand rather than overwriting. Added Redux Toolkit + RTK Query at
  latest, wired a real `availabilityApi` slice and Redux `Provider`, built
  and linted clean.
- **`Lakbay.Cms`:** installed `Umbraco.Templates` fresh — **discovered
  latest is 18.1.1, not 17** as every prior document assumed (the same
  category of mistake as the original "MockApi" framing: stated without
  checking). Scaffolded, confirmed via an actual browser screenshot that
  it boots to the real "Install Umbraco" wizard. Adapted the official
  `dotnet new umbraco-compose` template's SQL Server container (Dockerfile
  + setup/healthcheck scripts) into a trimmed, database-only
  `docker-compose.yml` matching ADR-0005 exactly — the official template
  also containerizes the Umbraco app itself, which ADR-0005 deliberately
  doesn't call for; one shared SQL Server instance creates both `umbracoDb`
  and `lakbayBookingDb`. Initialized `.NET user-secrets` for the connection
  string (never `appsettings.json`, keeps the SA password out of any
  committed file).
- **Cross-repo end-to-end proof:** ran `Lakbay.AvailabilityApi`'s query
  API and `Lakbay.Web`'s dev server simultaneously. First attempt failed
  (CORS — the browser's preflight `OPTIONS /graphql` 404'd with no CORS
  middleware configured); fixed by adding a configurable CORS policy read
  from `Cors:AllowedOrigins`, and separately caught that `dotnet run
  --no-launch-profile` defaults to the `Production` environment, so
  `appsettings.Development.json` wasn't even being loaded — fixed by
  setting `ASPNETCORE_ENVIRONMENT=Development` explicitly. After both
  fixes: the homepage's `useGetStatusQuery()` round-tripped for real and
  rendered "Lakbay.AvailabilityApi says: ok" in a live browser screenshot.
- **Preview tooling gotcha:** `preview_start` resolves `.claude/launch.json`
  configs from the fixed workspace root
  (`D:\_DEV\Personal_Projects\.claude\launch.json`), not from wherever a
  Bash shell's `cd` happens to be, and not from a per-repo
  `Lakbay.Web/.claude/launch.json` — a first attempt silently launched an
  unrelated existing config (`devhub-web`) instead of erroring. Fixed by
  adding the `lakbay-web-dev` entry to the workspace-root file, using
  `npm --prefix <path>` since that file format has no `cwd` field.
- Every doc updated to match reality: the Umbraco 17→18 correction
  propagated via `sed` across the living docs (historical ADR-0001 text
  left alone, matching the project's own convention); `04_TASKS.md`
  rewritten with what's actually done vs. genuinely Docker-blocked; all
  five repos' `Docs/DEVELOPER_HANDBOOK.md` written from what was proven,
  including the exact commands that failed and what fixed them.

**What's next:** Docker Desktop finishing install, then `docker compose
up -d` from `Lakbay.Cms`, completing the Umbraco install wizard for real,
and Phase 1's real `Lakbay.AvailabilityApi` resolvers against seeded
MongoDB data.

---

## 2026-09-06 (3) — AvailabilityApi rename + split, ordering guard, real double-booking fix, block rendering

**Asked:** Four things in one message. (1) Decouple event consumption
from query serving, "like hooks" — worried the search API would slow down
carrying that extra load. (2) Cache/read-model updates should respect
"record from db is newer than the cache." (3) The actual goal behind all
of this: two near-simultaneous bookings for the same slot must not both
succeed. (4) "Lakbay.SearchApi" is too generic — rename it. Plus a
separate question: how do Umbraco's Block List/Grid components reach the
page without Razor — can React handle that?

**What changed:**

- [ADR-0009](adr/ADR-0009-availabilityapi-rename-and-split.md):
  `Lakbay.SearchApi` renamed to **`Lakbay.AvailabilityApi`** (folder
  renamed on disk via robocopy after a direct `mv`/`Move-Item` failed on
  Windows `.git` permissions both times — git history verified intact
  both times before deleting the old folder) and split into two
  deployables sharing one repo: the GraphQL query API (pure read,
  untouched by events) and `Lakbay.AvailabilityApi.Sync`, an Azure
  Function with a `[ServiceBusTrigger]` — the direct .NET equivalent of a
  webhook, confirmed against current Microsoft docs before writing it up.
- [ADR-0010](adr/ADR-0010-last-write-wins-sync.md): the sync function only
  applies an incoming update if its source timestamp is newer than what's
  stored, guarding against Service Bus's lack of strict ordering
  guarantees — exactly the mechanism Randolf described.
- [ADR-0011](adr/ADR-0011-atomic-availability-decrement.md): the actual
  double-booking fix, and the one most important to get right — an
  atomic, conditional `UPDATE ... WHERE AvailableCount > 0` inside
  `Lakbay.Booking`'s confirm-booking handler, relying on SQL Server's row
  locking rather than any read-then-write application logic. Written up
  explicitly as independent of ADR-0008/0009/0010 — no amount of fast
  propagation to the read side prevents this race, only a correct atomic
  write on the booking side does.
- [ADR-0012](adr/ADR-0012-block-rendering-in-react.md): confirmed (web
  search against current Umbraco/headless-CMS practice) that a React
  block-registry — mapping each Umbraco element-type alias to a
  component, walking the Content Delivery API's block JSON — is the
  standard pattern here, the same idea Sanity/Contentful/Storyblok use
  under different names. Not a gap Lakbay needs to solve novel.
- Propagated the `SearchApi` → `AvailabilityApi` rename across every
  living doc (`01_CLAUDE.md`, `02_BUILD_PLAN.md`,
  `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`, `06_SYSTEM_ARCHITECTURE.md`,
  root `README.md`, `Lakbay.Docs/CLAUDE.md`, and the five application
  repos' own `CLAUDE.md`/`README.md`) — via targeted `sed` on files that
  are pure current-state, by hand on `04_TASKS.md`/this file where past
  entries needed to keep their historical wording intact. ADR-0004,
  0007, and 0008 keep `SearchApi`/`MockApi` in their original text
  deliberately — they are point-in-time records of decisions made under
  those names, not rewritten.

**What's next:** unchanged — rest of Phase 0 scaffolding, now including
the `Lakbay.AvailabilityApi.Sync` Function project and a concurrency test
proving ADR-0011's fix actually works under simulated simultaneous
requests.

---

## 2026-09-06 (2) — Real-time availability propagation designed (ADR-0008)

**Asked:** "I want real time data changes reflected in the UI... an
accommodation that were sold out same day will not show in the listings
anymore... I don't think that feature is not in Inghams or HotelPlan...
I don't know yet how to do that." — raised immediately after the
`Lakbay.SearchApi` rename work, in the same session.

**What changed:** Designed and documented
[ADR-0008](adr/ADR-0008-realtime-availability-propagation.md): when
`Lakbay.Booking` confirms a booking, it publishes `AvailabilityChanged`
over Service Bus; `Lakbay.SearchApi` (already the Service Bus consumer
for `Lakbay.Cms` sync, per ADR-0007) also consumes this, updates its read
model, and pushes a change notification over Azure SignalR Service (a
service already in the stack, originally scoped for exactly this);
`Lakbay.Web` holds a live SignalR connection per listing page and
invalidates just the affected item's RTK Query cache entry — no polling,
no full reload. Explicitly separated this from overselling prevention,
which is a concurrency-control problem inside `Lakbay.Booking`'s own
confirm-handler, unaffected by how fast the UI elsewhere updates — flagged
so it isn't mistaken for solved by this ADR. Updated
`06_SYSTEM_ARCHITECTURE.md` (new section, updated diagram, updated
`Lakbay.SearchApi`/`Lakbay.Booking`/`Lakbay.Web` rows), `02_BUILD_PLAN.md`
Phase 4, `01_CLAUDE.md`'s decision list, and the three affected repos'
`CLAUDE.md` files.

**What's next:** unchanged — rest of Phase 0 scaffolding. Phase 4 now
carries explicit real-time-propagation and concurrency-control scope that
wasn't spelled out before.

---

## 2026-09-06 — MockApi was never a mock: renamed to SearchApi, real role, full architecture doc written

**Asked:** Two things. (1) "I assume MockApi is our version of Sphinx-Api,
right? But do we call it MockApi? Aren't we going to use it as our API
application for real just like Sphinx-Api?" (2) A request for the
per-repo and whole-platform architecture to be written into the docs,
since this is a multi-application integration system.

**What changed:** Checked [[technical_playbook]] before answering rather
than trusting memory of it — `api-sphinx` at Hotelplan is real,
production code (55 commits, "the highest-risk-per-change repo," direct
Manticore search-engine query logic backing real price/availability/
product filtering, called by multiple consumer apps). The original
"disposable mock" framing for this repo was wrong from the start; it
followed Randolf's own casual first-message wording too literally instead
of checking what the real precedent actually was.

Corrected via [ADR-0007](adr/ADR-0007-searchapi-is-real-not-mock.md):

- `Lakbay.MockApi` renamed to **`Lakbay.SearchApi`** (folder renamed on
  disk, git history preserved — verified via `git log` after the move).
  Real, permanently deployed — not retired at any phase.
- It's a denormalized, facet-indexed read model of the catalog, synced
  **from** `Lakbay.Cms` (the authoring source of truth), queried by
  `Lakbay.Web` in production — the same role Manticore played at
  Hotelplan, MongoDB in its place.
- Recognized this as the same CQRS-at-the-repo-level pattern
  (ADR-0002/ADR-0003) applied one level up, at the whole-platform scale:
  `Lakbay.Cms` write side, `Lakbay.SearchApi` read side.
- Rewrote `02_BUILD_PLAN.md`'s Phase 1/3/5: Phase 3 is no longer "repoint
  `Lakbay.Web` from mock to real backend" — `Lakbay.Web` never stops
  talking to `Lakbay.SearchApi`; what changes at Phase 3 is that
  `Lakbay.SearchApi`'s data source switches from hand-seeded to synced
  from `Lakbay.Cms`, with zero `Lakbay.Web` code changes. This is a
  cleaner proof of the shared-contract story than the original framing,
  not just a correction.
- Wrote [06_SYSTEM_ARCHITECTURE.md](06_SYSTEM_ARCHITECTURE.md): per-repo
  architecture (ownership, internal shape, integration points) for all
  six repos, plus the whole-platform integration diagram and the "why
  this shape, not a monolith" synthesis across ADR-0001/0003/0006/0007.
- Propagated the rename everywhere it was stated:
  `01_CLAUDE.md`, `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`, `04_TASKS.md`,
  `Lakbay.SearchApi/CLAUDE.md`+`README.md`, `Lakbay.Web/CLAUDE.md`+`README.md`,
  `Lakbay.Contracts/CLAUDE.md`+`README.md`, `Lakbay.Docs/CLAUDE.md`, the
  root `README.md`, the Lakbay Blueprint artifact, and the Lakbay System
  Map diagram.

**What's next:** unchanged — rest of Phase 0 scaffolding, now including
deciding the `Lakbay.Cms` → `Lakbay.SearchApi` sync trigger mechanism
(flagged as new, real scope by ADR-0007, not assumed solved).

---

## 2026-09-05 (2) — Formalized the headless-CMS decision as ADR-0006

**Asked:** Randolf questioned whether Next.js/TypeScript meant Lakbay was
quietly heading back toward the ECMS/Prototype hybrid (Razor page shells
in the CMS with React embedded inside them) — Next.js being a React
framework, and TypeScript sounding like it could layer the same way.

**What changed:** Confirmed it's the opposite, and — since this decision
had only ever been stated in prose (the Blueprint and `Lakbay.Web/CLAUDE.md`)
— wrote it up properly as
[ADR-0006](adr/ADR-0006-headless-cms-no-razor-ui.md): `Lakbay.Cms` never
renders a page or holds Razor/UI code for the public site; `Lakbay.Web`
owns 100% of presentation. Updated `01_CLAUDE.md`'s decision list,
`Lakbay.Web/CLAUDE.md`, and `Lakbay.Cms/CLAUDE.md` to point at the ADR
instead of leaving it as a "no formal ADR yet" placeholder — closing a gap
I'd flagged myself but not yet acted on.

**What's next:** unchanged — rest of Phase 0 scaffolding.

---

## 2026-09-05 — MockApi stack changed to .NET; local DB story clarified

**Asked:** Two questions. (1) Can `Lakbay.MockApi` be .NET instead of
Node.js, so it uses MongoDB from .NET rather than Node? (2) Since
everything is being built local-only for now, can Azure SQL Database run
locally for free, or should local dev use SQL Server and only switch to
Azure SQL Database once live?

**What changed:**

- [ADR-0004](adr/ADR-0004-mockapi-dotnet-not-node.md): `Lakbay.MockApi`
  moves from Node.js + Apollo Server to ASP.NET Core + HotChocolate +
  MongoDB.Driver. This was the one repo in the whole plan using a second
  backend language without a real requirement behind it — now `Lakbay.Cms`,
  `Lakbay.Booking`, `Lakbay.Contracts`, and `Lakbay.MockApi` are all .NET;
  only `Lakbay.Web` is genuinely a different stack.
- [ADR-0005](adr/ADR-0005-local-sql-server-not-azure-sql.md): confirmed
  Azure SQL Database has no local/offline edition — it's cloud-only PaaS,
  even though it does have a real, no-cost-forever free tier (100,000
  vCore-seconds + 32GB/month) that just doesn't help with fully-offline
  local dev. Local dev for `Lakbay.Cms`/`Lakbay.Booking` runs SQL Server
  Developer Edition in Docker instead; Azure SQL Database is used only
  once a live/staging environment exists (Phase 5).
- Propagated both changes everywhere they were previously stated:
  `01_CLAUDE.md`'s repo map and decisions list, `02_BUILD_PLAN.md`'s
  Phase 0/1 sections, `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`,
  `04_TASKS.md`, `Lakbay.MockApi/CLAUDE.md` + `README.md` +
  `.gitignore`, the published Lakbay Blueprint artifact, and the Lakbay
  System Map diagram.

**What's next:** unchanged from the previous entry — the rest of Phase 0
scaffolding, now with the corrected stack for `Lakbay.MockApi` and the
SQL Server Docker service to define for `Lakbay.Cms`/`Lakbay.Booking`.

---

## 2026-09-03 — Repos created, Phase 0 documentation written

**Asked:** Create the actual repositories for the Lakbay applications, and
build the same kind of pre-implementation documentation/phasing used on
Galaxy Survivor and Ophir Mineral Ventures, before writing any application
code.

**What changed:**

- Six git repositories created under `Personal_Projects/Lakbay/`:
  `Lakbay.Docs`, `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web`,
  `Lakbay.MockApi`, `Lakbay.Contracts`. This is the first genuinely
  multi-repo personal project (Galaxy Survivor, DevHub, and Ophir Mineral
  Ventures are all single-repo), so the established single-repo doc
  pattern (`Docs/01_CLAUDE.md` … numbered files … `DEVLOG.md`) was adapted
  into a dedicated `Lakbay.Docs` repo holding the cross-cutting plan, with
  each application repo carrying a thin `CLAUDE.md` pointing back to it —
  so the "why" isn't duplicated five times.
- Wrote `01_CLAUDE.md` (platform AI operating manual), `02_BUILD_PLAN.md`
  (phases 0–6 with entry/exit criteria and which repos each phase
  touches), `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md` (the OOP/SOLID/pattern
  tables), and the first three ADRs — all adapted from the previously
  published Lakbay Blueprint artifact rather than re-derived from scratch.
- Each of the five application repos got a README, a thin `CLAUDE.md`
  pointer, and a stack-appropriate `.gitignore`. No application code was
  written — deliberately, per the request to have the documentation/phase
  plan exist *before* implementation starts.

**Why the phase order in `02_BUILD_PLAN.md` builds `Lakbay.Web` against
`Lakbay.MockApi` before `Lakbay.Cms` exists:** this carries forward the
Sphinx-API/Mantincore pattern from the Hotelplan estate on purpose — the
whole point is that frontend work is never blocked on backend
availability. Phase 3's exit criterion (repointing `Lakbay.Web` at
`Lakbay.Cms` with zero frontend code changes) is the actual proof that the
shared `Lakbay.Contracts` schema held.

**What's next:** the remainder of Phase 0 — confirm local toolchain
versions, scaffold each repo to an empty-but-buildable baseline, write
`Lakbay.Contracts` schema v0, stand up CI skeletons. See `04_TASKS.md` for
the exact checklist.
