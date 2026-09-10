# ADR-0025: Lakbay.AgentOps is built on ABP Framework, with Hangfire, Redis, and SignalR

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.AgentOps`

## Context

Four technologies — ABP Framework, Hangfire, Redis, SignalR — are named
together as one bundled stack against a single target job posting. They
also happen to combine naturally in exactly the service `Lakbay.AgentOps`
already needs to be (ADR-0021): a backend with real background work, a
caching need, and a real-time push requirement. Building `Lakbay.AgentOps`
on this stack isn't four separate justifications bolted together — it's
one coherent backend that happens to need all four.

## Decision

**ABP Framework** as the application framework for `Lakbay.AgentOps`
(not plain ASP.NET Core, unlike every other backend in this platform):

- Module system: `AgentOpsDomainModule`, `AgentOpsApplicationModule`,
  `AgentOpsEntityFrameworkCoreModule`, `AgentOpsHttpApiModule` — ABP's
  standard layering, chosen deliberately so this repo demonstrates ABP's
  actual conventions rather than a minimal "hello world" module.
- ABP's own DI (built on `Microsoft.Extensions.DependencyInjection`
  under the hood, via conventional registration) — not Unity. Unity is
  scoped specifically to `Lakbay.AgentDesktop`'s composition root
  (ADR-0022); forcing it into ABP would fight the framework's own
  auto-registration conventions for no real benefit.
- Application services expose both a conventional REST API (ABP's
  auto-API-controller generation) and are the backing for one GraphQL
  endpoint (via HotChocolate, matching `Lakbay.AvailabilityApi`'s
  precedent) for the aggregated "offer" query specifically, since that's
  the one query shape `Lakbay.AgentDesktop` benefits from composing
  flexibly.

**Hangfire** for background work: async call-record writes to Oracle
(ADR-0024), a post-call follow-up email job, and a retry job for booking
confirmations that failed to reach `Lakbay.Booking` on the first attempt
(fire-and-retry, not fire-and-forget — a failed background job here would
otherwise silently lose a phone booking). Hangfire's own dashboard
(`/hangfire`, auth-gated) doubles as a real operational tool for
diagnosing a stuck job during development.

**Redis** (`StackExchange.Redis`, via ABP's Redis distributed-cache
integration) caches the aggregated "offer" view per destination — the
same accommodation/package/activity bundle multiple agents look up
repeatedly during a live call, worth serving from cache rather than
re-querying `Lakbay.AvailabilityApi`/`Lakbay.Cms` on every lookup.
Cache invalidation ties to the same Service Bus events
`Lakbay.AvailabilityApi.Sync` already consumes (ADR-0013) — `Lakbay.AgentOps`
subscribes to `CatalogSyncEvent` and invalidates just the affected
destination's cache entry, never a blanket flush.

**SignalR** (`AgentAvailabilityHub`) pushes live availability changes to
every connected `Lakbay.AgentDesktop` client, the same event stream
`Lakbay.AvailabilityApi.Sync` already produces — this is the first real
implementation of the live-push mechanism ADR-0008 designed for
`Lakbay.Web` but never built; `Lakbay.AgentOps` proves the pattern with a
second, independent consumer.

## Alternatives considered

- **Plain ASP.NET Core instead of ABP** — the choice every other backend
  in this platform makes, and explicitly not repeated here: ABP is the
  actual named requirement, and using it only in the one service that
  exists partly to demonstrate it keeps the decision traceable rather
  than diluting it across the whole platform.
- **Azure SignalR Service** (which `Lakbay.Web`'s own future integration,
  per Phase 4 of `02_BUILD_PLAN.md`, is planned to use) — not used for
  `Lakbay.AgentOps` locally: self-hosted SignalR (`Microsoft.AspNetCore.SignalR`)
  is simpler to run without any Azure dependency for local development,
  and is architecturally interchangeable with Azure SignalR Service later
  (same hub code, different transport backplane) if this service is ever
  scaled across multiple instances.
- **A generic `BackgroundService`/`IHostedService` instead of Hangfire**
  — rejected: Hangfire is the named requirement, and it offers real,
  visible value here beyond a bare hosted service (persistent job
  storage surviving a restart, retry policies, the dashboard) that this
  service's actual jobs — especially the booking-confirmation retry —
  genuinely benefit from.

## Consequences

- `Lakbay.AgentOps` needs its own SQL Server database for ABP's identity
  and Hangfire's job-storage tables, on top of the Oracle call-log
  database (ADR-0024) and Redis — three infrastructure dependencies for
  one service, all documented together in this repo's own setup docs.
- ABP's opinions (module system, auto-API-controllers, its own identity/
  permission model) mean this repo's internal conventions genuinely
  diverge from `Lakbay.Cms`/`Lakbay.Booking`/`Lakbay.AvailabilityApi` —
  worth a short "if you know the rest of this platform, here's what ABP
  does differently" section in this repo's own `CLAUDE.md`.
- The SignalR hub's contract (`AvailabilityChanged(destinationId, ...)`)
  should stay shape-compatible with whatever `Lakbay.Web`'s own future
  Azure SignalR integration (Phase 4) ends up using, so the two aren't
  reinvented independently later.
