# Lakbay.Docs — Start Here

This file is intentionally short. It exists so any Claude Code session (or
other AI coding assistant) rooted here auto-loads it and is pointed at the
real documentation before touching anything.

**Read, in this order, before doing anything else:**

1. [docs/01_CLAUDE.md](docs/01_CLAUDE.md) — the platform AI operating
   manual. Constitution for the whole Lakbay estate (eight repos now,
   including the Agent Channel's `Lakbay.AgentDesktop`/`Lakbay.AgentOps`);
   if anything else conflicts with it, it wins unless the user explicitly
   overrides it in the current conversation.
2. [docs/04_TASKS.md](docs/04_TASKS.md) — what phase the platform is on
   right now and what's left in it.
3. The most recent entries at the top of
   [docs/05_DEVLOG.md](docs/05_DEVLOG.md) — what happened in the last few
   sessions, and why.
4. [WALKTHROUGH.md](WALKTHROUGH.md) — the accessible, narrative version of
   "how do the repos actually talk to each other" — three real scenarios
   traced end to end (a catalog publish, a booking, an agent-assisted
   booking), for building a working mental model fast rather than piecing
   it together from ADRs one at a time.

Everything else under `docs/` — the phased [02_BUILD_PLAN.md](docs/02_BUILD_PLAN.md),
the [03_ARCHITECTURE_AND_PATTERNS_GUIDE.md](docs/03_ARCHITECTURE_AND_PATTERNS_GUIDE.md),
and the individual records under [docs/adr/](docs/adr/) — is living
reference, pulled in when the phase or task at hand calls for it, not read
cover-to-cover up front.

**`docs/index.html`** — the "Lakbay Developer Atlas," a single-file
rendered snapshot for human onboarding (overview, architecture diagrams,
setup walkthrough, a searchable feature map, troubleshooting table, ADR
list). Open it directly in a browser, or serve it — it's plain,
dependency-free HTML/CSS/JS (Mermaid loads from a public CDN), so GitHub
Pages can serve it as-is from this folder with no build step. It's a
snapshot, not a source: the markdown files above are what to read and
edit; this file only gets manually regenerated to match them, so treat
any conflict between the two as the markdown being right.

**This repo has no application code.** If you were pointed here while
working inside `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web`,
`Lakbay.AvailabilityApi`, `Lakbay.Contracts`, `Lakbay.AgentDesktop`, or
`Lakbay.AgentOps`, that repo's own `CLAUDE.md`/`WALKTHROUGH.md` is what
should have loaded automatically — come back here only for the
platform-wide "why."

**End of session:** before finishing any session that changed a plan,
phase status, or made a new architectural decision, update
`docs/04_TASKS.md` and append a dated entry to `docs/05_DEVLOG.md`. A new
structural decision gets its own numbered file under `docs/adr/`, not a
paragraph buried in the devlog.
