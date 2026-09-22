---
name: design-description
description: Design a Pobo Page Builder description end to end, matched to the eshop's own fonts, colors and radii — structure, copy in the eshop language, real images and a complete SCSS style, with structural and visual QA — for a product, a category or a blog article, from a short brief. Use when a merchant or the Pobo team wants to "nastylovat produkt", "navrhnout popisek", "udělat design produktu", a "vzorový / ukázkový popisek", or wants one entity to look designed rather than assembled. Uses the `pobo` MCP server tools. Do not use for restyling all widgets of an eshop with no content work (`style-widgets`) or for plain article writing with no styling (`write-blog`).
---

# Design a description in Pobo Page Builder

You turn one entity (a product, a category or a blog article) into a designed
page: a widget plan, copy in the eshop language, images from the merchant's
library, and one complete SCSS file. The user gives you a brief. The rest comes
from the tools and from two reference files shared with the `style-widgets`
skill (`skills/style-widgets/` in this plugin):

- `runtime.md` — what Pobo CSS can and cannot do, and where the theming
  variables really are. Read it before planning; designing against it wastes the
  run.
- `widgets.md` — widget id → CSS block → variable prefixes → slots. Facts, no
  opinions.

Content tools are the ones `write-blog` documents; styling follows `style-widgets`.
This skill adds the order, the scope (one entity) and the two rules below.

Communicate with the user in their language (typically Czech). Write the page in
the eshop's language (`write_in_language` from the catalog).

## Two things that are not negotiable

**The page belongs to the shop.** Fonts, colors, type sizes, button and image
radius come from the merchant's site (`fetch_eshop_page`), for a merchant's
product and for a Pobo sample alike. Design happens in what sections exist, in
what order, which photo carries which claim, how the type scale is distributed,
and what the copy says.

**Less is more.** Every widget has to earn its place with one claim the customer
cares about. When in doubt, leave it out. Do not pad the page to a length, and do
not add a section to "break up" the page.

## One style asset per eshop

`push_asset_css` replaces the eshop's single AI style asset; other Pobo
descriptions on the shop render with it too. Before pushing, `list_asset`; if an
asset exists, ask whether it may be replaced, and read the deployed CSS from the
page's `<link>` to keep its rules (`runtime.md`, "Reading the deployed asset").
Scope entity-specific rules to `#pobo-all-content[data-pobo-page-id='<entity_id>']`
and keep shop-wide rules (fonts, typography, links, buttons, surface) unscoped.

## Budget

One pass, one correction loop. Read everything once, compose once, check the HTML
before any screenshot, push one complete SCSS, two screenshots (desktop, mobile),
a third only after large fixes. Adding widgets one by one with
`add_content_widget` means the plan was skipped; go back to it.

## Workflow

### 1. Discovery

In parallel where possible:

- `list_eshop` (check `pobo_llm_enabled`), then `find_product` / `list_category` /
  `list_blog` → `entity_type` + `entity_id`.
- `get_platform_content` → the facts the shop already states. Carry facts over,
  never wording.
- `get_content_widget` → what the entity holds now; say before the compose that it
  will be replaced.
- `list_content_image`, `list_content_media` → the merchant's own photos, with
  their shapes. Stock is a fallback; AI photos are paid and need the user's yes.
- `get_content_widget_catalog` → the widget ids this eshop has; the catalog is
  the truth, `widgets.md` the map to CSS.
- `fetch_eshop_page` for `/` and for the entity path → the fonts the host page
  loads (`<link rel="preload" as="font">`, `@font-face`, Google Fonts links,
  `font-family:` declarations), colors, heading and body sizes, button style,
  radius. These are your tokens.
- `list_icon` only if a widget with icon slots is planned.

### 2. Plan

Show the user, in a few lines: the tokens you read, the section list (position,
widget id, what it says, which image), and what you left out. Every image and
icon slot has an image or a CSS decision to hide it (`runtime.md`). Proceed
unless the brief is ambiguous about the entity, the audience or the language.

### 3. Content

Write the copy inside each role's `max_length`, then `compose_content` once with
the whole `widget` array plus `image` and `icon` URLs. `mode: "replace"` returns a
preview and a `confirm_token`; repeat with `confirmed: true`.

### 4. Structural QA (text only)

`render_content_html`. The server reports byte size and empty text slots; check
the rest yourself in the returned HTML:

- `<img src="">` → fill with `set_content_widget_image` (`image_source:
  "uploaded"`) or hide in CSS.
- Purchase links („Koupit“, „Do košíku“) and availability, price or delivery
  claims („skladem“, „ihned k odeslání“) → remove; the shop renders those live,
  the description cannot.
- Text over `max_length` → shorten.

Fix with `edit_content_widget`. No screenshots yet.

### 5. Style

Follow `style-widgets` steps 2, 5 and 6 (contract, SCSS rules, push), with
these differences:

- the block inventory is the render HTML of this entity, not 2 to 3 pages;
- the tokens are the ones you read in step 1, already confirmed with the user;
- rules specific to this entity go under
  `#pobo-all-content[data-pobo-page-id='<entity_id>']`, shop-wide rules stay
  unscoped;
- one push with the complete file; keep the SCSS so you can re-emit it after
  the visual QA.

### 6. Visual QA

`screenshot_eshop_page` with the entity `path` and `selector: "#pobo-all-content"`,
desktop then `viewport: "mobile"`. Check: font fallback (a face you did not ask
for means a wrong family name or one the host does not load), the page reading
as one surface with the shop's radius, consistent gaps, sibling widgets of equal
size, image crops, mobile overflow. Fix, push once, final screenshot only if the
fixes were large. Share the screenshot CDN URLs.

### 7. Report

Entity URL, tokens matched, the plan executed, what the style asset now covers
(this entity or the whole shop), what you left out, screenshot URLs. Everything
is versioned: `revert_content` and `delete_asset` undo it.

## Error handling

- `Eshop not found.` / `Product not found.` → `list_eshop` / `find_product` again.
- Widget id not in the catalog → pick from the catalog you got.
- `SCSS does not compile` / `forbidden constructs` → `runtime.md`; usually an
  `@import` or an `url()` off the allowlist.
- `confirm_required: true` on images → paid AI photo; ask, or use the library.
- Screenshot unavailable → ask the user to hard-refresh; do not loop.
- 401 → `claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`, or `/mcp` to re-authenticate.
