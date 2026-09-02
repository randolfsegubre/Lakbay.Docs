# Devlog

Reverse-chronological. One entry per session (or per phase boundary within
a long session) that changed a plan, wrote code, or made a decision.
Format: date, what was asked, what changed and why, what's next.

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
