# ADR-0011: Prevent double-booking with an atomic, conditional decrement — not caching, not events

- **Status:** Accepted
- **Date:** 2026-09-06
- **Repo(s) affected:** `Lakbay.Booking`

## Context

Randolf named the actual goal plainly: "there is an accommodation where
2 users booked, 1 was confirmed and the other was a split seconds also
book, we don't want for them to be duplicate." This is a different
problem from everything in ADR-0008/0009/0010, and it's important that it
stay recognized as different: those ADRs are about the *read side*
(search results, the storefront UI) staying current with what
`Lakbay.Booking` already knows. None of them touch what happens the
instant two confirm-booking requests for the same last slot arrive
milliseconds apart at `Lakbay.Booking` itself — no amount of fast
propagation to `Lakbay.AvailabilityApi`/`Lakbay.Web` prevents that race,
because both requests can already be in flight *before* either
propagation event is even published.

The classic bug shape here is read-then-write: a handler reads
"available count = 1," decides the booking is allowed, then writes
"available count = 0" — and a second concurrent request can read the same
"1" before the first request's write lands, so both proceed.

## Decision

`Lakbay.Booking`'s `ConfirmBookingCommandHandler` never does read-then-write
for the availability check. It issues a single atomic, conditional UPDATE
against Azure SQL (or SQL Server locally, per ADR-0005) as part of
confirming the booking:

```sql
UPDATE AvailabilitySlots
SET AvailableCount = AvailableCount - 1
WHERE ProductId = @productId
  AND DateSlot = @dateSlot
  AND AvailableCount > 0;
```

The handler checks rows-affected:

- **1 row affected** → the decrement succeeded, this request legitimately
  holds the last (or one of the remaining) slot(s); proceed to create the
  booking record in the same transaction, commit, then publish
  `BookingConfirmed`/`AvailabilityChanged`.
- **0 rows affected** → nothing was available; reject this booking
  attempt with a clear "no longer available" result. No partial state, no
  compensating rollback needed, because nothing was written.

SQL Server's row-level locking on the `UPDATE` is what makes this safe
under concurrency: two simultaneous requests against the same row
serialize at the database — the second one's `UPDATE` physically cannot
proceed until the first's transaction commits or rolls back, and by the
time it does run, the `WHERE AvailableCount > 0` clause correctly
reflects whatever the first request left behind. This holds regardless of
how the two requests arrived (same instance, different instances, same
millisecond) — it is enforced by the database engine, not by application
code timing.

## Alternatives considered

- **Optimistic concurrency via EF Core `[Timestamp]`/rowversion**, retry
  on `DbUpdateConcurrencyException` — a legitimate alternative, not chosen
  as the primary mechanism because it adds a retry loop for a case
  (booking confirmation) that isn't a high-throughput hot path where
  optimistic-concurrency's "usually no conflict" assumption pays off;
  pessimistic locking via the conditional atomic `UPDATE` above is simpler
  to reason about and just as fast at realistic booking volumes. Worth
  revisiting only if profiling ever shows row-lock contention as a real
  bottleneck.
- **Application-level distributed lock** (e.g. a Redis lock per
  product/date) around a read-then-write check — rejected: adds an extra
  moving part and failure mode (lock acquisition timeout, lock not
  released cleanly) to solve a problem the database's own transactional
  guarantees already solve natively. Redis is already in the stack for
  caching (Blueprint), not proposed here as a locking primitive.
- **Rely on `Lakbay.AvailabilityApi`'s real-time push to prevent the
  second booking** (i.e., assume the UI disables the "book" button fast
  enough) — rejected outright, and worth stating explicitly: this is a
  UX nicety, not a correctness guarantee. Network latency alone makes it
  trivial to defeat, and it was never the mechanism ADR-0008 claimed to
  provide.

## Consequences

- The availability check and the decrement must be the same database
  statement (or at minimum the same transaction with an appropriate
  isolation level) — a future refactor that splits them back into
  separate read/write calls silently reintroduces the exact race this ADR
  closes. Worth a code-review checklist item, not just a one-time fix.
- `Lakbay.Booking`'s xUnit test suite needs a concurrency test — two
  simulated simultaneous confirm-booking calls against a slot with
  `AvailableCount = 1`, asserting exactly one succeeds — not just
  sequential-call tests, which cannot catch this class of bug.
- This is entirely independent of ADR-0008/0009/0010 — a reviewer should
  be able to point at this ADR and confirm double-booking is prevented
  without needing the real-time propagation work to exist or work
  correctly at all.
