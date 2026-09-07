# ADR-0024: Oracle hosts the agent call-log/CRM read side — deliberate polyglot persistence

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.AgentOps`

## Context

Every database in this platform so far is SQL Server (`Lakbay.Cms`,
`Lakbay.Booking`, dev-local per ADR-0005) or MongoDB
(`Lakbay.AvailabilityApi`, ADR-0004). Oracle has no natural entry point in
any of that — and forcing it in artificially (e.g. "let's also support
Oracle for the catalog") would be exactly the checklist-driven design
ADR-0021 explicitly rejects.

A realistic reason Oracle shows up in a greenfield platform is the one
that actually happens in real call centers: **the call center's telephony
logging / CRM system already exists on Oracle**, predates this platform,
and isn't being migrated off just because a new booking system launched.
`Lakbay.AgentOps` needing to write call records and read agent
performance history against that pre-existing system is a genuine
integration reason, not a manufactured one.

## Decision

`Lakbay.AgentOps` owns its own Oracle database (`LAKBAY_AGENTOPS_CALLS`,
locally: Oracle Database Free — formerly XE, installed natively on
Windows, not in Docker) for exactly two things:

1. **Call records** — one row per call: caller number, matched customer
   (if any), agent id, call start/end, outcome (booked / no-sale /
   follow-up needed), written asynchronously via a Hangfire job (ADR-0025)
   so a slow Oracle write never blocks the live call.
2. **Agent performance read model** — calls-per-day, booking conversion
   rate per agent, queried for a simple "my stats" panel in
   `Lakbay.AgentDesktop`. Read-only from the application's perspective;
   this data is derived from call records, not written directly.

Everything else `Lakbay.AgentOps` owns (agent session state, the cached
"offer" aggregation) stays on SQL Server/Redis, per every other repo's
existing convention — Oracle's scope is deliberately narrow, matching
"the one thing that has a real reason to be there," not "the new default."

Access via Oracle's official `Oracle.EntityFrameworkCore` provider —
same EF Core-first pattern as `Lakbay.Cms`/`Lakbay.Booking`, so a
developer already familiar with this platform's SQL Server repos isn't
learning a second data-access idiom, only a second provider.

## Alternatives considered

- **Also put Oracle in front of the catalog or booking data** — rejected:
  there is no realistic reason for it, and ADR-0021 already commits to
  not adding technology without one. Oracle's entire justification here
  is "an existing system we're integrating with," and that justification
  only covers call/CRM data.
- **Oracle via Docker** — the user's own initial framing for this decision
  and the more portable long-term choice, but not what was chosen: Docker
  Desktop was unreliable in this environment during this build (see
  `Lakbay.Docs`'s own devlog, 2026-09-07 entry, for the specific crash),
  so a native Oracle Database Free install was used instead to make actual
  progress rather than block the whole Agent Channel build on one
  container runtime. `docker-compose.yml` support for Oracle should still
  be added once Docker is reliable again, for parity with every other
  repo's "infrastructure only in Docker" convention (07_MANUAL_SETUP_GUIDE.md).
- **Devart/ODP.NET instead of the official EF Core provider** — rejected:
  the official Oracle-maintained EF Core provider is the more standard,
  better-documented choice for a new build with no existing ODP.NET
  investment to preserve.

## Consequences

- `Lakbay.AgentOps`'s local setup gains a real, non-trivial prerequisite
  (Oracle Database Free installed natively) beyond what any other repo in
  this platform needs — must be documented clearly in this repo's own
  setup docs and cross-linked from `07_MANUAL_SETUP_GUIDE.md`, not left
  implicit.
- `Lakbay.AgentOps` is now the platform's only repo touching three
  different persistence technologies at once (SQL Server, Redis, Oracle)
  — a real polyglot-persistence example worth pointing to directly in an
  interview, not just an accident of gap-filling.
- Production hosting (Phase 5-equivalent for this new phase) needs an
  actual decision on where the production Oracle instance lives (Oracle
  Autonomous Database on OCI, or a customer-provided on-prem instance
  reached via VPN) — explicitly deferred, not decided in this ADR.
