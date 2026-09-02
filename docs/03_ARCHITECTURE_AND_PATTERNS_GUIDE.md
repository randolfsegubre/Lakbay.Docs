# Architecture & Patterns Guide

This is the "why is it built like that" reference for every `Lakbay.*`
repo. It exists so a developer — junior or veteran, human or AI, online or
offline — can understand a structural choice without reverse-engineering
it from a diff. Pulled from the Lakbay Blueprint artifact's "Design-rationale
& documentation practice" section; kept here as the versioned, on-disk
copy of record.

**The test for everything in this file:** could someone follow it on a
plane, no internet, no AI agent, using only this repo?

## OOP pillars — where each one actually earns its place

| Pillar | Where it shows up in Lakbay | Repo(s) |
|---|---|---|
| Encapsulation | Domain entities protect their own invariants — `Booking.Confirm()` is a method, not `booking.Status = Confirmed` from outside. If a rule governs a state change, it lives inside the type that owns the state. | Booking, Cms |
| Abstraction | `IProductCatalogRepository`, `IPaymentGateway`, `IAvailabilityService` hide Umbraco, PayMongo, and SQL specifics from the application layer — callers depend on what a thing does, never on how. | Cms, Booking |
| Inheritance | Used sparingly, only for genuine is-a relationships (e.g. a shared `AuditableEntity` base for created/modified metadata). Business variation is handled with composition and interfaces, not deep class hierarchies — deep inheritance for behavior that varies is exactly what tends to break Liskov Substitution later. | Cms, Booking |
| Polymorphism | `IPaymentGateway` (PayMongo today, Stripe later) and `INotificationChannel` (email, SMS) are swapped via dependency injection — new implementations, zero changes to the code that calls them. | Booking |

## SOLID — which principle, applied where, and why

| Principle | Applied to | Why it matters here |
|---|---|---|
| Single Responsibility | MediatR command/query handlers — one handler, one job (`ConfirmBookingCommandHandler` confirms; it does not also send the confirmation email) | A handler that does one thing is testable in one assertion and safe to change without side effects elsewhere. |
| Open/Closed | Payment gateways, notification channels | New gateway = new class implementing the interface, not a growing `if`/`switch` inside existing code. |
| Liskov Substitution | Every `IPaymentGateway` implementation | Enforced with the same contract-test suite run against both the PayMongo sandbox and a fake — any implementation that can't pass it isn't a valid substitute, by definition. |
| Interface Segregation | Booking service's read vs. write surfaces | `Lakbay.Web`'s GraphQL layer depends only on the narrow read interfaces it actually needs — it can't accidentally call a write method it was never meant to reach. |
| Dependency Inversion | Application/domain layer depends on interfaces; Umbraco and EF Core implementations live in an Infrastructure layer, wired at the composition root | This is the exact discipline `E-Commerse.AI.API` skipped — interfaces were defined but never registered in DI, leaving the controllers wired to nothing. Every Lakbay repo's Phase exit criteria include proving the DI graph resolves, not just that the interface exists. |

## Design patterns — named, and why chosen over the alternative

| Pattern | Used for | Alternative rejected |
|---|---|---|
| Repository | `IProductCatalogRepository` over Umbraco's content queries | Querying `IPublishedContentQuery` directly from handlers — untestable without a live Umbraco instance, and couples domain logic to a specific CMS API. |
| CQRS (via MediatR) | Separate Commands (`PlaceBookingCommand`) and Queries (`GetAvailabilityQuery`) in `Lakbay.Booking` — see [ADR-0002](adr/ADR-0002-cqrs-booking.md) | One fat `BookingService` class handling both reads and writes — the exact shape that left `E-Commerse.AI.API`'s controllers as disconnected stubs no one could safely extend. |
| Strategy | `IPaymentGateway`, `INotificationChannel` | Conditional branching on a gateway/channel enum scattered through the codebase — harder to test, harder to extend. |
| Adapter | Thin adapters translate Umbraco's Content Delivery API and the Apollo/Mongo mock into the same `Lakbay.Contracts` GraphQL shape | Letting each frontend query know which backend it's talking to — would leak backend-specific shapes into `Lakbay.Web`. |
| Domain events (Observer) | `BookingConfirmed` triggers email, SMS, and availability sync as independent subscribers | Calling all three inline inside the command handler — violates SRP and makes the handler's test have to know about email/SMS delivery to pass. |
| Specification | Deferred — flagged for catalog filtering (destination + theme + price + date) once facet complexity outgrows simple Examine queries | Building it now, before the filter requirements are real, would be premature abstraction. |

## Where this gets written down, per repo

- **Architecture Decision Records** — `docs/adr/ADR-000N-*.md` here in
  `Lakbay.Docs`, not duplicated per repo. A significant decision (a new
  pattern, a reversed earlier call) gets a numbered ADR at the moment it's
  made: Context → Decision → Alternatives considered → Consequences.
  Superseded decisions get a new ADR that says so — the old one is never
  silently deleted.
- **Per-repo `Docs/DEVELOPER_HANDBOOK.md`** — exact offline-runnable local
  setup and 2–3 worked "adding a new X" walkthroughs (add a payment
  gateway, add a product line, add a GraphQL field), turning the tables
  above into instructions rather than trivia. Seeded per repo once Phase 0
  scaffolding proves the setup steps actually work — not written from
  memory.
- **PR review gate** — any PR introducing a new abstraction or pattern
  includes a one-paragraph design-rationale note, checked against this
  guide before approval.
- **Comments stay rare** — code stays self-explanatory through naming;
  inline comments are reserved for genuinely non-obvious "why" (a
  workaround, a subtle invariant). The rationale for a pattern itself
  lives here and in the ADRs, not repeated at every call site.

## Full worked ADR examples

See [docs/adr/](adr/) — currently:

- [ADR-0001](adr/ADR-0001-unify-ecms-pcms.md) — Unify ECMS and PCMS into a
  single Umbraco solution
- [ADR-0002](adr/ADR-0002-cqrs-booking.md) — Use CQRS instead of a shared
  booking service class
- [ADR-0003](adr/ADR-0003-separate-booking-data.md) — Keep transactional
  booking data out of the Umbraco database
- [ADR-0004](adr/ADR-0004-mockapi-dotnet-not-node.md) — `Lakbay.MockApi`
  is .NET (HotChocolate), not Node.js/Apollo
- [ADR-0005](adr/ADR-0005-local-sql-server-not-azure-sql.md) — SQL Server
  in Docker for local dev; Azure SQL Database only in the live environment

Note on the tables above: since ADR-0004, `Lakbay.MockApi` is .NET like
`Lakbay.Cms` and `Lakbay.Booking`, so the same Repository-pattern and
Dependency-Inversion rows apply there too, not just to the two production
repos — worth reusing the same `IProductCatalogRepository`-shaped
abstraction rather than inventing a parallel one, if/when that
duplication is noticed during Phase 1.
