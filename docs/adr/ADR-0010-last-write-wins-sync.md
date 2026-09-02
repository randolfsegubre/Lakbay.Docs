# ADR-0010: Last-write-wins ordering guard on synced availability/catalog data

- **Status:** Accepted
- **Date:** 2026-09-06
- **Repo(s) affected:** `Lakbay.AvailabilityApi.Sync` (the Azure Function
  from ADR-0009)

## Context

Randolf asked directly whether the cache should "update every time when a
record has been updated, or if record from db is newer than the cache."
The second half is the correct instinct: Azure Service Bus does not
guarantee strict in-order delivery across a topic/subscription unless
messages are explicitly grouped into sessions (not planned here — see
Consequences). Two `AvailabilityChanged` events for the same listing sent
close together could theoretically be *processed* out of order (network
retry, competing consumer instances under the Function's own
auto-scaling). Blindly applying whatever arrives last, in receipt order,
risks writing a stale value over a newer one.

## Decision

Every document `Lakbay.AvailabilityApi` stores carries a
`sourceUpdatedUtc` timestamp (or an incrementing version, whichever the
source system — `Lakbay.Cms` or `Lakbay.Booking` — can produce more
reliably) copied from the source event, not generated at write time.
`Lakbay.AvailabilityApi.Sync` compares this against the currently stored
value before writing:

```
if (incoming.sourceUpdatedUtc > stored.sourceUpdatedUtc) apply the update;
else discard it — a newer or equal value is already stored.
```

This is applied as a single atomic MongoDB `findOneAndUpdate` with the
timestamp comparison in the query filter (not a separate read-then-write),
so two Function instances processing events concurrently can't race each
other into the same wrong outcome the double-booking scenario in
ADR-0011 illustrates for a different repo.

## Alternatives considered

- **Trust receipt order** — rejected for the reason above: not
  guaranteed, and the failure mode (a sold-out listing incorrectly shown
  as available again because a late-arriving stale event overwrote the
  correct state) is exactly the kind of bug this whole real-time effort
  exists to prevent.
- **Service Bus sessions**, forcing strict per-listing ordering at the
  broker level — a valid alternative, not chosen now because it adds
  operational complexity (session-aware consumers, session state
  management) to solve the same problem the timestamp guard solves more
  simply. Worth revisiting only if the timestamp approach proves
  insufficient under real load.

## Consequences

- `Lakbay.Cms` and `Lakbay.Booking` must both include a reliable, source-
  generated timestamp (or version) on every event they publish — a small,
  explicit contract addition to `Lakbay.Contracts` or the event schema
  itself, not assumed to already exist.
- Discarding a stale event is silent by design (it's the correct
  behavior, not an error) but should be logged/counted for observability
  — worth surfacing in Application Insights so a consistently high
  discard rate is visible as a signal something upstream is misbehaving.
- This does not require Service Bus sessions or any broker-level ordering
  guarantee, keeping `Lakbay.Booking`'s and `Lakbay.Cms`'s publishing code
  simple — they just publish events as they occur.
