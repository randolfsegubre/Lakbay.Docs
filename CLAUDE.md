# Lakbay.Docs — Start Here

This file is intentionally short. It exists so any Claude Code session (or
other AI coding assistant) rooted here auto-loads it and is pointed at the
real documentation before touching anything.

**Read, in this order, before doing anything else:**

1. [docs/01_CLAUDE.md](docs/01_CLAUDE.md) — the platform AI operating
   manual. Constitution for the whole Lakbay estate (all six repos); if
   anything else conflicts with it, it wins unless the user explicitly
   overrides it in the current conversation.
2. [docs/04_TASKS.md](docs/04_TASKS.md) — what phase the platform is on
   right now and what's left in it.
3. The most recent entries at the top of
   [docs/05_DEVLOG.md](docs/05_DEVLOG.md) — what happened in the last few
   sessions, and why.

Everything else under `docs/` — the phased [02_BUILD_PLAN.md](docs/02_BUILD_PLAN.md),
the [03_ARCHITECTURE_AND_PATTERNS_GUIDE.md](docs/03_ARCHITECTURE_AND_PATTERNS_GUIDE.md),
and the individual records under [docs/adr/](docs/adr/) — is living
reference, pulled in when the phase or task at hand calls for it, not read
cover-to-cover up front.

**This repo has no application code.** If you were pointed here while
working inside `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web`,
`Lakbay.SearchApi`, or `Lakbay.Contracts`, that repo's own `CLAUDE.md` is
the one that should have loaded automatically — come back here only for
the platform-wide "why."

**End of session:** before finishing any session that changed a plan,
phase status, or made a new architectural decision, update
`docs/04_TASKS.md` and append a dated entry to `docs/05_DEVLOG.md`. A new
structural decision gets its own numbered file under `docs/adr/`, not a
paragraph buried in the devlog.
