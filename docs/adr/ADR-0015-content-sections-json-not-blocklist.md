# ADR-0015: Content-tree page sections are JSON-in-Textarea, not a real Block List (v0)

- **Status:** Accepted — amends ADR-0012's storage mechanism, not its intent
- **Date:** 2026-09-08
- **Repo(s) affected:** `Lakbay.Cms`

## Context

Building the Content tree (Phase 3's remaining piece), a real Umbraco
Block List was attempted for each page's `sections` field — two element
types (`heroBanner`, `imageTextBlock`), a Block List data type configured
against them (`CmsSchemaBuilder.GetOrCreateBlockListAsync`, hand-building
the `DataType.ConfigurationData` dictionary to match
`BlockListConfiguration`'s shape), and real content authored against it
by hand-constructing the stored property value to match
`Umbraco.Cms.Core.Models.Blocks.BlockListValue`'s C# shape
(`layout`/`contentData`/`settingsData`, camelCase).

The stored JSON parsed and saved without error, and was confirmed
structurally sound directly in SQL Server. But querying it back out
through Umbraco's own Content Delivery API returned `"sections": {"items":
[]}` — empty, despite two real blocks being stored. Two rounds of
reflection against the real Umbraco 18.1.1 assemblies (confirming
`BlockListLayoutItem`/`BlockItemData`/`BlockPropertyValue`'s property
shapes and JSON attributes) didn't surface why the Delivery API's own
converter fails to read it back — `BlockPropertyValue.EditorAlias` is
read but was left unset by this seeder, which is one real candidate,
but not confirmed as the actual cause within this session's time budget.

## Decision

`sections` is a Textarea holding a JSON array — the exact pattern already
proven correct and verified for `Product.priceBands` (ADR-... none
needed at the time; documented in `CatalogContentTypeSeeder`'s own doc
comment). Each array item carries a `type` discriminator
(`"hero"` | `"imageText"`) plus that type's fields:

```json
[
  { "type": "hero", "heading": "...", "subtext": "...", "imageUrl": "..." },
  { "type": "imageText", "heading": "...", "text": "...", "imageUrl": "...", "imagePosition": "left" }
]
```

No `heroBanner`/`imageTextBlock` Umbraco element types, no Block List data
type — `ContentTreeSeeder` writes the JSON array directly with
`System.Text.Json`, and `Lakbay.Web`'s block registry (ADR-0012) parses it
directly rather than going through Umbraco's Content Delivery API's
Block List conversion.

**ADR-0012's actual intent is unaffected**: a `type` string still maps to
a specific React component in `Lakbay.Web`'s block registry — this ADR
only changes *how the block list is stored and read on the Cms side*,
not the "element type alias → component" pattern itself.

## Alternatives considered

- **Keep debugging the real Block List** — rejected for now on time-
  boxing grounds, not because it's provably impossible. The stored JSON
  was self-consistent and matched the reflected C# model's property
  names; something more specific (a required `EditorAlias`, a
  `contentTypeAlias` needed despite its `[JsonIgnore]`, or a Delivery-
  API-specific "expose this element type" configuration step not yet
  found) is the likely gap. Worth revisiting with the Umbraco source
  itself open side-by-side, which this session didn't have.
- **Drop the Content tree entirely, wait for real Block List support** —
  rejected: it would leave Phase 3 without any Content-tree pages at all,
  when the actual page content (real copy, real images, a working
  Cms → Lakbay.Web read path) is genuinely there and correct — only the
  storage mechanism for the repeatable-blocks part fell back to a simpler
  shape.

## Consequences

- Editors author `sections` as raw JSON in the backoffice for now — a
  real backoffice-UX regression versus a proper Block List's visual
  block editor, worth fixing once the real cause above is found.
- `Lakbay.Web`'s block registry (built alongside this ADR) is written
  against the JSON-array shape above. If a real Block List is adopted
  later, only the *parsing* layer changes — the registry's "type →
  component" mapping concept stays the same.
- The orphaned `heroBanner`/`imageTextBlock` content types and the "Lakbay
  Page Sections" Block List data type from the abandoned attempt are left
  in the local database (harmless, unreferenced by any content) rather
  than risking an unverified content-type-deletion code path just for
  tidiness — clean up manually via the backoffice if it bothers you
  locally; not a problem for a fresh machine, which never creates them
  now that `ContentTreeSeeder` no longer does.
