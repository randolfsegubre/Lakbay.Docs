# ADR-0004: Lakbay.MockApi is .NET (HotChocolate), not Node.js/Apollo

- **Status:** Accepted — supersedes the Node.js verdict for `Lakbay.MockApi` in the Lakbay Blueprint's technology-stack table
- **Date:** 2026-09-05
- **Repo(s) affected:** `Lakbay.MockApi`

## Context

The original plan (Lakbay Blueprint, technology-stack decisions) scoped
Node.js specifically to `Lakbay.MockApi`, carrying forward the
Sphinx-API/Mantincore precedent as-is: a Node.js + Apollo Server + MongoDB
service, kept deliberately separate in stack from the .NET production
backend. On review, that was the one place the plan introduced a second
language/runtime for no requirement-driven reason — `Lakbay.Contracts`,
`Lakbay.Cms`, and `Lakbay.Booking` are all .NET; only `Lakbay.Web` has a
genuine reason to be a different stack (it's a browser-facing Next.js
app).

## Decision

Build `Lakbay.MockApi` as an ASP.NET Core service using **HotChocolate**
(the standard .NET GraphQL server) with the `HotChocolate.Data.MongoDb`
package, against **MongoDB.Driver** (the official .NET MongoDB driver) —
same MongoDB database as originally planned, different language serving
it.

## Alternatives considered

- **Keep Node.js + Apollo Server, as originally planned** — rejected: the
  only reason to introduce a second backend runtime was fidelity to the
  Sphinx-API/Mantincore precedent, not a technical requirement. A small
  team maintaining `Lakbay.Cms`, `Lakbay.Booking`, and `Lakbay.Contracts`
  in .NET gains nothing from also maintaining a Node/Apollo toolchain for
  a service that exists only to be temporarily swapped out (see
  `02_BUILD_PLAN.md` Phase 3's exit criteria).
- **Drop `Lakbay.MockApi` entirely, build `Lakbay.Cms`/`Lakbay.Booking`
  first** — rejected: this was already considered and rejected when the
  platform was first planned. `Lakbay.Web` still needs to be built and
  tested without waiting on Umbraco/booking infrastructure to exist; a
  same-language mock is strictly better than no mock, not a reason to
  remove it.

## Consequences

- One backend language across `Lakbay.Cms`, `Lakbay.Booking`,
  `Lakbay.Contracts` (C# types), and now `Lakbay.MockApi` — only
  `Lakbay.Web` stays a different stack, and that split is genuinely
  requirement-driven (browser runtime).
- `Lakbay.Contracts`' generated-types story simplifies slightly: the C#
  types it generates now serve three consumers instead of two; the
  TypeScript types still serve only `Lakbay.Web`.
- The schema-diff CI check between `Lakbay.MockApi` and
  `Lakbay.Contracts` (see `02_BUILD_PLAN.md` Phase 1) is unaffected by
  this change — it checks the served GraphQL schema, not the language
  that serves it.
- Updates every place the Blueprint and `Lakbay.Docs` described
  `Lakbay.MockApi` as Node.js: `01_CLAUDE.md`'s repo map,
  `02_BUILD_PLAN.md`'s Phase 0/1 sections, `Lakbay.MockApi/CLAUDE.md` and
  `README.md`, the published Lakbay Blueprint artifact, and the Lakbay
  System Map diagram — all updated alongside this ADR, not left to drift.
