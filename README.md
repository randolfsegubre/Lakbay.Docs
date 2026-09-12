# Lakbay

A Philippines-first holiday and experience platform — curated package
holidays, an independent accommodation marketplace (`/stays`), and a
fixed-price local activity marketplace (`/activities`), booked directly by
consumers. Same business category as Inghams/Hotelplan (curated packages,
per-booking revenue), not SaaS. Four themed product lines: **Alon**
(islands & water adventure), **Amihan** (highlands & cool-climate),
**Parul** (festive/light tourism), **Pamana** (living heritage & culture).

This repo is the platform's shared plan — architecture decisions, the
build history, and cross-repo reference. **No application code lives
here**; the eight repos below are where the platform actually runs.

## The platform, in one line each

| Repo | Job | Stack |
|---|---|---|
| [`Lakbay.Cms`](../Lakbay.Cms) | Editorial CMS + product catalog — one Umbraco backoffice, content and catalog cross-referenced natively | Umbraco 18, .NET 10 |
| [`Lakbay.AvailabilityApi`](../Lakbay.AvailabilityApi) | Real, permanently deployed search/read service — denormalized catalog synced from Cms, modeled on a real production Manticore search service | ASP.NET Core, HotChocolate GraphQL, MongoDB |
| [`Lakbay.Booking`](../Lakbay.Booking) | Orders, availability, and payment orchestration — the atomic, race-safe availability decrement that actually prevents double-booking | .NET minimal API, MediatR (CQRS), gRPC |
| [`Lakbay.Web`](../Lakbay.Web) | The public storefront — fully headless, server-rendered for SEO | Next.js, Redux Toolkit + RTK Query |
| [`Lakbay.AgentDesktop`](../Lakbay.AgentDesktop) | A call-center agent's desktop tool: caller screen-pop, live availability, book on the customer's behalf | WPF/MVVM, Unity Container, WCF |
| [`Lakbay.AgentOps`](../Lakbay.AgentOps) | Backend-for-frontend for the agent channel — offer aggregation, call logging, live push | ABP Framework, Hangfire, Redis, SignalR, Oracle |
| [`Lakbay.Contracts`](../Lakbay.Contracts) | The shared GraphQL schema every service agrees to, versioned as a package | SDL + codegen (C#/TypeScript) |
| `Lakbay.Docs` | This repo — architecture, ADRs, devlog, cross-repo reference | Markdown |

**New here?** [WALKTHROUGH.md](WALKTHROUGH.md) traces three real scenarios end
to end (an editor publishing a package, a customer booking online, an agent
booking by phone) across every repo — the fastest way to see how the pieces
actually fit together, without reading eight repos' worth of code cold.

## What's real here, not just planned

Verified live, not just designed — full trail in
[`docs/05_DEVLOG.md`](docs/05_DEVLOG.md):

- **The full content-to-storefront pipeline**: an editor publishes in
  `Lakbay.Cms` → Azure Service Bus → a dedicated sync Azure Function →
  MongoDB → GraphQL → the Next.js storefront, end to end, with a
  last-write-wins ordering guard against Service Bus's non-strict delivery.
- **Real double-booking prevention** — a single atomic, conditional SQL
  `UPDATE` in `Lakbay.Booking`'s confirm-handler (never read-then-write),
  proven with a real concurrency test (two simultaneous confirms against
  one remaining slot, exactly one succeeds).
- **A gRPC service call between two independently-running processes** —
  `Lakbay.AgentOps` confirms a phone booking against `Lakbay.Booking` over
  gRPC (not REST), verified with both services actually running as
  separate processes, not an in-memory test host. Recovered once from an
  unexplained git history rewrite that had silently dropped it, and a real
  TLS/ALPN configuration bug was found and fixed in the process — see
  [ADR-0027](docs/adr/ADR-0027-agent-channel-confirm-booking-over-grpc.md).
- **A second, genuinely different front-end client** on the same backend —
  a WPF desktop app (`Lakbay.AgentDesktop`) alongside the Next.js storefront,
  talking to a real WCF duplex service for simulated call-center telephony
  screen-pop.
- **A live accommodation and activity marketplace** — 14 real destinations,
  each with real accommodations (filterable by type/tag) and fixed-price,
  itemized local activities — not just curated packages.

Full platform verified running simultaneously (5 services, real Docker
infrastructure) on 2026-09-12 — see the devlog's entry that date for the
complete evidence trail, including the one piece not re-verified that
session (`Lakbay.AgentDesktop`'s own WPF UI — no headless tool exists for
driving a Windows desktop app the way a browser can be driven).

## The architecture decisions that matter

Each has a full [ADR](docs/adr/) — the short version:

- **CMS and product catalog share one Umbraco backoffice**, but
  **transactional booking data lives in a completely separate service and
  database** — Umbraco's content cache is tuned for read-heavy publishing,
  not high-frequency transactional writes.
- **CQRS via MediatR**, not a shared service class with both read and
  write methods — the exact shape that made an earlier project's
  controllers unmaintainable stubs.
- **One backend language** (.NET) across every service except the
  storefront — the search/read API was deliberately moved off Node.js
  once there was no real requirement for a second backend language.
- **Fully headless frontend** — the CMS never renders a page; the
  storefront owns 100% of presentation, talking to GraphQL only. The
  direct opposite of a Razor-hosted-React hybrid.
- **Real-time availability, and correctness, are two separate concerns,
  solved separately** — Service Bus + SignalR push keeps the UI current;
  a database-level atomic decrement is what actually prevents overselling.
  Neither one substitutes for the other.

Full reasoning for every decision: [docs/adr/](docs/adr/). Platform-wide
technical detail: [docs/01_CLAUDE.md](docs/01_CLAUDE.md) (architecture
manual) and [docs/06_SYSTEM_ARCHITECTURE.md](docs/06_SYSTEM_ARCHITECTURE.md)
(diagrams). Full original market research and stack-decision matrix: the
[Lakbay Blueprint](https://claude.ai/code/artifact/f3565f50-0b09-46c3-8923-e393f7ac6099).
