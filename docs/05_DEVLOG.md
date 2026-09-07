# Devlog

Reverse-chronological. One entry per session (or per phase boundary within
a long session) that changed a plan, wrote code, or made a decision.
Format: date, what was asked, what changed and why, what's next.

---

## 2026-09-07 — Portfolio dev-completion pass: secret found and fixed, builds re-verified, new Docker blocker hit

**Asked:** as part of a wider pass bringing several personal-project
portfolios to "dev complete, verified E2E locally" for job applications,
verify Lakbay's actual current state and close small gaps — not re-derive
architecture, just confirm what's real.

**What was found:** all five application repos except `Lakbay.Booking` had
substantial uncommitted work sitting in their working trees — everything
`04_TASKS.md`/this devlog's entries (7) through (14) already describe (the
full Phase 3 sync pipeline, Country/Region/Accommodation hierarchy, Room
Types, Stays search, Activities marketplace) was real and present on disk,
just never committed. Confirmed with the user before touching git state;
committed all five repos' working trees locally (no push) so this work is
no longer at risk of loss.

**A real secret was caught before it went further:** `Lakbay.Cms`'s
`appsettings.json` had a live Umbraco Imaging `HMACSecretKey` value
committed in the very commit just being made. Cleared it back to an empty
string and moved the real value to `dotnet user-secrets` instead (the
project already had a `UserSecretsId` provisioned, it just wasn't being
used for this key) — fixed by amending that one still-local, unpushed
commit, not by adding a follow-up "oops" commit.

**Build/test verification (no Docker needed):** `Lakbay.Contracts` (both
C# and — implicitly, via the committed `generated/types.ts` — TypeScript
sides), `Lakbay.Booking` (`dotnet test` — 1 passed), and
`Lakbay.AvailabilityApi` all build clean. `Lakbay.Cms` builds with only 3
pre-existing nullable-reference warnings. `Lakbay.Web` builds and lints
clean (`npm run build` — all 9 routes compiled, including the new
`/stays` and `/activities` pages).

**New Docker blocker, different from the onboarding one this devlog's
earlier "Docker unblocked by machine restart" entry describes:** Docker
Desktop's backend now crashes ~10-15 seconds after every launch with
`starting services: initializing Ingest server: ... rename
.../run/sailor-ingest.sock .../run/sailor-ingest.sock.stale: The file
cannot be accessed by the system.` Both the live and `.stale` socket files
are orphaned NTFS reparse points that resist deletion via `Remove-Item`,
`cmd /c del`, and `fsutil reparsepoint delete` alike (all three report the
same "cannot be accessed by the system" error) — consistent with a handle
orphaned by an earlier unclean shutdown. Given this exact class of Docker
Desktop startup failure was fixed by a plain machine restart last time
(see the entry below, "Docker unblocked by machine restart"), that's the
likely fix again, but restarting the machine wasn't done unilaterally in
an active session — left for the user to do when convenient, then resume
from "Full E2E verification, still pending" in `04_TASKS.md`.

**Not touched:** no application code changed, no ADRs added — this was a
verification/hygiene pass, not new feature work. `Lakbay.AvailabilityApi.Tests`
couldn't be run (needs Docker for its Testcontainers-backed MongoDB
fixture) — expected to pass per the existing devlog record of 8/8 green,
not verified fresh this session.

**Next:** once Docker is confirmed working again (`docker ps` returns a
table), follow `07_MANUAL_SETUP_GUIDE.md` §4-7 top to bottom in one sitting
to re-prove the full Cms -> Service Bus -> Sync -> MongoDB ->
AvailabilityApi -> Web pipe live, the way session (5)/(7) originally did —
that's the one thing this session couldn't re-verify.

---

## 2026-09-06 (15) — Competitive landscape research added to the Blueprint

**Asked:** whether any website is similar to Lakbay, why the platform is
called "Philippines-first," and to fold both into the Blueprint artifact
and the docs rather than just answering in chat.

**What changed:**

- Live web research (not assumed): the closest existing comparable is
  **Guide to the Philippines** ("Philippines' biggest travel marketplace"
  — PH-only curated packages, installment payments, island/heritage/
  highland categories) — structurally a marketplace aggregating
  third-party operators, versus Lakbay's MVP as a single
  vertically-integrated tour operator (the Inghams/Hotelplan shape this
  plan already commits to). Also checked: Exploring Tourism
  Philippines/TraveloPhilippines (independently arrived at a near-identical
  "Highland to Island" split — corroborates the Alon/Amihan cluster
  segmentation), WayPH.com (flagged "Questionable" by a third-party trust
  scorer, 50.6/100 — a caution, not a template), and global marketplaces
  (Klook, GetYourGuide, Agoda, Traveloka) which aren't Philippines-first by
  definition.
- Added a new "Competitive Landscape" section to the Blueprint artifact
  (between Market research and Product line strategy), plus new entries in
  its Research Sources list. Mirrored the summary into `01_CLAUDE.md`
  section 1, along with an explicit "first, not only" clarification that
  was previously implicit (stated in the Blueprint's market section but
  not surfaced as its own callout).
- Confirmed: nothing in the existing four-cluster (Alon/Amihan/Parul/Pamana)
  curatorial structure exists elsewhere — that segmentation looks like a
  real differentiator, not a gap copied from an incumbent.

**Next:** none of this changes architecture or roadmap — it's grounding
documentation only. Revisit if the business-model decision (tour operator
vs. marketplace, still open per the Blueprint's Open Risks) gets made,
since it would change how directly Lakbay competes with Guide to the
Philippines specifically.

---

## 2026-09-08 (14) — Accommodation types, room photos, long-stay pricing, and a fair-price Activities marketplace (ADR-0020)

**Asked:** four related gaps in one message, right after ADR-0019
shipped — Room Types had no photos; every accommodation read as
"hotel-shaped" when real destinations offer apartments/Airbnbs/etc.;
nothing supported foreigners staying indefinitely with no fixed return
date; and no way to book activities independently at a fair price
instead of buying from a local and risking a scam or a mark-up. Asked to
research what accommodation types these destinations actually offer
before designing anything.

**What changed:**

- Researched live before building: real Philippine Department of
  Tourism-accredited accommodation categories (Hotel, Resort, Apartel,
  Pension House) plus what's actually listed on Airbnb in these exact
  destinations (Hostel, Homestay, Vacation Rental); real long-stay
  pricing patterns (30-50% off nightly rate for a monthly stay, no fixed
  end date, PH visa extensions up to 36 months); the real Klook/
  GetYourGuide fair-price model (fixed price agreed upfront, every
  inclusion itemized in writing).
- Three scope questions confirmed via `AskUserQuestion` before writing
  code: the DOT-based `AccommodationType` category list; long-stay
  modeled as a monthly rate on individual Room Types, not an
  Accommodation-level flag; a full new top-level `Activity` entity
  across all 14 destinations, not a smaller upgrade to the existing
  text-only `Destination.optionalAddOns`.
- `RoomType` gained `heroImageUrl` (reuses the owning Accommodation's own
  real photo — never a fabricated distinct-room interior, extending
  ADR-0018's photo-honesty rule one level deeper) and `monthlyRatePhp`
  (nullable, seeded only in 5 destinations where a longer stay is
  realistic: Baguio, Cebu, Siargao, La Union, Boracay).
- `Accommodation` gained `type: AccommodationType!` (a new 7-value enum)
  — all 14 existing accommodations reassigned a real,
  description-grounded category (e.g. Coron Bayside Inn → Pension
  House, Calle Crisologo Heritage House → Vacation Rental, Session Road
  Pine House → Apartel, reframed for Baguio's real remote-work
  reputation) rather than left defaulting to Hotel.
- New `Activity` entity across all four repos (Contracts, Cms, Sync,
  AvailabilityApi, Web) — 14 real activities seeded, one per
  destination, each with a realistic fixed PHP price (₱500-1,900/person)
  and itemized inclusions grounded in the research (e.g. Coron's
  "Ultimate Island Hopping Tour," ₱1,800, full-day, includes licensed
  boatman, snorkeling gear, lunch, entrance fees, life jackets). Embeds
  a full `Destination` object (like Product/Accommodation), not a thin
  id reference (unlike RoomType) — Activities are independently
  cross-destination browsable.
- New `Lakbay.Web` `/activities` route (destination filter, client-side
  over the full list, same pattern `/stays` uses) plus "Activities near
  your stay" / "Book activities here" teaser sections on the
  Accommodation detail and Destination pages — the real answer to "give
  them flexibility, no rigid package schedule": pick a room, then add
  fixed-price activities on your own terms; `Product` stays available as
  a secondary "ready-made package" option, unchanged in meaning.
- Deliberately avoided repeating ADR-0019's HotChocolate `String`-vs-`ID`
  trap on the new `activities(destinationId)` field — declared `String`
  in the SDL from the start this time, matching what a plain C#
  `string?` parameter actually serves.
- **A real orphaned-generation bug found, a new variant of a pattern
  seen four times before**: the established `$dateTrunc`-at-hour-
  granularity cleanup check silently merged two genuinely different
  reseed generations (this session's ADR-0019 reseed and this ADR-0020
  one) into a single bucket, because both happened to run within the
  same clock hour — the check reported "one generation, no orphans" when
  28 accommodation documents (14 stale, silently defaulting to `HOTEL`
  since they predated the `Type` field entirely) actually existed in
  Mongo. Fixed by re-running the grouping at minute granularity, which
  correctly separated the two generations for a clean targeted
  `deleteMany`. Minute-level grouping is now the safer default for this
  check going forward, not hour-level.
- **A background task the user had spawned from the ADR-0019 session's
  flagged follow-up landed mid-session**: it closed out the
  `ProductFilter.destinationId` SDL/runtime mismatch (changed the SDL
  from `ID` to `String`, matching the real server type — see its own
  devlog entry immediately below), correctly detected this session's
  in-flight `Accommodation.Type` build break, and deliberately left it
  alone as someone else's in-progress work. Verified after the fact that
  both changes coexist cleanly in `schema/lakbay.graphql` and the
  regenerated TypeScript types — no lost update, confirmed by rebuilding
  `Lakbay.Contracts` and re-running `tsc --noEmit` clean.
- Same discipline as every prior ADR this session: `dotnet build`/`test`
  clean (8/8), `tsc --noEmit` clean, all three long-running processes
  restarted in order before reseeding, and a full real-browser pass
  confirming room photos, the long-stay callout, the new
  accommodation-type filter on `/stays`, and the Activities marketplace
  all render and link correctly — including the exact Boracay scenario
  from ADR-0019 now showing a real photo on both room cards, a
  ₱43,000/month long-stay option on the Garden View Room, and a linked
  "Sunset Sailing & Island Hopping" activity at a fixed ₱1,200.

**What's next:** dedicated `/regions`/`/countries` browsing pages remain
out of scope (still true as of ADR-0017). Phase 4 (`Lakbay.Booking`)
still blocked on a PayMongo sandbox account.

## 2026-09-08 (13) — ProductFilter.destinationId SDL/runtime mismatch closed out

**Asked:** the ADR-0019 session flagged `ProductFilter.destinationId` as
carrying the identical latent `ID`-vs-`String` mismatch that had just been
fixed for the new `accommodations(destinationId)`/`roomTypes
(accommodationId)` arguments — HotChocolate infers a plain C# `string?`
member as GraphQL `String`, not `ID` (only a member literally named `Id`
gets `ID` by convention), so the authored SDL's `destinationId: ID` never
matched what `Lakbay.AvailabilityApi`'s live server actually served. Asked
to close this out for real: change the SDL (matching the direction already
chosen for the two new fields, over annotating the C# resolver with
`[HotChocolate.Types.ID]` to make the server actually serve `ID`).

**What changed:**

- `schema/lakbay.graphql`'s `ProductFilter.destinationId` changed from `ID`
  to `String`, matching the real runtime type. `Lakbay.Contracts/csharp/
  ProductFilter.cs`'s `DestinationId` was already a plain `string?` and the
  HotChocolate-side `ProductFilterInputType` only overrides the input
  type's *name* (to `ProductFilter`, not HotChocolate's default
  `ProductFilterInput`) — it doesn't touch field types — so no C# change
  was needed on the API side; this was purely a documentation-vs-reality
  fix, exactly like the accommodations/roomTypes one.
- Regenerated `Lakbay.Contracts/typescript/generated/types.ts`
  (`npm run codegen`) — all three `destinationId`/`accommodationId` filter
  fields now agree on `Scalars['String']`.
- **Live GraphQL introspection could not be run to confirm this against a
  running server**: `Lakbay.AvailabilityApi.Api` currently fails to
  compile for an unrelated reason — uncommitted, in-progress work adding
  `Accommodation.Type` (a required member, presumably ADR-0020's real
  Philippine accommodation categories) hasn't yet updated
  `CatalogSeeder.cs`'s four seeded `Accommodation` object initializers to
  set it, so `dotnet build` fails there with `CS9035` regardless of this
  change. Left untouched — it's someone else's in-flight edit, not this
  fix's to finish, and guessing seed data (what `AccommodationType` each
  of the four real seeded inns "should" be) isn't this session's call to
  make. Verified instead by reading `Query.cs`/`ProductFilterInputType.cs`
  directly: nothing sets `[HotChocolate.Types.ID]` or otherwise overrides
  `destinationId`'s inferred type, so the server was already serving
  `String` before and after this change — only the SDL moved to match it.
  A live introspection pass is still worth doing once the unrelated build
  break is resolved.

