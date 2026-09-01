# Redkisses Shopify Theme

This repo is the Shopify theme for the **Redkisses** store. It is built on Shopify's
**Horizon** base theme (`sections/`, `blocks/`, `snippets/` retain Horizon's stock
components) with a custom brand layer of `nani-*` sections (e.g. `nani-header.liquid`,
`nani-product.liquid`, `nani-collection.liquid`, `nani-contact.liquid`) that carry the
Redkisses-specific design and behavior. Prefer editing/extending the `nani-*` layer for
brand-specific work; treat non-`nani-*` Horizon files as upstream unless a task
specifically targets them.

## Environment map

| | |
|---|---|
| GitHub repo | `assadujjaman112/redkisses` |
| Main branch → | Shopify draft theme `redkisses/main` |
| Draft theme ID (testing/preview) | `192706838850` |
| Live theme ID (production) | `192704807234` |

Treat theme **`192706838850`** as the testing/preview environment. All iteration and
verification happens there.

## Hard rules

- **Never publish a Shopify theme unless explicitly asked to.**
- **Never modify or publish the live theme (`192704807234`) directly.**
- **Theme push, theme pull, and publish are risky operations — always ask for
  confirmation before running them**, even against the draft theme.
- Do not use `shopify theme check --auto-correct`.

## Workflow for every change

1. Inspect the existing code and relevant Shopify/Horizon patterns before editing —
   don't guess at conventions this theme already establishes elsewhere.
2. Preserve the existing Redkisses visual design and functionality unless the
   requested change specifically requires altering it.
3. Make only the changes the task calls for — no unrelated edits, formatting passes,
   or opportunistic refactors.
4. Before considering a code change complete, run `shopify theme check --output json`
   (no `--auto-correct`).
5. Run the Shopify MCP `validate_theme` tool on modified theme files when appropriate.
6. Review `git diff` after modifications to confirm the change is exactly what was
   intended and nothing else moved.
7. Explain what changed and why.
8. If a request is ambiguous — especially anywhere it implies a design or behavior
   decision rather than a mechanical fix — ask rather than guessing.

## Historical context: Gate 1 critical fixes

A full read-only theme audit was run against this repo using both
`shopify theme check --output json` and the Shopify MCP `validate_theme` tool. It
found 48 issues (4 Critical, 13 High, 17 Medium, 9 Low, 5 Informational). "Gate 1"
then fixed **only** the 4 Critical, storefront-breaking issues; High/Medium/Low
findings were deliberately left for later gates and remain outstanding. This section
is a record of that work — **do not re-modify these files based on this summary
alone**; if they need further work, inspect current source first.

1. **`sections/nani-contact.liquid`** — the section used a templated HTML tag name
   (`<{% if is_link %}a{% else %}div{% endif %} ...>`), which Shopify's Liquid-HTML
   parser rejects as a hard error, breaking `templates/page.contact.json`. Fixed by
   capturing the shared inner markup once and emitting two complete, valid branches
   (`<a>` vs `<div>`) instead of interpolating the tag name itself.

2. **`sections/nani-product.liquid` (`has_product` logic)** — `has_product` was
   derived from `p.available != blank`. In Liquid, `false == blank` is `true`, so a
   real sold-out product (`available: false`) was incorrectly treated as "no
   product," pushing it into demo/placeholder rendering branches (fake gallery,
   fake variants, fake price). Fixed by deriving `has_product` from `p != blank`
   only — availability is a separate concern from "is this a real product."

3. **`sections/nani-product.liquid` (Add to Bag logic)** — introduced an explicit
   `is_sold_out` flag (real product AND `current_variant.available == false`) as the
   single source of truth, and used it consistently for the Add to Bag button
   (disabled + "Sold out" label), the stock/inventory label, and the Buy Now button
   — rather than patching one condition in isolation, per the instruction to check
   surrounding product-form logic.

4. **`sections/nani-collection.liquid` (faceted filtering)** — the filter form's
   only submit control lived inside `<noscript>` with no JavaScript, so filtering
   was effectively non-functional for real shoppers. Fixed using Shopify's
   documented supported pattern: a real GET form using `collection.filters` /
   `filter.values[].param_name` etc., with a real (always-rendered) submit button,
   progressively enhanced by a small section-scoped script that auto-submits on
   `change` and hides the button via a CSS class. A hidden `sort_by` input was added
   so filtering doesn't wipe the active sort. No filter values were hardcoded, and
   no existing filter functionality was removed.

**Design decision made during Gate 1 (documented, not undone):** Horizon's own async
faceted-filtering component (`assets/facets.js`, `<facets-form-component>`,
`sectionRenderer.renderSection()`) was evaluated and deliberately *not* adopted,
because it requires restructuring the section's markup into that custom-element
pattern and would have conflicted with preserving the existing Redkisses design.
The simpler GET-form-with-JS-auto-submit pattern was used instead (full page reload
on filter change rather than async section replacement). Adopting the async version
is a legitimate future enhancement, not a bug — it should be a deliberate ask, not
assumed.

All three files above were verified after the fix with `shopify theme check` (errors
1 → 0, warnings unchanged) and MCP `validate_theme`, and confirmed by tracing actual
Liquid state (available / sold-out / demo-no-product / low-stock) rather than by the
checker simply going quiet.
