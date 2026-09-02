# Devlog

Reverse-chronological. One entry per session (or per phase boundary within
a long session) that changed a plan, wrote code, or made a decision.
Format: date, what was asked, what changed and why, what's next.

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