**What's next:** the `CatalogSeeder.cs`/`Accommodation.Type` build break
found along the way is real and blocks `dotnet build`/`dotnet run` for
`Lakbay.AvailabilityApi.Api` right now — flagged for whoever's mid-flight
on that ADR-0020 work, not fixed here.

## 2026-09-08 (12) — Room Types, destination-level perks, and a Stays search (ADR-0019)

**Asked:** a concrete scenario — search for "a budget-friendly hotel in
Boracay for a family holiday," find its room options, add activities
around that stay. Wanted to know how the website would handle it, and
what real accommodation search/browsing should look like — closer to how
Inghams actually sells holidays than how Lakbay currently worked.

**What changed:**

- Verified the real Inghams "Room Types" pattern live (Hotel Post, St
  Anton) before building anything: per-room size/bed-configuration/
  occupancy/price, plus a resort-wide "Included in your ski holiday"
  perks list and optional promotional add-ons shown on the hotel page but
  identical across that resort's hotels.
- Two scope questions confirmed with `AskUserQuestion` before writing
  code: shared perks move to `Destination` (Product keeps its own
  curated-itinerary activities separately — a deliberate, confirmed
  split, not an oversight), and a new cross-destination `/stays` search
  page gets built, not just a per-destination upgrade.
- New `RoomType` entity across all four repos, synced as its own
  top-level entity carrying a plain `accommodationId` reference rather
  than an embedded object — same "queried separately by parent id" shape
  `ProductFilter.destinationId` already used for Product/Destination, not
  a new pattern. 28 real room types seeded (2 per Accommodation).
- `Accommodation` gained `destination` (previously only reachable
  indirectly via a Product picking both as siblings — a real, necessary
  structural addition for the Stays page to work at all), `tags`, and
  `officialRating`. `Destination` gained `includedPerks`/`optionalAddOns`.
