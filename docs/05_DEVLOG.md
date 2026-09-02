# Devlog

Reverse-chronological. One entry per session (or per phase boundary within
a long session) that changed a plan, wrote code, or made a decision.
Format: date, what was asked, what changed and why, what's next.

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
