# ADR-0006: Lakbay.Cms is headless — zero Razor/UI code; Lakbay.Web owns all presentation

- **Status:** Accepted — formalizes a decision stated in prose in the
  Lakbay Blueprint and `Lakbay.Web/CLAUDE.md` since 2026-08-29; written up
  as its own ADR 2026-09-05 after direct questioning surfaced it was never
  given one.
- **Date:** 2026-09-05 (decision itself dates to the original Blueprint)
- **Repo(s) affected:** `Lakbay.Cms`, `Lakbay.Web`

## Context

The enterprise precedent this platform is modeled on ran ECMS (Umbraco)
and Prototype (React) as a *hybrid*: Razor views and page templates lived
in ECMS, with React components embedded inside those Razor views to
render interactive parts of the page. In practice this meant ECMS ended up
owning the majority of the UI/presentation code — page structure, layout,
routing — while Prototype's React served more as a component library
consumed by ECMS's pages than as an independent application. This created
real friction: SEO, hydration, and caching all had to negotiate between
two rendering models (Razor's server-rendered HTML and React's
client-side hydration) in the same request.

Randolf asked directly (2026-09-05) whether Lakbay would repeat this
shape, prompted by Next.js/TypeScript sounding — on the surface — like it
could be layered the same way ECMS layered React inside Razor.

## Decision

`Lakbay.Cms` (Umbraco) never renders a public-facing page. It has **no
Razor views for the storefront** and returns only structured data —
Umbraco's Content Delivery API plus a GraphQL layer matching
`Lakbay.Contracts` — to callers. `Lakbay.Web` (Next.js) is the *only*
place page templates, layout, routing, and UI code exist; it owns 100% of
rendering, using SSR/ISR for SEO-critical pages.

Umbraco's Razor view engine and backoffice-adjacent templating still exist
in `Lakbay.Cms` in the narrow sense that Umbraco itself uses Razor for its
own admin backoffice UI — that's Umbraco's own tooling, not this project's
code, and is out of scope for this decision.

## Alternatives considered

- **Repeat the ECMS/Prototype hybrid** (Razor page shells in `Lakbay.Cms`,
  React embedded for interactive regions) — rejected: this is the exact
  shape that produced the SEO/hydration friction the Blueprint's frontend
  section calls out. It also means two teams (or one team switching
  contexts) maintaining two rendering pipelines for one page.
- **Server-render everything from Umbraco, no client-side React at all**
  — rejected: loses Next.js's component model, RTK Query's caching, and
  the ability to build/test `Lakbay.Web` against `Lakbay.MockApi` before
  `Lakbay.Cms` exists (see `02_BUILD_PLAN.md` Phase 2) — that decoupling
  is only possible because `Lakbay.Web` doesn't depend on Umbraco's
  rendering pipeline at all, only on the shared GraphQL contract.

## Consequences

- `Lakbay.Cms`'s Phase 3 exit criterion — repointing `Lakbay.Web` from
  `Lakbay.MockApi` to `Lakbay.Cms` with zero frontend code changes — is
  only possible *because* `Lakbay.Cms` never had page-rendering
  responsibility to begin with. If this ADR were ever reversed, that exit
  criterion would need rewriting too.
- Content editors work entirely through Umbraco's backoffice (structured
  fields, not WYSIWYG page layout) — there is no "preview the actual
  rendered page inside Umbraco" experience the way a Razor-hosted CMS
  would give them. `Lakbay.Web` needs its own preview/draft-mode story
  (a Next.js concern, not an Umbraco one) if that gap matters before
  launch.
- Any future contributor asking "should this UI code go in Cms or Web" has
  one answer: Web, always. `Lakbay.Cms` should never grow a `Views/`
  folder for public pages — if one appears, that's a sign this ADR is
  being silently reversed and needs a new ADR, not a quiet PR.
