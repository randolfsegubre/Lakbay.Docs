# ADR-0026: Agent-assisted bookings use deferred payment, not a gateway — unblocks real Booking code without PayMongo

- **Status:** Accepted
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.Booking`, `Lakbay.AgentOps`

## Context

`Lakbay.Booking` Phase 4 (`02_BUILD_PLAN.md`) has been entirely unbuilt
since Phase 0, blocked on a PayMongo sandbox account that doesn't exist
yet — see `04_TASKS.md`. The Agent Channel (ADR-0021) needs `Lakbay.Booking`
to do something real: "browse availability, then hang up without
actually booking anything" would make `Lakbay.AgentDesktop` a read-only
catalog viewer, not the "book the customer directly" tool it's meant to
be, and would leave Phase 4 exactly as unbuilt as before this work
started.

The resolution is a genuine product distinction, not a workaround dressed
up as one: **phone bookings taken by a call-center agent commonly don't
collect payment during the call at all.** The agent confirms availability
and creates a reservation; payment is collected afterward — a follow-up
payment link, a bank transfer the customer arranges, or an invoice for a
travel-agency-style booking. This is how real phone-based travel bookings
routinely work, independent of whether an online payment gateway exists.
It means the Agent Channel can exercise real `Lakbay.Booking` domain
logic — the part that actually matters (correct availability handling,
ADR-0011) — without needing PayMongo at all.

## Decision

`Booking` gains a `Channel` field: `Online` (the original Phase 4 plan,
still blocked on PayMongo) or `Agent` (new, unblocked now).

- `ConfirmBookingCommandHandler` (ADR-0002, CQRS) takes a `Channel`
  parameter. The atomic availability decrement (ADR-0011) is **identical
  for both channels** — this is the correctness-critical part, and there
  is no reason for it to differ by how the booking was taken.
- For `Channel = Agent`: `PaymentStatus` is set to `PendingInvoice`
  instead of going through `IPaymentGateway` (the interface Phase 4 was
  always going to need per the Architecture & Patterns Guide's Strategy
  pattern table) — no gateway call happens at all for this channel.
  `Lakbay.AgentOps`'s Hangfire jobs (ADR-0025) handle sending the
  customer a payment-collection follow-up (email/SMS, matching Phase 4's
  original Twilio plan) asynchronously after the call ends.
- For `Channel = Online`: unchanged from the original Phase 4 plan —
  still routes through `IPaymentGateway`/PayMongo, still blocked until
  that sandbox account exists. This ADR does not attempt to unblock the
  online channel; it only adds a second, independently-unblockable path
  to the same domain.
- `BookingConfirmed`/`AvailabilityChanged` are published identically
  regardless of channel — `Lakbay.AvailabilityApi.Sync` and `Lakbay.Web`
  don't need to know or care which channel produced a booking.

## Alternatives considered

- **A fake/mock `IPaymentGateway` implementation standing in for
  PayMongo** — considered and rejected (this was one of the options
  presented and not the one chosen): it would make the flow look
  end-to-end complete when it isn't, and "confirmed via a fake payment
  that doesn't represent a real product decision" is a worse thing to
  have to explain later than an honestly-named deferred-payment channel
  that reflects how phone bookings actually work.
- **Wait for a real PayMongo account before building any Booking code** —
  rejected: this would leave Phase 4 exactly as unbuilt as it's been
  since Phase 0, and leaves the Agent Channel unable to do its one core
  job. The deferred-payment channel is real functionality on its own
  merits, not a stand-in for the online flow.
- **Two entirely separate booking domains** (one for online, one for
  agent-assisted) — rejected: the availability/concurrency correctness
  guarantee (ADR-0011) must be identical and shared regardless of channel,
  and splitting the domain risks the two paths drifting apart on exactly
  the part that must not drift.

## Consequences

- `02_BUILD_PLAN.md` Phase 4's exit criteria need a companion Phase 7
  (Agent Channel) exit criterion: an agent-assisted booking succeeds
  end-to-end (browse → confirm → `PendingInvoice` → follow-up job
  queued), independently of whether the online/PayMongo path is ever
  unblocked.
- `Lakbay.Booking`'s concurrency test (ADR-0011) must run for both
  channels, or at minimum be channel-agnostic at the point it actually
  exercises the atomic decrement, so the deferred-payment path is not
  accidentally exempted from the platform's core correctness guarantee.
- When PayMongo eventually unblocks the online channel, `Channel = Agent`
  bookings are unaffected — the two paths only ever shared the decrement
  logic, not the payment logic, so finishing one does not require
  touching the other.
