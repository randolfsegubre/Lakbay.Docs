# ADR-0003: Keep transactional booking data out of the Umbraco database

- **Status:** Accepted
- **Date:** 2026-08-29
- **Repo(s) affected:** `Lakbay.Cms`, `Lakbay.Booking`

## Context

[ADR-0001](ADR-0001-unify-ecms-pcms.md) unifies editorial content and the
product catalog into one Umbraco database because both are read-heavy,
publish-then-cache workloads that Umbraco's NuCache is built for. Orders,
baskets, and live availability are the opposite profile: high-frequency,
low-latency, transactional writes, especially at checkout.

## Decision

`Lakbay.Booking` owns orders, baskets, and the availability calendar in
its **own** database (Azure SQL), entirely separate from `Lakbay.Cms`'s
database. The two services communicate over Azure Service Bus events
(e.g. `BookingConfirmed`, `AvailabilityChanged`) rather than sharing
tables or having one service query the other's database directly.

## Alternatives considered

- **Store bookings as a third Umbraco content tree, alongside Content and
  Products** — rejected: risks lock contention on the CMS database every
  time someone checks out, since NuCache's read-heavy design isn't tuned
  for that write pattern. Content nodes are also the wrong conceptual
  model for rapidly-changing transactional state (an availability count
  that changes on every booking isn't "content" in any meaningful sense).
- **One shared database, two logical schemas** — rejected: doesn't avoid
  the lock-contention risk (same physical database, same resource
  contention under load), only adds the illusion of separation.

## Consequences

- Two databases to operate instead of one, and cross-service consistency
  is eventual (via Service Bus events) rather than transactional — a
  content editor changing a product's availability-related copy and a
  customer completing a booking are not part of the same transaction.
  This is an accepted trade-off, not an oversight.
- The CMS stays fast under load regardless of booking volume, and the
  booking path can scale (and be redeployed, and fail) independently of
  content publishing.
- Any future change that would merge these back together (e.g. for
  simplicity at very low volume) needs its own ADR explicitly reversing
  this one, not a quiet schema migration.
