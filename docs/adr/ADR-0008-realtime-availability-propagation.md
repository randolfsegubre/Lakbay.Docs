# ADR-0008: Real-time availability propagation via Service Bus → SearchApi → SignalR

- **Status:** Accepted
- **Date:** 2026-09-06
- **Repo(s) affected:** `Lakbay.Booking`, `Lakbay.SearchApi`, `Lakbay.Web`

## Context

Randolf named a gap directly observed in Inghams/Hotelplan: when an
accommodation sold out, that fact reached the search backend eventually,
but nothing pushed the change out to a browser already showing that
listing — a shopper could keep seeing a sold-out item until they
refreshed. He wants Lakbay to actually close this gap, but didn't yet
know the mechanism.

Two things are easy to conflate here and shouldn't be:

1. **Staleness of the search read-model** — `Lakbay.SearchApi` showing
   availability that `Lakbay.Booking` no longer agrees with. This is a
   data-propagation problem.
2. **Overselling** — two shoppers both completing checkout for the last
   available slot. This is a concurrency-control problem inside
   `Lakbay.Booking` itself, at the moment a booking is confirmed, and is
   unaffected by how fast the UI updates elsewhere. This ADR does not
   address it; `Lakbay.Booking`'s `ConfirmBookingCommandHandler` must
   check current availability transactionally regardless of how "live"
   the storefront looks.

This ADR is scoped to (1) only.

## Decision

Three-hop propagation, using infrastructure already in the stack (Service
Bus, SignalR) rather than introducing a new mechanism:

1. **`Lakbay.Booking`** publishes `AvailabilityChanged` (product/holiday
   ID, new availability state) to Azure Service Bus the moment a booking
   is confirmed — this was already planned in ADR-0003/Phase 4, now given
   an explicit consumer.
2. **`Lakbay.SearchApi`** subscribes to `AvailabilityChanged` (alongside
   its existing subscription to `Lakbay.Cms`'s publish-sync events — see
   ADR-0007) and updates the matching MongoDB document's availability
   field immediately. This is the same "keep the read model current"
   responsibility ADR-0007 already gave this repo, applied to a second
   event source.
3. **`Lakbay.SearchApi`** then pushes a small change notification (just
   the affected product ID, not the full record) to Azure SignalR
   Service, scoped to a group per listing/product page.
   **`Lakbay.Web`** subscribes to that group while a listing/catalog page
   is open and, on receiving a notification, re-fetches just that one
   item through the normal RTK Query path (`api.util.invalidateTags` on
   the specific product) rather than a full page reload or a blanket
   refetch of the whole listing.

`Lakbay.SearchApi` is the single place that both consumes availability
changes and originates the real-time push — it already owns "what does
the current read-model say," so it's also the only correct place to
decide "and therefore what should connected browsers be told."

## Alternatives considered

- **Polling** (`Lakbay.Web` re-queries `Lakbay.SearchApi` on an interval)
  — rejected as the primary mechanism: real availability changes are
  bursty (a popular listing selling out during a promotion) and rare
  otherwise, so a fixed poll interval is either wasteful most of the time
  or too slow exactly when it matters. Worth keeping as a *fallback* for
  browsers where the SignalR connection drops, not as the main path.
- **GraphQL subscriptions directly from `Lakbay.SearchApi`** (HotChocolate
  supports this natively) instead of SignalR — a reasonable alternative,
  rejected mainly because Azure SignalR Service was already adopted in
  the Blueprint's stack decisions specifically for "live availability
  updates," and introducing a second real-time transport (GraphQL-WS)
  alongside it would be the redundant-tooling mistake the Blueprint's
  stack table otherwise avoided everywhere else. Revisit only if
  GraphQL subscriptions turn out to simplify the client noticeably once
  Phase 4 is actually built.
- **`Lakbay.Booking` pushes to SignalR directly**, skipping `Lakbay.SearchApi`
  — rejected: `Lakbay.Booking` doesn't know the denormalized shape
  `Lakbay.Web` actually displays (that's `Lakbay.SearchApi`'s job); having
  two services independently decide what to tell the browser risks them
  disagreeing about the current state.

## Consequences

- `Lakbay.SearchApi` gains a second Service Bus subscription and an
  outbound SignalR dependency — real, new scope for Phase 4, not
  something Phase 1's simpler "sync from Cms" work already covers.
- `Lakbay.Web` needs a SignalR client connection and a narrow
  invalidate-one-item cache-update path (not a full listing refetch) —
  flagged as explicit Phase 4 scope in `02_BUILD_PLAN.md`.
- This does not solve overselling — that remains `Lakbay.Booking`'s job,
  and needs its own explicit concurrency-control design (e.g. optimistic
  concurrency on the availability count) when Phase 4 is built, called
  out separately so it isn't assumed solved by this ADR.
- If SignalR connection volume or cost ever becomes a real constraint,
  revisit the "push to everyone viewing a listing" scope — e.g. narrowing
  to only the specific date/room combination rather than the whole
  listing — with real usage data, not upfront.