- New `Lakbay.Web` routes: `/stays` (destination + tag filters, both
  client-side over the full ~14-item list, matching the collection
  pages' existing filtering convention) and `/stays/[accommodationId]`
  (photo, tags, rating, all Room Types via a new shared `RoomTypeList`
  component, the Destination's perks/add-ons, and a "book as a
  ready-made package" section listing any linked Products). The existing
  Destination page was updated, not replaced — its "Where you'll stay"
  section now also shows Room Types, and "Holidays here" was retitled
  "Or book as a ready-made package," matching the repositioning.
- **A real, pre-existing schema gap found live in the browser, not by any
  build/test step**: HotChocolate serves a plain C# `string?` parameter
  as GraphQL `String`, not `ID` — only a member literally named `Id` gets
  `ID` by convention. The authored SDL had declared
  `accommodations(destinationId: ID...)`/`roomTypes(accommodationId:
  ID!)`; a client query with `$destinationId: ID` against the real
  `String`-typed argument failed GraphQL's variable-usage compatibility
  check outright — caught via the browser's network tab, since `curl`
  testing with inline literals (as most of this session's manual checks
  do) never exercises that specific check. Fixed by matching the SDL and
  the hand-written `Lakbay.Web` queries to the real runtime type.
  `ProductFilter.destinationId` carries the identical latent mismatch and
  predates this session — left alone, since it's never actually
  triggered (always passed inside an object literal, never a bare
  `$var: ID`), but worth a dedicated look if a raw-variable `destinationId`
  argument is ever added elsewhere.
- Same migration/verification discipline as every prior ADR this
  session: a self-limiting stale-shape check (`accommodation` missing
  `destination`) triggered a real delete-and-recreate; all three
  long-running processes (Cms, `Lakbay.AvailabilityApi.Api`, the Sync
  Function host) stopped and restarted in order before reseeding;
  `dotnet build`/`test` clean (8/8), `npx tsc --noEmit` clean; one round
  of orphaned pre-reseed Mongo documents (the same class of issue as
  every prior reseed) cleaned up via a targeted `deleteMany`, confirmed
  via a `$dateTrunc`-grouped count first; a full real-browser pass
  through the exact scenario asked about — `/stays`, filtered to Boracay
  + Family-Friendly, landing on exactly Station 2 Beachfront Inn, its two
  real room options (Garden View Room ₱2,200/night sleeps 2, Beachfront
  Family Room ₱3,800/night sleeps 4), Boracay's destination-wide perks/
  add-ons, and a link to the existing "Boracay White Beach Getaway"
  Product as a secondary package — plus regression checks on the
  Destination page and the existing holiday detail page.
- Mid-session, the user separately asked whether describing Lakbay as
  "Philippines-first" amounted to claiming it's the first holiday-booking
  platform of any kind. Confirmed it doesn't: the phrase is a market-scope
  term (the Philippines is the initial/primary market), matching a
  "mobile-first" usage, not a first-mover claim — nothing in the docs
  positions Lakbay as unprecedented, and real competitors (Klook,
  Traveloka, Agoda, and others) were never in question.

**What's next:** dedicated `/regions`/`/countries` browsing pages remain
out of scope (still true as of ADR-0017). The `ProductFilter.destinationId`
SDL/runtime type mismatch flagged above is a small, real, currently-inert
bug worth a standalone fix if it's ever exercised through a raw GraphQL
variable. Phase 4 (`Lakbay.Booking`) still blocked on a PayMongo sandbox
account.

## 2026-09-08 (10) — Developer Atlas checked into the repo as `docs/index.html`

**Asked:** fix a table-text-overlap bug the user spotted (via DevTools) in
the published "Lakbay Developer Atlas" Artifact, then bring the loose
local copy (`D:\_DEV\Personal_Projects\Lakbay\_qa_preview.html`, sitting
outside all six git repos, untracked) up to date and into `Lakbay.Docs`
so it ships with the GitHub repo.

**What changed:**

- **The table-overlap bug**, found first: the feature-map/troubleshooting
  tables had no `table-layout:fixed`, so the browser sized columns by
  content rather than the `<th style="width:%">` hints, and a long
  unbroken token (a file path, a camelCase identifier) had nothing to
  make it wrap — it overflowed its column, which read as overlapping
  text. Fixed with `table-layout:fixed` on `table` and `overflow-wrap:
  anywhere` on every text-bearing cell/element (`tbody td`, `.file`,
  `.fn`, `code.inline`). Also dropped a vestigial `position:sticky` on
  `thead th` that had no benefit in this non-independently-scrolling
  wrapper and was a second plausible contributor to the same symptom.
- **Content refresh**, since the page hadn't been touched since before
  the rename/destinations/accommodation sessions: the four product-line
  cards now say Islands/Highlands/Festivals/Heritage (not Alon/Amihan/
  Parul/Pamana) with their real current destinations; ADR-0015 and
  ADR-0016 added to the decision-records list; the "where things stand"
  callout, topbar status pill, and holiday-detail feature-map row updated
  to mention the 14-product catalog and the accommodation/activities
  fields; two new feature-map rows for `Product.cs`'s new fields and
  `CatalogPublishSyncHandler.ParseActivityLines`.
- **Moved into the repo, not just re-published as an Artifact**: the fixed/
  refreshed page is now `Lakbay.Docs/docs/index.html`, rebuilt as a
  genuinely standalone document rather than a straight copy of the
  Artifact's raw HTML — the Artifact platform injects its own iframe
  runtime script and serves Mermaid from an internal `/_runtime/` path
  that only resolves inside claude.ai; neither works once the file is
  opened locally or served by GitHub Pages. Rebuilt with a proper
  `<!DOCTYPE html>`/`<head>`/`<body>` structure and Mermaid loaded from a
  public CDN (`cdn.jsdelivr.net`, pinned to the same 11.16.1), initialized
  once against the page's own fixed dark palette rather than the
  Artifact runtime's dynamic light/dark detection — this page was already
  single-theme by design (see the CSS's own comment), so that detection
  logic was dead weight once outside the Artifact viewer. Verified by
  serving `Lakbay.Docs/docs/` locally (`npx serve`) and checking in a
  real browser: layout, the fixed tables, and both Mermaid diagrams all
  render, zero console errors.
- **The loose, untracked `_qa_preview.html` at the Lakbay root was
  deleted** — it predated this repo home and would otherwise be a second,
  silently-diverging copy of the same content. `Lakbay.Docs/CLAUDE.md`
  now points at `docs/index.html`, described plainly as a manually
  regenerated snapshot, not a second source of truth — the markdown docs
  win on any conflict between the two.

**Also answered, not a code change:** whether GitHub has a
Confluence-equivalent — see the conversation for the full comparison
(GitHub Wiki, GitHub Pages, and the plain-markdown-in-repo approach this
project already uses, versus real third-party options like Confluence
itself, Notion, or GitBook). No action taken here; recorded as an open
question the user can decide, not a call for a session to make
unilaterally.

---

## 2026-09-08 (11) — Nested Content tree (ADR-0018) + accommodation prominence

**Asked:** "Yes, please do that" (extend the Content tree to nest
Region/Destination pages, matching ECMS's real depth — proposed at the
end of the prior session). In the same message: "I still don't see
accommodation or something a like in our website. I want to see how it
would look." Mid-implementation, a follow-up: "I don't see Accommodations
like Hotels, Inn... In Inghams, those are the types of accommodation
shown. That is what they sell and they market... double check Inghams and
Inntravel. I might be wrong."

**Verified the accommodation visibility complaint first**: confirmed live
(GraphQL response + rendered page text) that the "Where you'll stay" card
on a holiday detail page was genuinely working — the real issue was that
it only appeared there, nowhere else, and as text only, no photo. Pointed
the user at their own running dev server (`http://localhost:3000/holidays/
coron-island-hopping-3d2n`) to see it directly rather than trying to hand
over a screenshot.

**Used `EnterPlanMode` for the Content-tree work**: explored the current
`ContentTreeSeeder.cs`/`cmsContentApi.ts`, designed nested Region/
Destination pages matching ECMS's confirmed real depth (from the prior
session's live-database research), and got explicit approval before
touching code.

**Mid-implementation, re-verified the business-model question live**
rather than assuming either the user or the existing design was right.
Browsed Inghams' real search results: every result *is* a named,
individually-priced hotel ("Hotel Serre Palas, Les 2 Alpes — From
£696pp, 7 nights, Bed & Breakfast, flight from London Gatwick included")
— accommodation genuinely is the primary sellable unit there. Then
Inntravel's real holiday page ("Timeless Tuscany"): the sellable unit is
the *holiday*, but its one accommodation gets a full dedicated tab
("Overview | Itinerary | **Accommodation** | Extend your stay | Prices &
travel | Reviews") with a real hotel name, gallery, description. Lakbay's
own positioning — curated packages, not a hotel marketplace — matches
Inntravel's shape, not Inghams'. Conclusion reported back to the user
directly: the ADR-0017 data model (`Product` owns one `Accommodation`)
was already correct; the real gap was presentation weight, not structure.

**What changed:**

- `Region`/`Destination` gained `slug: String!` (`Lakbay.Contracts`,
  mirroring `Product.slug`) — the stable join key between the Products
  tree's flat data and the new nested Content-tree pages, same "plain
  text, not a picker" reasoning `productLineCode` already established
  (no ordering dependency between independently-booting seeders).
- `Lakbay.Cms`: two new content types, `regionLandingPage` and
  `destinationLandingPage`, created with **real Umbraco parent-child
  nesting** this time (`Home → ProductLine landing → Region landing →
  Destination landing`) — deliberately different from the Products
  tree's flat + Content-Picker shape (ADR-0017), matching how ECMS
  (nests) and PCMS (doesn't) really differ. 12 Region pages + 14
  Destination pages seeded, copy adapted from the Products tree's own
  descriptions/highlights but written fresh and shorter (the editorial
  overlay, not a duplicate of the structured data).
- `cmsContentApi.ts` gained `getRegionPageContent`/
  `getDestinationPageContent` — filtering by content type across the
  whole tree and matching by `slug` client-side, since (unlike Home)
  there's no single well-known parent ID a client could hardcode when
  Region/Destination pages nest under a *different* parent per line.
- Two new routes: `collections/[code]/[regionSlug]/page.tsx` (region
  highlights + a grid of its destinations) and
  `collections/[code]/[regionSlug]/[destinationSlug]/page.tsx` — the
  latter giving accommodation a full, unmissable section (photo, name,
  description, highlights), directly answering the visibility complaint.
  `collections/[code]/page.tsx` gained an "Explore by region" grid,
  additive to the existing flat product listing.
- The existing holiday detail page's accommodation card was upgraded to
  match — a two-column layout with the accommodation's own photo, not
  text-only.
- **No fabricated accommodation photography, stated as an explicit
  design rule**: `Accommodation.heroImageUrl` reuses the same real
  Wikimedia photo already sourced for that destination's Product —
  accommodation names on this platform are invented placeholders
  ("Coron Bayside Inn" isn't a real business, ADR-0016), so captioning a
  real photo as if it depicts that specific building would be
  misleading; reusing a genuine photo of the place is honest.
- Same migration/verification discipline as every ADR this session:
  self-limiting stale-shape check (`regionLandingPage` missing) →
  delete-and-recreate; all three long-running processes stopped and
  restarted in order; `dotnet build`/`test` clean (8/8); `tsc --noEmit`
  clean; orphaned Mongo generation cleaned up (targeted `deleteMany`,
  confirmed via `$dateTrunc` grouping first); verified live via GraphQL,
  the Content Delivery API (`route.path:
  /islands-landing-page/palawan-landing-page/`, confirming real nesting),
  and a full browser pass through Islands → Palawan → Coron.

**Next:** dedicated top-level `/regions`/`/countries` index pages
(browsing outside any one product line) remain out of scope. Phase 4
(`Lakbay.Booking`) remains blocked on a PayMongo sandbox account.

---

## 2026-09-08 (10) — Country → Region → Destination → Accommodation hierarchy (ADR-0017)

**Asked:** "The content tree doesn't look like ECMS or PCMS at all. If you
remember, in ECMS, we have Product (Ski Holidays) under it are Countries,
under it are Regions, under it are resorts, under it are accommodations.
We want to have the same structure so details of location can be
highlighted... but in a way that will fit our business model."

**Grounding, before designing anything:** dispatched an Explore agent to
read the real ECMS repo, now available locally at `D:\_DEV\HPUK\
Hotelplan.Inghams.V2.CMS` (read-only reference, per the user's own
instruction last turn — never pushed to). Confirmed the real tree —
`Product → Geography → Country → Region → Resort → Accommodation`, built
via Umbraco Compositions and duplicated per product line — plus the
crucial finding: ECMS's actual "highlights" content (descriptions,
features, images, ratings) doesn't live in Umbraco fields at all, it
lives in a separate external Product CMS joined by a `{level}Code`
property. Lakbay's own ADR-0001 already rejected that two-system split
(ECMS+PCMS unified for cost reasons), so the adaptation was: same tree
shape, highlights as real Umbraco fields throughout, not a second system.

Given the size (a cross-repo schema redesign with real judgment calls,
not a bug fix), used `EnterPlanMode` for the first time this session —
explored the current schema, wrote a full plan to
`C:\Users\devra\.claude\plans\rustling-munching-dream.md`, and used
`AskUserQuestion` mid-plan to confirm two open decisions before writing
any code: **Country is one shared "Philippines" node**, not duplicated
per ProductLine the way ECMS's real multi-country scale does it (Lakbay
is one country today); and **this pass enriches the existing holiday
detail page** rather than also building dedicated `/regions/[id]`-style
browsing pages (an explicit next step, not silently dropped).

**What changed:**

- New `Country`, `Region`, `Accommodation` GraphQL types (nested-object
  embedding, matching how `Product.destination` already worked) —
  `Destination.region` went from a plain string to a full `Region!`;
  `Product.accommodationName`/`accommodationDescription` (two flat
  strings, ADR-0016) became a single `accommodation: Accommodation`
  reference — resolving the "revisit only if..." case ADR-0016 explicitly
  flagged.
- `Lakbay.Cms`: three new content types (`country`, `region`,
  `accommodation`) via the existing `CmsSchemaBuilder`; `destination`'s
  `region` property converted from Textstring to ContentPicker,
  `product`'s two accommodation strings replaced with a ContentPicker.
  `CatalogPublishSyncHandler` gained `BuildCountry`/`BuildRegion`/
  `BuildAccommodation`, reusing the already-generic `ParseActivityLines`
  helper for highlights (no new parsing mechanism). Full seed-data
  rewrite: 1 Country, 12 Regions, 14 Destinations (now parented under
  Region via picker), 14 Accommodations (promoted off the old flat
  strings), 14 Products (referencing Accommodation via picker).
- **A real, pre-existing data quality issue fixed along the way**: the
  original 14 destinations mixed province-level names ("Palawan",
  "Ilocos Sur") with proper regional groupings ("Western Visayas",
  "Cordillera Administrative Region") for the same `region` field — a
  flat string let that inconsistency slip in silently. The new Region
  entity forces one consistent, traveler-recognizable granularity,
  applied uniformly.
- `Lakbay.AvailabilityApi`: three new Mongo collections/class maps
  (`countries`, `regions`, `accommodations`), three new `Apply*Async`
  sync handlers (same upsert-if-newer shape as every existing one), and
  `country`/`regions` GraphQL resolvers (confirmed again: HotChocolate
  needs no `AddType<>` for a plain nested C# record).
- `Lakbay.Web`: nested GraphQL fragments for the new levels; the holiday
  detail page gained a `Philippines → Region → Destination` breadcrumb
  with the region's highlights (and the country's, visually
  de-emphasized) shown inline — the actual "details of location
  highlighted" ask — and the accommodation card now reads from real
  content instead of two flat strings, with its own highlights list.
- **Applied a lesson instead of rediscovering it**: stopped all three
  long-running processes (Cms, `Lakbay.AvailabilityApi.Api`, *and* the
  Sync Azure Function host) before rebuilding this time, proactively —
  ADR-0016's rollout found the hard way that forgetting the Sync host
  leaves it silently running stale code against a new schema.
- **Same migration discipline as ADR-0015/0016**: a self-limiting
  stale-shape check (Region content type missing) triggers a full
  delete-and-recreate of the Products-tree types — no in-place data
  transform, since this is still disposable pre-launch dev data. One
  round of orphaned Mongo documents from the recreate cycle, cleaned up
  with the same targeted `deleteMany`-on-stale-generation approach used
  every time this session.
- **Verified live**: `dotnet build` clean across `Lakbay.Contracts`,
  `Lakbay.Cms.Web`, and the full `Lakbay.AvailabilityApi` solution;
  `dotnet test` 8/8 green (test-only `CatalogSeeder.cs` fixture rewritten
  for parity with the real shape); `tsc --noEmit` clean; a direct GraphQL
  query returning the full nested chain
  (`product.accommodation`/`product.destination.region.country`); and a
  real browser pass on a holiday detail page showing the breadcrumb,
  region highlights, country highlights, and the accommodation card.

**Next:** dedicated region/country browsing pages are the natural
follow-up (the GraphQL queries already exist) but were explicitly scoped
out of this pass. Phase 4 (`Lakbay.Booking`) remains blocked on a
PayMongo sandbox account.

---

## 2026-09-08 (9) — Accommodation + activities on Product (ADR-0016), 2 new surf offers, brand-scope decision

**Asked:** "Add products/offers with Surfing activities. I also notice
that in the collections, it is just showing the place... Check these
HotelPlan websites [Inghams, Inntravel, Santa's Lapland]... Maybe you
can get ideas... Not sure if I should include other holidays in Asia
because that was the initial plan but since you've already named this
Lakbay..." Two decisions were confirmed via AskUserQuestion before any
code changed: keep "Lakbay," stay Philippines-only for now; extend
`Product` with new fields rather than a bigger separate-Accommodation
entity or no schema change at all.

**Research, done before writing any code:** browsed all three sites the
user named. The concrete, reusable pattern: Inghams shows "Properties
available: 13 · From £1,049pp · View accommodation" on every destination
card — the accommodation is the product, not scenery behind it. Santa's
Lapland spells out an included-vs-optional split explicitly: the base
package includes a husky ride, reindeer sleigh, tobogganing; a separate
"Fancy more snow fun?" section sells Northern Lights excursions,
snowmobile safaris, and a longer husky ride as their own add-on cards.
Full writeup and the reasoning behind every schema choice in
[ADR-0016](adr/ADR-0016-accommodation-and-activities-on-product.md).

**What changed:**

- `Product` gained `accommodationName`, `accommodationDescription`,
  `includedActivities: [String!]!`, `optionalActivities: [String!]!` —
  propagated across `Lakbay.Contracts` (SDL, C# record, regenerated TS
  types via `npm run codegen`), `Lakbay.Cms` (new Textstring/Textarea
  properties on the Product content type, real seed data for all 14
  products, `CatalogPublishSyncHandler`'s new `ParseActivityLines`
  newline-splitter), `Lakbay.AvailabilityApi.Sync` (four new `.Set(...)`
  calls in `ApplyProductAsync`), and `Lakbay.Web` (`availabilityApi.ts`'s
  shared GraphQL fragment, a new "Where you'll stay" + two-column
  included/optional-activities section on the holiday detail page, and a
  "Stay: X" line added to every collection-page card).
- **Two new surfing products**, the user's explicit ask, under Islands
  (ALON): Siargao Surf Camp (Cloud 9 — the country's surf capital) and
  La Union Surf Weekend (San Juan — the accessible beginner surf town
  three hours from Manila). Both real, both well-known, each with a
  verified Wikimedia Commons image. Islands is now a 5-product line;
  total catalog is 14.
- **Real gap #1**: the seeder's `if (!typesAlreadyExist)` guard only
  creates Umbraco content types once, on a genuinely empty schema — on
  this machine `ProductLine` already existed from earlier sessions, so
  the guard skipped the whole type-creation block and the Product
  content type never got the new properties. First reseed attempt
  crashed: `SetValue("accommodationName", ...)` threw "No PropertyType
  exists with the supplied alias." Fixed with a one-time migration
  matching ADR-0015's own established shape — gated on a specific tell
  (`accommodationName` missing from the existing type), deletes and
  recreates all three Products-tree types + content, self-limiting since
  the tell is permanently false afterward.
- **Real gap #2, found only by reading the actually-synced data**: after
  fixing gap #1, `accommodationName` still came back `null` over
  GraphQL. The Sync Azure Function host (`func start`) had been running
  since before this session's code changes and was silently still
  executing the *old* `CatalogSyncFunction.cs` build — it had no
  `.Set(p => p.AccommodationName, ...)` in its update chain yet. Fixed
  by stopping the stale `func`/worker processes and restarting from the
  rebuilt project. Lesson stated plainly: a schema change needs every
  long-running local process that touches it restarted — Cms and the
  query API are the two usually remembered; the Sync function host is
  easy to forget precisely because it's started once and left running.
- **Real gap #3, mechanical consequence of #1 and #2**: each
  crash-then-fix cycle recreated Products-tree content with fresh
  Umbraco UDIs, and the Sync function upserts Mongo by that UDI — so
  every recreation orphaned the previous generation's Mongo documents
  (three generations accumulated: 12+14+14 destinations, 12+14 products,
  confirmed via a `$dateTrunc`-grouped count before touching anything).
  Cleaned up twice, same disciplined approach as the 2026-09-08 (8)
  entry: a targeted `deleteMany` on the pre-cutoff generation, never a
  collection drop, verified by document count before and after each time.
- **Brand/scope decision, made explicit rather than assumed**: Lakbay
  stays Philippines-only under its current name. "Lakbay" means
  "journey" in Filipino, not literally "Philippines" — the word itself
  doesn't block a future non-PH destination if that's ever decided later.
  Recorded in ADR-0016 rather than left as an unrecorded verbal call.
- **Verified live**: `dotnet build` clean across `Lakbay.Contracts`,
  `Lakbay.Cms.Web`, and the full `Lakbay.AvailabilityApi` solution;
  `dotnet test` still 8/8 green; `tsc --noEmit` clean on `Lakbay.Web`;
  then a real browser pass — the Siargao product's detail page (hero,
  "Where you'll stay," "What's included"/"Optional extras" two-column
  list) and the Islands collection page showing all 5 products each with
  a "Stay: X" line, screenshotted end-to-end through the real
  Cms → Service Bus → Sync → MongoDB → AvailabilityApi → Web pipeline.

**Next:** Phase 4 (`Lakbay.Booking`) remains blocked on a PayMongo
sandbox account. The accommodation/activities work is deliberately
geography-agnostic, so nothing here needs rework if the Philippines-only
decision is ever revisited.

---

## 2026-09-08 (8) — Renamed the four product lines to English, expanded the catalog to 12 destinations

**Asked:** "Can we rename everything in English. Our target audience not
just Philippines but foreigners as well. Also add more content. Add
content for most popular holiday destinations in the Philippines." A
scope-clarifying question confirmed: rename only the four product lines
(Alon/Amihan/Parul/Pamana); "Lakbay" stays the platform's brand name.

**What changed:**

- **Rename, done at the code level, not just content.** `Lakbay.Web`'s
  nav bar, collection headings, and product "Back to X" link were all
  deriving their text directly from the GraphQL enum code string
  (`code[0] + code.slice(1).toLowerCase()`) — a real finding, since it
  meant renaming Cms content alone would have left the nav bar showing
  "Alon" forever. Added `PRODUCT_LINE_META[code].label` in
  `lib/catalog.ts` as the one place English display names live, and
  swapped all four call sites to read it. GraphQL enum codes
  (`ALON`/`AMIHAN`/`PARUL`/`PAMANA`) deliberately untouched — the schema
  already states codes are the stable identifier to key logic off, never
  the display name.
- **Cms side:** `CatalogContentTypeSeeder`'s ProductLine `Name` fields
  renamed to English (Islands/Highlands/Festivals/Heritage).
  `ContentTreeSeeder`'s landing-page `Name` fields got a `" Landing
  Page"` suffix rather than reusing the same plain English name —
  giving both trees an identically-named node would have recreated the
  exact URL-routing collision the previous session (2026-09-08 (7))
  had just found and worked around.
- **Catalog expanded 4 → 12 destinations/products**, three per product
  line instead of one: Islands adds Boracay and El Nido; Highlands adds
  Sagada and Tagaytay; Festivals adds Cebu (Sinulog Festival) and Iloilo
  (Dinagyang Festival); Heritage adds Banaue (Ifugao Rice Terraces) and
  Intramuros. Each destination is real, independently well-known, and
  paired with a real Wikimedia Commons photo — every image URL verified
  resolvable with `curl -I -L` (Wikimedia requires a real User-Agent or
  returns 429) before use, same discipline as the previous session.
- **A real bug found in the reseed flow, not anticipated going in:**
  `LAKBAY_FORCE_RESEED_CATALOG=true` deletes and recreates every Cms
  content node, which get fresh Umbraco UDIs on recreation.
  `CatalogSyncFunction` upserts Mongo documents keyed by that UDI, so
  the *old* Mongo documents from the previous seeding session were never
  cleaned up — GraphQL queries came back with each of the original 4
  products/destinations duplicated (old doc + new doc, same slug/name,
  different Mongo `_id`). Confirmed via direct Mongo inspection
  (`SourceUpdatedUtc` cleanly separated the stale docs from this
  session's), then removed with a targeted `deleteMany` on the
  before-today's-reseed timestamp — never a collection drop, matching
  the standing instruction to keep local dev actions secure/deliberate
  even though this data is disposable. The underlying gap (reseed
  doesn't garbage-collect orphaned synced documents) is *not* fixed at
  the code level — it's a rough edge specific to the
  `LAKBAY_FORCE_RESEED_CATALOG` dev-only escape hatch, which a real
  editor flow can never trigger (a backoffice edit never changes a
  node's UDI), so left as a known limitation rather than engineered
  around.
- **Verified live**, not just via API responses: nav bar (all 4 English
  labels), homepage collection cards, all four `/collections/[code]`
  pages (Islands correctly showing Coron/Boracay/El Nido; Heritage
  showing Vigan/Banaue with real images), and a product detail page's
  "← Back to Islands" link — screenshotted in a real browser against the
  locally running `Lakbay.Cms` → Service Bus → Sync → MongoDB →
  `Lakbay.AvailabilityApi` → `Lakbay.Web` pipeline.

**Also true, unrelated to this session's work, surfaced only because
these are now real synced products:** every product shows "Sold out" —
`AvailableCount` is 0 for everything, because Cms never sets that field
(ADR-0014: it's `Lakbay.Booking`'s field to own, and Phase 4 isn't built
yet). Not a regression, not in scope here — just newly visible now that
the full real pipeline is what's rendering the page, not Phase 1's
hand-seeded fixture data.

**Next:** Phase 4 (`Lakbay.Booking`) remains blocked on a PayMongo
sandbox account.

---

## 2026-09-08 (7) — Phase 3 completed: Content tree live, real Block List attempted and pivoted, a real URL collision found and fixed

**Asked:** continue and complete Phase 3, and add real content with real
images sourced online.

**What changed:**

- `Lakbay.Cms`: `ContentTreeSeeder` creates a Home Page and one landing
  page per product line, seeded with real copy and real Wikimedia
  Commons photos (each URL verified resolvable), published the same
  programmatic-authoring way the Products tree already is.
  `CmsSchemaBuilder` extracted from `CatalogContentTypeSeeder` so both
  seeders share the same code-first document-type/data-type logic
  instead of a second copy.
- **A real Umbraco Block List was built first, not skipped**: two element
  types, a Block List data type (`DataType.ConfigurationData` hand-built
  against the reflected `BlockListConfiguration` shape), and real content
  authored by hand-constructing the stored value to match
  `BlockListValue`'s C# model. The stored JSON was structurally sound —
  confirmed directly in SQL Server — but Umbraco's own Content Delivery
  API silently returned zero items reading it back. Two rounds of
  reflection against the real assemblies didn't surface why within this
  session's time budget. **Pivoted to ADR-0015**: `sections` is
  JSON-in-Textarea, the same already-proven pattern `Product.priceBands`
  uses — ADR-0012's actual intent (a `type` string maps to one React
  component) is unaffected, only the storage mechanism changed. Stated
  plainly in the ADR, not hidden.
- **A real, previously-missing piece found enabling the Delivery API**:
  `Program.cs`'s Umbraco builder chain never called `.AddDeliveryApi()`
  — enabling it via config alone left its DI services unregistered,
  throwing on first request. Added, plus a CORS allow-list matching
  `Lakbay.AvailabilityApi`'s own pattern, since `Lakbay.Web` calls it
  cross-origin.
- `Lakbay.Web`: a real block registry (`BlockRegistry.tsx`, ADR-0012)
  and a new `cmsContentApi` RTK Query slice, following the exact pattern
  `availabilityApi` already established. Homepage and all four
  `/collections/[code]` pages now show real Cms hero imagery and copy
  above the existing product grids, falling back to static copy if Cms
  isn't reachable.
- **A real URL-routing collision found live**: the Products tree's
  `ProductLine` nodes and the Content tree's landing pages share names
  ("Alon", etc.), so Umbraco's route resolution collided both onto
  `/alon` — and served the wrong one (the taxonomy record, not the
  landing page) with no error, just wrong data silently rendering. Fixed
  by fetching Home's children and matching by `productLineCode`
  client-side instead of a direct route lookup — a real fix for now, not
  a proper one; the real fix is giving each tree its own container node,
  left open rather than rushed.
- Verified in a real browser across the homepage and all four
  collections — hero images, headings, and the "on the ground" sections
  all rendering, with the existing product grids untouched underneath.

**What's next:** the GraphQL-on-Umbraco package decision (still open —
the Delivery API is a working interim path, not a resolution of that
specific question), a CI skeleton, the two-trees-share-a-namespace fix,
or Phase 4 once a PayMongo sandbox account exists.

---

## 2026-09-08 (6) — Real photos added; Phase 1/Phase 3 duplication resolved; a real test-isolation bug found and fixed

**Asked, same session as (5) above:** whether Phase 3 was "done" and
testable end to end (answered: mid-flight, blocked on Docker Desktop's
first-run dialog until Randolf clicked through it), and to add real
product photography ("just use images available online").

**What changed:**

- Four real Wikimedia Commons photos (Coron/Kayangan Lake, Baguio, the
  Giant Lantern Festival, Vigan/Calle Crisologo — each URL verified
  resolvable before use) added to `Lakbay.AvailabilityApi.Api`'s seed data
  and rendered in `Lakbay.Web` (collection cards, holiday detail hero) —
  confirmed in a real browser across all four collections.
- `Lakbay.Cms`'s `heroImage` field simplified from a Media Picker to a
  plain URL text field, matching "just use images available online" and
  `Lakbay.Contracts.Product.HeroImageUrl`'s own shape exactly.
- Docker Desktop's blocker resolved by Randolf (killed and relaunched it,
  skipped sign-in) — unblocked full live verification of everything
  designed in entry (5): all three Products-tree content types confirmed
  for real in SQL Server; real content published through the real Service
  Bus emulator into MongoDB; two more real bugs found only by actually
  publishing — `SaveAndPublish`'s `culturesToPublish` needed `[]`, not
  `null` or `["*"]`; and the `productLine` Content Picker was being read
  as raw picker text instead of resolved to the picked node's `code`.
- **Real architectural gap found and put to Randolf as a decision, not
  guessed:** Phase 1's hand-seeded data and Phase 3's Cms-authored data
  were coexisting in MongoDB as 8 products instead of 4, under different
  IDs for the same real-world holidays. Decided: `Lakbay.Cms` is now the
  sole source — `CatalogSeeder`'s automatic boot-time seed retired from
  `Program.cs`, kept only as a test-support utility; the 4 stale
  documents deleted from the real local database.
- **A real, previously-invisible bug found as a side effect of that
  retirement, not looked for:** rewriting `Lakbay.AvailabilityApi.Tests`
  to call `CatalogSeeder` explicitly (since `Program.cs` no longer does)
  surfaced that `Program.cs` had always read Mongo configuration *before*
  `WebApplicationFactory`'s per-test database override could apply — every
  test had silently been running against the real local MongoDB the whole
  time, not an isolated Testcontainers instance, since Phase 1. Fixed by
  binding lazily inside each DI factory delegate instead of capturing the
  setting once at top level. That fix immediately exposed a second latent
  bug it had been masking — three test names, once prefixed, exceeded
  MongoDB's 63-character database-name limit — fixed with a
  truncate-plus-hash helper. All 8 tests now pass against genuinely
  isolated data, for the first time.
- Also found live: a fresh Umbraco 18.1.1 install doesn't pre-create a
  data type for every built-in property editor (`Umbraco.Decimal` had
  none) — `CatalogContentTypeSeeder` now creates one on demand when
  missing, rather than assuming it exists.
- A Docker-startup diagnostic technique (checking
  `%APPDATA%\Docker\settings-store.json`'s `DisplayedOnboarding` flag, and
  using `wsl -d docker-desktop` to isolate WSL2-vs-Docker-Desktop issues)
  saved to memory for reuse if this happens again on any machine.

**What's next:** decide the deferred GraphQL-on-Umbraco package question,
build the Content tree (pages, Block List, ADR-0012's block registry), add
a CI skeleton, or move to Phase 4 once a PayMongo sandbox account exists.

## 2026-09-08 (5) — Phase 3: Cms sync pipeline designed, built, and proven live end-to-end; real photos added

**Asked:** continue from a prior session after a Claude Desktop reset
(confirmed nothing was actually lost — the memory system survived
intact). Picked Phase 3 over Phase 4 since Phase 4 needs a PayMongo
sandbox account this session doesn't have. Mid-session, also asked to add
real product/content photography ("just use images available online").

**What changed:**

- ADR-0013: Cms→AvailabilityApi sync goes over Service Bus, reusing
  `Lakbay.AvailabilityApi.Sync` (scaffolded empty since ADR-0009).
- ADR-0014: found while implementing, not asked for — ADR-0010's single-
  timestamp last-write-wins guard breaks the moment `Product` gets a
  second writer (Cms owns catalog fields, Booking will own
  `availableCount`, Phase 4). Fixed now with a second, internal-only
  ordering field, before Booking exists to be broken by it.
- `Lakbay.AvailabilityApi.Shared` extracted — the query API and the Sync
  function now share one Mongo access layer, matching what ADR-0009
  always intended but hadn't been built.
- `Lakbay.Cms`: `CatalogContentTypeSeeder` creates the Products tree
  (ProductLine, Destination, Product) code-first, and — in the same
  method, deliberately, since Umbraco runs same-notification handlers
  concurrently — seeds and publishes real content for it, standing in for
  backoffice authoring since no session has interactive login credentials
  (kept out of the repo and memory on purpose, even locally).
- **Verified genuinely live, not just compiled**, after Docker Desktop's
  first-run onboarding dialog (blocking since the previous session) was
  finally clicked through by Randolf: real SQL Server database now has
  all three content types with all 17 fields correctly typed; a real
  Umbraco 18.1.1 install doesn't pre-create a data type for every built-in
  editor (`Umbraco.Decimal` had none) — found and fixed with a
  create-on-demand fallback. Content published for real, flowed through
  the real Service Bus emulator (12 messages, 0 failures after fixes) into
  MongoDB, and came back out through `Lakbay.AvailabilityApi`'s GraphQL —
  the actual Phase 3 exit criterion.
- **Two more real bugs found only by actually publishing, not by reading
  the code:** (1) `SaveAndPublish`'s `culturesToPublish` parameter — tried
  `null` (ArgumentNullException), then `["*"]` ("wildcards or nulls are
  not allowed" — that message is misleading, it means "not for invariant
  content"), landed on `[]`. (2) `productLine` on Destination/Product is a
  Content Picker, not free text — the sync handler was parsing the raw
  picker UDI string as if it were a `ProductLineCode`, failing on every
  single node; fixed by resolving the picked ProductLine node first and
  reading *its* `code` field, the same way the `destination` picker was
  already being resolved.
- Real photos (Wikimedia Commons, license-appropriate for hotlinking,
  each URL verified resolvable before use) added to all four seeded
  products and rendered in `Lakbay.Web` — collection cards and the holiday
  detail hero — confirmed in a real browser across all four collections
  and the detail page.
- **A real architectural gap surfaced, left as an open decision, not
  guessed:** Phase 1's `CatalogSeeder` (hand-seeded data) and Phase 3's
  Cms-authored data now both exist in MongoDB under different IDs for the
  same real-world holidays — 8 products instead of 4. See `04_TASKS.md`'s
  "Blocked" section for the options; not resolved in this session.

**What's next:** decide the Phase 1/Phase 3 data-duplication question
above, then either finish Phase 3 (Content tree, Block List, the deferred
GraphQL-on-Umbraco package decision) or move to Phase 4 once a PayMongo
sandbox account exists.

---

## 2026-09-08 (4) — Phase 2: real Lakbay.Web storefront, verified in a browser, two real bugs found and fixed

**Asked:** "Continue up to the end. Include context and put some real
data or mock data but base on real world records and scenarios. I want to
see the end results." Interpreted as: build the next real, demoable
increment (Phase 2, the storefront) rather than attempting to fabricate a
finished Phase 4/5 (real payments, real deployment) that would need
external credentials and infrastructure this session doesn't have — and
prove it actually works by looking at it, not just by describing it.

**What changed:**

- `Lakbay.Web`: three real pages (`/`, `/collections/[code]`,
  `/holidays/[slug]`), typed RTK Query endpoints against
  `Lakbay.Contracts`' generated types, a shared `catalog.ts` formatting
  util (PHP currency, dates, board-basis labels), a site header/nav across
  the four product lines. Visual identity reuses the exact palette from
  the Lakbay Blueprint / System Map artifacts.
- **Two real bugs found by actually running it, not by reading the
  code:**
  1. Turbopack refused to resolve `@lakbay/contracts` (a sibling-repo
     `file:` dependency) even with the standard `transpilePackages` fix,
     because it infers `Lakbay.Web`'s own lockfile as the workspace
     boundary. Fixed with a `tsconfig.json` path alias *and* a widened
     `turbopack.root` — confirmed neither alone was sufficient by testing
     each in isolation.
  2. `Lakbay.AvailabilityApi`'s `products(filter: $filter)` broke the
     moment a real client (this storefront) sent `filter` as a proper
     GraphQL variable — every prior `curl` test had used an inline
     literal, which never exercises variable-type validation and so never
     caught that HotChocolate had named the type `ProductFilterInput`,
     not `ProductFilter` as the shared schema declares. Fixed with an
     explicit type descriptor in `Lakbay.AvailabilityApi`; added a
     regression test that specifically uses the real-variable path, since
     none of the existing 7 did.
  3. A smaller display bug ("Coron, Palawan, Palawan") caught in the
     browser screenshot itself — a destination whose name already
     includes its region was appending the region again. Fixed with a
     small `destinationLocation()` helper.
- **A repeated process-management slip, caught faster this time:**
  rebuilding `Lakbay.AvailabilityApi` after the fix failed with a file
  lock; the error named the exact locking PID, killed by that PID
  specifically (not by process name) — no collateral damage this time,
  unlike the `Lakbay.Cms` incident earlier this session.
- Booking is a real, visibly disabled button ("coming in Phase 4") —
  deliberately not a fake/stubbed checkout, since a real payment flow
  needs PayMongo credentials this session doesn't have.
- Verified live in the Browser pane: homepage renders all four product
  lines; `/collections/alon` and `/collections/pamana` render their real
  seeded products with correct pricing; `/holidays/coron-island-hopping-3d2n`
  renders full itinerary/board-basis/availability/price-band detail.
- Updated `Lakbay.Web`'s, `Lakbay.AvailabilityApi`'s, and
  `Lakbay.Contracts`' own handbooks with both real gotchas and a proven
  "adding a new page/catalog view" walkthrough (previously deferred).

**What's next:** Phase 3 (`Lakbay.Cms` content/product trees — the
`Lakbay.AvailabilityApi` sync trigger decision from `04_TASKS.md` becomes
relevant here) or Phase 4 (`Lakbay.Booking` — real checkout, replacing the
disabled button; needs a PayMongo sandbox account, which doesn't exist
yet). Full production deployment (Phase 5) needs a real Azure environment
and is out of scope for local development entirely.

---

## 2026-09-08 (3) — Phase 1: real Lakbay.AvailabilityApi resolvers against MongoDB

**Asked:** continue development, after fixing a gap in the setup guide
(it was missing tool-installation instructions — fixed first, see
`07_MANUAL_SETUP_GUIDE.md`'s own history).

**What changed:**

- Added `MongoDB.Driver` to `Lakbay.AvailabilityApi.Api` and
  `Testcontainers.MongoDb` to its test project.
- Wrote `MongoClassMaps` (maps `Lakbay.Contracts`' plain POCOs onto Mongo
  documents via an Adapter, without putting Mongo attributes on the
  shared Contracts types), `CatalogContext` (typed collection access),
  and `CatalogSeeder` (idempotent, real Philippine seed data).
- Rewrote `Query.cs`: `productLines`, `destinations(productLine:)`,
  `products(filter:)`, `product(slug:)` — matching
  `schema/lakbay.graphql` exactly. `products`' filter logic is hand-built
  from the explicit `ProductFilter` input (not HotChocolate's
  `[UseFiltering]`, which would auto-generate a different argument shape
  and break parity with `Lakbay.Contracts`) — price/date bounds combine
  into a single `PriceBands` `ElemMatch` so they're checked against the
  *same* band, not independently against any band.
- Added a `docker-compose.yml` for MongoDB (no auth, unlike SQL Server —
  nothing secret to protect locally).
- **Verified for real, twice over:** `dotnet test` (7 passed, against a
  real Testcontainers-managed MongoDB, not mocks) and manually via `curl`
  against a live running instance with real seeded data — including
  filtering (`AMIHAN` + `minPricePhp: 5000` correctly returned only the
  Baguio product at ₱6,800, excluding everything below the floor).
- **A real bug, caught and fixed via the live run, not just by
  inspection:** `productLines` initially failed with an opaque
  "Unexpected Execution Error." `Destination` and `Product` both map
  their `Id` property to Mongo's `_id`; `ProductLine` has no `Id`
  property (its key is the `Code` enum), so it was left to Mongo's
  auto-assigned `_id` — which then had no matching C# member during
  deserialization, and `BsonClassMap` throws on unmapped document fields
  by default. Fixed with `SetIgnoreExtraElements(true)` on `ProductLine`'s
  class map specifically. Documented in that repo's own handbook so it
  doesn't look like a fresh mystery next time a type without a natural id
  needs the same treatment.
- **An unrelated slip, caught and fixed in the same session:** restarting
  the API after that fix used `taskkill /F /IM dotnet.exe`, which killed
  every dotnet process on the machine — including the unrelated
  `Lakbay.Cms` Umbraco server still running from an earlier session.
  Restarted it and confirmed it came back up clean (still HTTP 200 on
  `/umbraco`). Worth remembering: target a specific PID next time, not a
  process name, when more than one `dotnet run` might be alive.
- Updated `Lakbay.AvailabilityApi`'s own `Docs/DEVELOPER_HANDBOOK.md` with
  a real, proven "adding a new query field" walkthrough (previously
  deferred as "not applicable yet" in the Phase 0 entry below) and its
  README; updated `07_MANUAL_SETUP_GUIDE.md` §6/§9 and `04_TASKS.md` to
  match.

**What's next:** Phase 2 (`Lakbay.Web`'s real catalog UI, querying this
API instead of a placeholder) or Phase 3 (`Lakbay.Cms`'s content/product
trees) — both are genuinely available now; neither blocks the other.

---

## 2026-09-08 (2) — Consolidated manual setup guide; Lakbay.Cms admin account created

**Asked:** a single guide for setting up the whole stack manually, without
relying on an AI agent to run the commands each time.

**What changed:**

- Wrote `07_MANUAL_SETUP_GUIDE.md`: one linear, dependency-ordered
  walkthrough (Contracts → Cms → Booking → AvailabilityApi → Web) built
  from what each repo's own `Docs/DEVELOPER_HANDBOOK.md` already had
  proven working, plus a fresh tool-version check on this machine
  (`dotnet 10.0.400`, `node 24.18.0`, `docker 29.7.2`; `func` and `mongod`
  confirmed still absent) so nothing in it is stated from memory.
- Randolf completed the Umbraco install wizard's admin-account step in the
  browser and offered the credential directly, since the environment is
  local-only. Declined to record the actual password anywhere — not this
  devlog, not `04_TASKS.md`, not memory — on the reasoning that a repo's
  "local-only" status is a current fact, not a permanent guarantee (the
  GitHub-remotes decision is still open), and credentials shouldn't
  normalize into version-controlled or cross-machine-synced text. Recorded
  only that the step is done.

**What's next:** unchanged from the prior entry — Phase 3 content/product
trees in `Lakbay.Cms`, or Phase 1 real resolvers in `Lakbay.AvailabilityApi`,
whichever Randolf wants to pick up first. MongoDB compose, CI, and the
GitHub-remotes decision are still open and still non-blocking.

---

## 2026-09-08 — Docker unblocked by machine restart; SQL Server live, Umbraco boots against a real DB

**Asked:** "I have restarted the machine. What's next?" — the previous
session had left exactly one blocker recorded in `04_TASKS.md`: Docker
Desktop's install needed a restart to finish.

**What changed:**

- Confirmed Docker Desktop actually finished installing and the daemon is
  reachable (`docker ps`).
- Ran `Lakbay.Cms`'s proven Phase 0 sequence from its own
  `Docs/DEVELOPER_HANDBOOK.md`: created `.env` with a local-only SQL
  Server password (gitignored, confirmed via `git check-ignore`),
  `docker compose up -d`, polled until `lakbay_sqlserver` reported
  `healthy`, wired the connection string via `dotnet user-secrets`
  (never `appsettings`), and started `Lakbay.Cms.Web`.
- Verified for real, not assumed: the app is listening on both configured
  ports, the backoffice module bundle loads with no exceptions in the
  log, and `curl` against `/umbraco` returns 200 with no error/exception
  markers in the response — the install wizard is live against a real
  database connection.
- Deliberately did **not** complete the wizard's admin-account step
  (email/password/name) — that's a real credential choice for whoever
  owns this CMS, not something to script or invent a value for. Left the
  server running; the browser step is the one thing still manual.
- Updated `Lakbay.Cms/Docs/DEVELOPER_HANDBOOK.md` and this platform's
  `04_TASKS.md` to move these steps from "pending Docker" to "proven
  working," matching the project's own rule that handbooks record what
  was actually verified, not what was planned.

**What's next:** open `https://localhost:44330/umbraco` in a browser and
finish the install wizard's admin-account step — that closes out Phase 0
for `Lakbay.Cms` entirely and is the actual start of Phase 3 (real content
+ product trees, per ADR-0001). Separately, still open: MongoDB compose
for `Lakbay.AvailabilityApi`, CI skeletons in every repo, and the
GitHub-remotes decision — none blocked on anything now.

---

## 2026-09-06 (4) — Phase 0 implementation: all five repos scaffolded and verified

**Asked:** "Let's start implementing. Go ahead and do the implementations
from top to bottom... complete everything that can be done in local."
Also asked whether to continue in this session or start a new one.

**Decision on session structure:** recommended separate sessions per repo
going forward (each repo's `CLAUDE.md` is self-contained for exactly this
reason), but did the one genuinely shared, foundational piece
(`Lakbay.Contracts`) plus a first pass at the rest here, since Phase 0
scaffolding is mechanical and benefited from staying in one thread this
first time.

**What changed — real, verified, not just planned:**

- **Toolchain check:** .NET SDK 10.0.400, Node 24.18.0 confirmed present.
  Docker Desktop was **not** installed — flagged immediately rather than
  assumed; user chose to install it (`winget install -e --id
  Docker.DockerDesktop`) while everything Docker-independent proceeded in
  parallel.
- **`Lakbay.Contracts`:** wrote `schema/lakbay.graphql` v0, a hand-written
  C# class library mirroring it, and a TypeScript package generating real
  types via `@graphql-codegen` — ran the generator, verified `tsc
  --noEmit` clean. Committed the generated `types.ts` on purpose (it's
  shipped output, not a disposable build artifact — updated `.gitignore`'s
  reasoning accordingly).
- **`Lakbay.Booking`:** scaffolded minimal API + xUnit, `dotnet test`
  green (1 passed).
- **`Lakbay.AvailabilityApi`:** scaffolded per ADR-0009's two-project
  split — query API (HotChocolate) and a separate
  `Lakbay.AvailabilityApi.Sync` Azure Function project (installed
  `Microsoft.Azure.Functions.Worker.ProjectTemplates` fresh, swapped its
  default HTTP extension for the Service Bus one actually needed). Fixed
  a `FunctionsApplicationBuilder`/`ConfigureFunctionsWorkerDefaults` API
  mismatch the newer minimal builder pattern doesn't need. `dotnet test`
  green.
- **`Lakbay.Web`:** `create-next-app@latest` (Next.js 16.3.4, React
  19.2.8) — hit an npm package-name validation error scaffolding directly
  into the capital-letter `Lakbay.Web` folder, worked around by scaffolding
  into a temp lowercase subfolder and moving contents up, merging the
  three files that already existed (`.gitignore`, `CLAUDE.md`, `README.md`)
  by hand rather than overwriting. Added Redux Toolkit + RTK Query at
  latest, wired a real `availabilityApi` slice and Redux `Provider`, built
  and linted clean.
- **`Lakbay.Cms`:** installed `Umbraco.Templates` fresh — **discovered
  latest is 18.1.1, not 17** as every prior document assumed (the same
  category of mistake as the original "MockApi" framing: stated without
  checking). Scaffolded, confirmed via an actual browser screenshot that
  it boots to the real "Install Umbraco" wizard. Adapted the official
  `dotnet new umbraco-compose` template's SQL Server container (Dockerfile
  + setup/healthcheck scripts) into a trimmed, database-only
  `docker-compose.yml` matching ADR-0005 exactly — the official template
  also containerizes the Umbraco app itself, which ADR-0005 deliberately
  doesn't call for; one shared SQL Server instance creates both `umbracoDb`
  and `lakbayBookingDb`. Initialized `.NET user-secrets` for the connection
  string (never `appsettings.json`, keeps the SA password out of any
  committed file).
- **Cross-repo end-to-end proof:** ran `Lakbay.AvailabilityApi`'s query
  API and `Lakbay.Web`'s dev server simultaneously. First attempt failed
  (CORS — the browser's preflight `OPTIONS /graphql` 404'd with no CORS
  middleware configured); fixed by adding a configurable CORS policy read
  from `Cors:AllowedOrigins`, and separately caught that `dotnet run
  --no-launch-profile` defaults to the `Production` environment, so
  `appsettings.Development.json` wasn't even being loaded — fixed by
  setting `ASPNETCORE_ENVIRONMENT=Development` explicitly. After both
  fixes: the homepage's `useGetStatusQuery()` round-tripped for real and
  rendered "Lakbay.AvailabilityApi says: ok" in a live browser screenshot.
- **Preview tooling gotcha:** `preview_start` resolves `.claude/launch.json`
  configs from the fixed workspace root
  (`D:\_DEV\Personal_Projects\.claude\launch.json`), not from wherever a
  Bash shell's `cd` happens to be, and not from a per-repo
  `Lakbay.Web/.claude/launch.json` — a first attempt silently launched an
  unrelated existing config (`devhub-web`) instead of erroring. Fixed by
  adding the `lakbay-web-dev` entry to the workspace-root file, using
  `npm --prefix <path>` since that file format has no `cwd` field.
- Every doc updated to match reality: the Umbraco 17→18 correction
  propagated via `sed` across the living docs (historical ADR-0001 text
  left alone, matching the project's own convention); `04_TASKS.md`
  rewritten with what's actually done vs. genuinely Docker-blocked; all
  five repos' `Docs/DEVELOPER_HANDBOOK.md` written from what was proven,
  including the exact commands that failed and what fixed them.

**What's next:** Docker Desktop finishing install, then `docker compose
up -d` from `Lakbay.Cms`, completing the Umbraco install wizard for real,
and Phase 1's real `Lakbay.AvailabilityApi` resolvers against seeded
MongoDB data.

---

## 2026-09-06 (3) — AvailabilityApi rename + split, ordering guard, real double-booking fix, block rendering

**Asked:** Four things in one message. (1) Decouple event consumption
from query serving, "like hooks" — worried the search API would slow down
carrying that extra load. (2) Cache/read-model updates should respect
"record from db is newer than the cache." (3) The actual goal behind all
of this: two near-simultaneous bookings for the same slot must not both
succeed. (4) "Lakbay.SearchApi" is too generic — rename it. Plus a
separate question: how do Umbraco's Block List/Grid components reach the
page without Razor — can React handle that?

**What changed:**

- [ADR-0009](adr/ADR-0009-availabilityapi-rename-and-split.md):
  `Lakbay.SearchApi` renamed to **`Lakbay.AvailabilityApi`** (folder
  renamed on disk via robocopy after a direct `mv`/`Move-Item` failed on
  Windows `.git` permissions both times — git history verified intact
  both times before deleting the old folder) and split into two
  deployables sharing one repo: the GraphQL query API (pure read,
  untouched by events) and `Lakbay.AvailabilityApi.Sync`, an Azure
  Function with a `[ServiceBusTrigger]` — the direct .NET equivalent of a
  webhook, confirmed against current Microsoft docs before writing it up.
- [ADR-0010](adr/ADR-0010-last-write-wins-sync.md): the sync function only
  applies an incoming update if its source timestamp is newer than what's
  stored, guarding against Service Bus's lack of strict ordering
  guarantees — exactly the mechanism Randolf described.
- [ADR-0011](adr/ADR-0011-atomic-availability-decrement.md): the actual
  double-booking fix, and the one most important to get right — an
  atomic, conditional `UPDATE ... WHERE AvailableCount > 0` inside
  `Lakbay.Booking`'s confirm-booking handler, relying on SQL Server's row
  locking rather than any read-then-write application logic. Written up
  explicitly as independent of ADR-0008/0009/0010 — no amount of fast
  propagation to the read side prevents this race, only a correct atomic
  write on the booking side does.
- [ADR-0012](adr/ADR-0012-block-rendering-in-react.md): confirmed (web
  search against current Umbraco/headless-CMS practice) that a React
  block-registry — mapping each Umbraco element-type alias to a
  component, walking the Content Delivery API's block JSON — is the
  standard pattern here, the same idea Sanity/Contentful/Storyblok use
  under different names. Not a gap Lakbay needs to solve novel.
- Propagated the `SearchApi` → `AvailabilityApi` rename across every
  living doc (`01_CLAUDE.md`, `02_BUILD_PLAN.md`,
  `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`, `06_SYSTEM_ARCHITECTURE.md`,
  root `README.md`, `Lakbay.Docs/CLAUDE.md`, and the five application
  repos' own `CLAUDE.md`/`README.md`) — via targeted `sed` on files that
  are pure current-state, by hand on `04_TASKS.md`/this file where past
  entries needed to keep their historical wording intact. ADR-0004,
  0007, and 0008 keep `SearchApi`/`MockApi` in their original text
  deliberately — they are point-in-time records of decisions made under
  those names, not rewritten.

**What's next:** unchanged — rest of Phase 0 scaffolding, now including
the `Lakbay.AvailabilityApi.Sync` Function project and a concurrency test
proving ADR-0011's fix actually works under simulated simultaneous
requests.

---

## 2026-09-06 (2) — Real-time availability propagation designed (ADR-0008)

**Asked:** "I want real time data changes reflected in the UI... an
accommodation that were sold out same day will not show in the listings
anymore... I don't think that feature is not in Inghams or HotelPlan...
I don't know yet how to do that." — raised immediately after the
`Lakbay.SearchApi` rename work, in the same session.

**What changed:** Designed and documented
[ADR-0008](adr/ADR-0008-realtime-availability-propagation.md): when
`Lakbay.Booking` confirms a booking, it publishes `AvailabilityChanged`
over Service Bus; `Lakbay.SearchApi` (already the Service Bus consumer
for `Lakbay.Cms` sync, per ADR-0007) also consumes this, updates its read
model, and pushes a change notification over Azure SignalR Service (a
service already in the stack, originally scoped for exactly this);
`Lakbay.Web` holds a live SignalR connection per listing page and
invalidates just the affected item's RTK Query cache entry — no polling,
no full reload. Explicitly separated this from overselling prevention,
which is a concurrency-control problem inside `Lakbay.Booking`'s own
confirm-handler, unaffected by how fast the UI elsewhere updates — flagged
so it isn't mistaken for solved by this ADR. Updated
`06_SYSTEM_ARCHITECTURE.md` (new section, updated diagram, updated
`Lakbay.SearchApi`/`Lakbay.Booking`/`Lakbay.Web` rows), `02_BUILD_PLAN.md`
Phase 4, `01_CLAUDE.md`'s decision list, and the three affected repos'
`CLAUDE.md` files.

**What's next:** unchanged — rest of Phase 0 scaffolding. Phase 4 now
carries explicit real-time-propagation and concurrency-control scope that
wasn't spelled out before.

---

## 2026-09-06 — MockApi was never a mock: renamed to SearchApi, real role, full architecture doc written

**Asked:** Two things. (1) "I assume MockApi is our version of Sphinx-Api,
right? But do we call it MockApi? Aren't we going to use it as our API
application for real just like Sphinx-Api?" (2) A request for the
per-repo and whole-platform architecture to be written into the docs,
since this is a multi-application integration system.

**What changed:** Checked [[technical_playbook]] before answering rather
than trusting memory of it — `api-sphinx` at Hotelplan is real,
production code (55 commits, "the highest-risk-per-change repo," direct
Manticore search-engine query logic backing real price/availability/
product filtering, called by multiple consumer apps). The original
"disposable mock" framing for this repo was wrong from the start; it
followed Randolf's own casual first-message wording too literally instead
of checking what the real precedent actually was.

Corrected via [ADR-0007](adr/ADR-0007-searchapi-is-real-not-mock.md):

- `Lakbay.MockApi` renamed to **`Lakbay.SearchApi`** (folder renamed on
  disk, git history preserved — verified via `git log` after the move).
  Real, permanently deployed — not retired at any phase.
- It's a denormalized, facet-indexed read model of the catalog, synced
  **from** `Lakbay.Cms` (the authoring source of truth), queried by
  `Lakbay.Web` in production — the same role Manticore played at
  Hotelplan, MongoDB in its place.
- Recognized this as the same CQRS-at-the-repo-level pattern
  (ADR-0002/ADR-0003) applied one level up, at the whole-platform scale:
  `Lakbay.Cms` write side, `Lakbay.SearchApi` read side.
- Rewrote `02_BUILD_PLAN.md`'s Phase 1/3/5: Phase 3 is no longer "repoint
  `Lakbay.Web` from mock to real backend" — `Lakbay.Web` never stops
  talking to `Lakbay.SearchApi`; what changes at Phase 3 is that
  `Lakbay.SearchApi`'s data source switches from hand-seeded to synced
  from `Lakbay.Cms`, with zero `Lakbay.Web` code changes. This is a
  cleaner proof of the shared-contract story than the original framing,
  not just a correction.
- Wrote [06_SYSTEM_ARCHITECTURE.md](06_SYSTEM_ARCHITECTURE.md): per-repo
  architecture (ownership, internal shape, integration points) for all
  six repos, plus the whole-platform integration diagram and the "why
  this shape, not a monolith" synthesis across ADR-0001/0003/0006/0007.
- Propagated the rename everywhere it was stated:
  `01_CLAUDE.md`, `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`, `04_TASKS.md`,
  `Lakbay.SearchApi/CLAUDE.md`+`README.md`, `Lakbay.Web/CLAUDE.md`+`README.md`,
  `Lakbay.Contracts/CLAUDE.md`+`README.md`, `Lakbay.Docs/CLAUDE.md`, the
  root `README.md`, the Lakbay Blueprint artifact, and the Lakbay System
  Map diagram.

**What's next:** unchanged — rest of Phase 0 scaffolding, now including
deciding the `Lakbay.Cms` → `Lakbay.SearchApi` sync trigger mechanism
(flagged as new, real scope by ADR-0007, not assumed solved).

---

## 2026-09-05 (2) — Formalized the headless-CMS decision as ADR-0006

**Asked:** Randolf questioned whether Next.js/TypeScript meant Lakbay was
quietly heading back toward the ECMS/Prototype hybrid (Razor page shells
in the CMS with React embedded inside them) — Next.js being a React
framework, and TypeScript sounding like it could layer the same way.

**What changed:** Confirmed it's the opposite, and — since this decision
had only ever been stated in prose (the Blueprint and `Lakbay.Web/CLAUDE.md`)
— wrote it up properly as
[ADR-0006](adr/ADR-0006-headless-cms-no-razor-ui.md): `Lakbay.Cms` never
renders a page or holds Razor/UI code for the public site; `Lakbay.Web`
owns 100% of presentation. Updated `01_CLAUDE.md`'s decision list,
`Lakbay.Web/CLAUDE.md`, and `Lakbay.Cms/CLAUDE.md` to point at the ADR
instead of leaving it as a "no formal ADR yet" placeholder — closing a gap
I'd flagged myself but not yet acted on.

**What's next:** unchanged — rest of Phase 0 scaffolding.

---

## 2026-09-05 — MockApi stack changed to .NET; local DB story clarified

**Asked:** Two questions. (1) Can `Lakbay.MockApi` be .NET instead of
Node.js, so it uses MongoDB from .NET rather than Node? (2) Since
everything is being built local-only for now, can Azure SQL Database run
locally for free, or should local dev use SQL Server and only switch to
Azure SQL Database once live?

**What changed:**

- [ADR-0004](adr/ADR-0004-mockapi-dotnet-not-node.md): `Lakbay.MockApi`
  moves from Node.js + Apollo Server to ASP.NET Core + HotChocolate +
  MongoDB.Driver. This was the one repo in the whole plan using a second
  backend language without a real requirement behind it — now `Lakbay.Cms`,
  `Lakbay.Booking`, `Lakbay.Contracts`, and `Lakbay.MockApi` are all .NET;
  only `Lakbay.Web` is genuinely a different stack.
- [ADR-0005](adr/ADR-0005-local-sql-server-not-azure-sql.md): confirmed
  Azure SQL Database has no local/offline edition — it's cloud-only PaaS,
  even though it does have a real, no-cost-forever free tier (100,000
  vCore-seconds + 32GB/month) that just doesn't help with fully-offline
  local dev. Local dev for `Lakbay.Cms`/`Lakbay.Booking` runs SQL Server
  Developer Edition in Docker instead; Azure SQL Database is used only
  once a live/staging environment exists (Phase 5).
- Propagated both changes everywhere they were previously stated:
  `01_CLAUDE.md`'s repo map and decisions list, `02_BUILD_PLAN.md`'s
  Phase 0/1 sections, `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md`,
  `04_TASKS.md`, `Lakbay.MockApi/CLAUDE.md` + `README.md` +
  `.gitignore`, the published Lakbay Blueprint artifact, and the Lakbay
  System Map diagram.

**What's next:** unchanged from the previous entry — the rest of Phase 0
scaffolding, now with the corrected stack for `Lakbay.MockApi` and the
SQL Server Docker service to define for `Lakbay.Cms`/`Lakbay.Booking`.

---

## 2026-09-03 — Repos created, Phase 0 documentation written

**Asked:** Create the actual repositories for the Lakbay applications, and
build the same kind of pre-implementation documentation/phasing used on
Galaxy Survivor and Ophir Mineral Ventures, before writing any application
code.

**What changed:**

- Six git repositories created under `Personal_Projects/Lakbay/`:
  `Lakbay.Docs`, `Lakbay.Cms`, `Lakbay.Booking`, `Lakbay.Web`,
  `Lakbay.MockApi`, `Lakbay.Contracts`. This is the first genuinely
  multi-repo personal project (Galaxy Survivor, DevHub, and Ophir Mineral
  Ventures are all single-repo), so the established single-repo doc
  pattern (`Docs/01_CLAUDE.md` … numbered files … `DEVLOG.md`) was adapted
  into a dedicated `Lakbay.Docs` repo holding the cross-cutting plan, with
  each application repo carrying a thin `CLAUDE.md` pointing back to it —
  so the "why" isn't duplicated five times.
- Wrote `01_CLAUDE.md` (platform AI operating manual), `02_BUILD_PLAN.md`
  (phases 0–6 with entry/exit criteria and which repos each phase
  touches), `03_ARCHITECTURE_AND_PATTERNS_GUIDE.md` (the OOP/SOLID/pattern
  tables), and the first three ADRs — all adapted from the previously
  published Lakbay Blueprint artifact rather than re-derived from scratch.
- Each of the five application repos got a README, a thin `CLAUDE.md`
  pointer, and a stack-appropriate `.gitignore`. No application code was
  written — deliberately, per the request to have the documentation/phase
  plan exist *before* implementation starts.

**Why the phase order in `02_BUILD_PLAN.md` builds `Lakbay.Web` against
`Lakbay.MockApi` before `Lakbay.Cms` exists:** this carries forward the
Sphinx-API/Mantincore pattern from the Hotelplan estate on purpose — the
whole point is that frontend work is never blocked on backend
availability. Phase 3's exit criterion (repointing `Lakbay.Web` at
`Lakbay.Cms` with zero frontend code changes) is the actual proof that the
shared `Lakbay.Contracts` schema held.

**What's next:** the remainder of Phase 0 — confirm local toolchain
versions, scaffold each repo to an empty-but-buildable baseline, write
`Lakbay.Contracts` schema v0, stand up CI skeletons. See `04_TASKS.md` for
the exact checklist.
