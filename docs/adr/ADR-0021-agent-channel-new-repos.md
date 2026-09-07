# ADR-0021: Add an Agent Channel — two new repos, Lakbay's first second front-end client

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** New: `Lakbay.AgentDesktop`, `Lakbay.AgentOps`. Existing: `Lakbay.Booking` (new channel), `Lakbay.Docs`.

## Context

A call-center agent needs to take an inbound customer call, look up who's
calling, browse the same live accommodation/package/activity availability
`Lakbay.Web` shows a self-service shopper, and book the customer directly
over the phone — without the customer ever touching the website. This is
a real product need (not every customer books online), and it happens to
be the natural place to close out several stack gaps identified against
target job postings: a WPF desktop client, an ABP-based backend service,
a WCF integration point, and Oracle as a second persistence engine.

The temptation with "close these gaps" work is to bolt unrelated tech
onto existing repos wherever they fit syntactically. That produces a
portfolio that reads as a checklist, not a system. This ADR's whole point
is the opposite: every new technology below only appears because a real
call-center agent tool actually needs it, and each one is named to a
specific job posting's stated requirement so the reasoning stays
traceable later.

## Decision

Two new repos, siblings to the existing six under `Lakbay/`:

1. **`Lakbay.AgentDesktop`** — a WPF (.NET) desktop application, MVVM,
   Unity as the composition-root container (ADR-0022). This is the
   agent's actual tool: caller lookup, live availability browse, booking
   confirmation. It talks to `Lakbay.AgentOps` over HTTP/GraphQL — never
   directly to any database, exactly like `Lakbay.Web` never does.
2. **`Lakbay.AgentOps`** — a new backend microservice, built on the ABP
   Framework, owning agent sessions, call logging, and an
   agent-shaped aggregation of catalog + availability + package data
   (bundling what `Lakbay.AvailabilityApi` and `Lakbay.Cms` expose
   separately into one "offer" view suited to a live phone call). Uses
   Hangfire, Redis, and SignalR internally (ADR-0025).

This makes `Lakbay.AgentDesktop` the **second real front-end client**
against the same headless backend `Lakbay.Web` already proves out
(ADR-0006) — the same catalog and availability data, a completely
different presentation technology, no shared UI code, no coupling beyond
the API contract. That's the multi-client-integration story this system
was always implicitly capable of; this ADR is the first thing that
actually exercises it.

`Lakbay.Booking` gains an agent channel (ADR-0026) rather than a
duplicate booking system — one booking domain, two ways to reach it.

## Alternatives considered

- **Add agent features directly into `Lakbay.Web`** (an "agent mode" in
  the existing Next.js app) — rejected: a call-center agent's tool is a
  genuinely different UI paradigm (keyboard-driven, screen-pop-triggered,
  often running on locked-down call-center desktop images where a
  installed native app is normal and a browser tab is not), and conflating
  it with the public storefront would pull unrelated concerns into one
  codebase. It also would not exercise the WPF/desktop gap at all, which
  is the actual reason this work exists.
- **One combined repo for the WPF client and its backend** — rejected for
  the same reason `Lakbay.Cms` and `Lakbay.Web` are separate repos: the
  desktop client and its backend deploy independently, are owned by
  different concerns (UI framework vs. service), and following the
  established one-repo-per-deployable convention keeps this consistent
  with the rest of the platform rather than a special case.
- **Skip a dedicated backend and have `Lakbay.AgentDesktop` call
  `Lakbay.AvailabilityApi`/`Lakbay.Booking` directly** — rejected: the
  agent's aggregated "offer" view, call logging, and session state are
  real responsibilities that don't belong in either of those services,
  and this is also the only place ABP/Hangfire/Redis/SignalR — a
  bundled stack named together in one target posting — has a genuine home.

## Consequences

- `02_BUILD_PLAN.md` gains a new phase (Phase 7 — Agent Channel) alongside
  the existing six.
- `01_CLAUDE.md`'s repo table grows from six repos to eight.
- `Lakbay.AgentOps` becomes a second consumer of `Lakbay.Contracts`
  (alongside `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.AvailabilityApi`) —
  no new shared-schema mechanism needed.
- This is the first ADR in the platform's history driven by portfolio/gap
  coverage rather than a product requirement discovered mid-build — worth
  naming explicitly rather than pretending it emerged organically.
