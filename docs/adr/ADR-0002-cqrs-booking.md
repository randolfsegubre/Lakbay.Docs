# ADR-0002: Use CQRS (via MediatR) instead of a shared booking service class

- **Status:** Accepted
- **Date:** 2026-08-29
- **Repo(s) affected:** `Lakbay.Booking`

## Context

Booking reads (availability lookups, price quotes) and booking writes
(place order, confirm, cancel) have different validation rules, different
performance characteristics, and different testing needs. A prior personal
project, `E-Commerse.AI.API`, used a single service-class shape for
similar responsibilities and its controllers ended up as disconnected
stubs with hardcoded fake data — MediatR and EF Core were referenced but
never registered in DI, so nothing was actually wired together, and no
test caught it because the fat service class made "does this even work
end to end" hard to answer with a small, focused test.

## Decision

Separate MediatR Commands (`PlaceBookingCommand`, `ConfirmBookingCommand`,
`CancelBookingCommand`) from Queries (`GetAvailabilityQuery`,
`GetPriceQuoteQuery`) in `Lakbay.Booking`. Each has its own handler and its
own test. Side effects of a successful command (sending a confirmation
email/SMS, syncing availability) are triggered via domain events
(`BookingConfirmed`) consumed by independent subscribers, not called
inline from the command handler.

## Alternatives considered

- **One `BookingService` class with public methods for everything** —
  rejected on direct evidence: this is the shape that left
  `E-Commerse.AI.API`'s controllers untestable and disconnected. A test
  for "confirming a booking" would have had to also know about email/SMS
  delivery to pass, discouraging anyone from writing it.
- **Calling side effects inline inside the command handler** (skip domain
  events) — rejected: violates Single Responsibility (a "confirm booking"
  handler doing confirmation *and* notification *and* availability sync)
  and makes the handler's own unit test carry unrelated concerns.

## Consequences

- More files (one handler per command/query) than a single service class
  would have, but each one has a single, obvious responsibility and a
  single, obvious test — matching the SRP row of
  `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`.
- Requires MediatR (or an equivalent in-process mediator) as a dependency
  — a small, well-understood library, not a framework-level commitment.
- The DI-wiring failure mode from `E-Commerse.AI.API` is explicitly called
  out in `Lakbay.Booking`'s Phase 0/4 exit criteria: proving the DI graph
  resolves and handlers are actually invoked end to end, not just that the
  classes compile.
