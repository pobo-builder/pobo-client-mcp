---
name: style-widgets
description: Style Pobo Page Builder widgets to match the design of the user's e-shop. Use when the user wants to restyle, unify, theme, or adjust the look of Pobo Page Builder widgets / product descriptions on their e-shop (Shoptet, Shopify, WooCommerce, PrestaShop, Upgates), or asks to make Pobo Page Builder content match their brand. Uses the `pobo` MCP server tools.
---

# Style Pobo Page Builder widgets to match the e-shop design

You generate SCSS that makes Pobo Page Builder widgets visually match the user's e-shop, and deploy
it through the `pobo` MCP server. All logic runs on the Pobo Page Builder backend — your job is to
read the e-shop's design, write the SCSS, push it, and iterate.

Communicate with the user in their language (typically Czech).

## Prerequisites & auth

The `pobo` MCP server is connected once via OAuth — the user runs
`claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
and logs in with their Pobo Page Builder account in the browser. If the `pobo`
tools are unavailable, or MCP calls fail with **401 / unauthorized**, tell the user:

> Connect the Pobo server with
> `claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
> and sign in with your Pobo account in the browser. If the connection expired,
> run `/mcp` and sign in again.

There are no tokens to handle — authentication is a browser login, never ask the
user for credentials in the conversation.

Rate limit is 60 requests/min per user; the normal workflow fits comfortably, so do
not fire bulk page fetches in tight loops. On 429, wait and retry.

## Workflow

### 1. Pick the e-shop

Call `list_eshop`. If exactly one e-shop is returned, use it. If multiple, ask the
user which one to style. If none, tell the user to set up their e-shop in Pobo Page Builder first.

### 2. Load the theming contract

Call `get_theming_contract`. It returns:

- `variable` — ~1400 CSS custom properties (`--pobo-widget-*`, `--pobo-global-*`) with
  their default values. Naming convention: `--pobo-widget-{block}-{property}` with
  suffixes like `-bg`, `-padding`, `-margin`, `-border-radius`, `-box-shadow`, and
  `-before-*` / `-after-*` for pseudo-elements.
- `widget_block` — ~80 native BEM blocks usable as selectors (block `.widget-infobox`,
  elements like `.widget-infobox__title`).
- `declared_on` — where the variables are declared (`:root`). **Overrides must also be
  written on `:root`.**

Keep the variables and blocks relevant to the task in mind; don't dump the full list
on the user. Before styling a widget, filter the contract for its variables
(everything starting with `--pobo-widget-{block}-`) so you know what is themable —
this list, not the raw CSS properties, is your styling surface.

### 3. Extract design tokens from the e-shop

Call `fetch_eshop_page` with the `eshop_id` and `path: "/"` for the homepage. Also
fetch one product detail page — find a product URL path in the homepage HTML, or ask
the user for one. Pass only the path (starting with `/`); the host is fixed to the
stored e-shop URL and cannot be changed — never try to work around this.

From the HTML and inline CSS extract the design tokens:

- primary and secondary brand colors
- font family (and weights if visible)
- border radius style (sharp / rounded / pill)
- button style (background, border, radius, hover if discoverable)
- shadows

Bodies over 500 KB are truncated (look for a `<!-- TRUNCATED: ... -->` marker); work
with what you get. Summarize the extracted tokens to the user before generating SCSS.

### 4. Inspect the widget content itself

On the fetched product page, find the Pobo wrapper: `<div id='pobo-all-content'
data-pobo-content='...' data-pobo-page-id='...'>`. Take those two attribute values
and call `get_content_html` with them (plus `eshop_id`). It returns:

- the **clean widget HTML** of that page — no shop chrome (menus, footer, cookie
  bars), so you see the exact markup your CSS will target;
- `widget_class` in structured content — the inventory of `widget-*` blocks the
  page actually uses.

**Style only blocks that appear in the inventory** — do not blind-style all ~80
contract blocks. For a broader picture, repeat on 2-3 representative pages (a
product, a category) rather than trying to enumerate everything.

### 5. Generate the SCSS

Rules:

- **Variables first — this is the core rule.** For every property you want to change,
  first look up the matching variable in the contract: `--pobo-global-*` for
  cross-widget tokens (colors, fonts, radii), then
  `--pobo-widget-{block}-{property}` for the specific widget (including `-before-*` /
  `-after-*` pseudo-element variants). Override it on `:root`. Write a direct
  `.widget-*` rule **only after confirming the contract has no variable** for that
  exact property — and scope it as narrowly as possible.

  ```scss
  // BAD — bypasses the contract, fragile against Pobo updates
  .widget-infobox { background: #f5f0ea; border-radius: 12px; }

  // GOOD — overrides the contract variables
  :root {
    --pobo-widget-infobox-bg: #f5f0ea;
    --pobo-widget-infobox-border-radius: 12px;
  }
  ```

- Self-check before every push: each declaration inside a `.widget-*` selector is a
  smell. Re-check the contract for each one; keep it only if no variable exists, and
  note in your reply to the user which direct rules you kept and why.
- **Always produce the complete SCSS file.** A push replaces the entire previous
  content — never generate an incremental diff. When iterating, re-emit everything.
- **Never generate:** `@import`, `expression(`, `javascript:`, `vbscript:`,
  `behavior:`, `-moz-binding`. For `url(...)`, only these are accepted: a
  relative/local path, an absolute `https://` URL on the Pobo CDN or the
  platform's own asset CDN (`cdn.myshoptet.com`, `cdn.shopify.com`), or a
  `data:image/...` URI (png/jpeg/gif/webp/avif/svg+xml). Any other origin, any
  protocol-relative `//host/...` URL, or a non-image `data:` URI is rejected.
  The server rejects all of this (the blacklist also catches CSS-escape
  obfuscation), so don't produce it in the first place. For fonts, set
  font-family variables to font names — never `@import` a font.
- CSS only — no JavaScript assets, no widget/content management (out of scope).
- Max 256 KB of SCSS.

### 6. Push

Call `push_asset_css` with `eshop_id`, a descriptive `name` shown to the merchant in
the Pobo Page Builder admin (e.g. "AI design unification"), and the complete `scss`.

- Each e-shop has at most one AI asset; the push is an idempotent upsert
  (`action: created | updated`).
- On `SCSS does not compile: ...` — fix the SCSS and push again.
- On `Compiled CSS contains forbidden constructs ...` — remove the offending
  construct and push again.
- On success, the CSS deploys to the e-shop's CDN bundle (`cdn_url` in the response);
  propagation takes seconds.

### 7. Verify — see the result yourself

Call `screenshot_eshop_page` with the `eshop_id`, the product page `path` and
`selector: "#pobo-all-content"` — you get a real-browser screenshot of exactly the
widget area you just styled (the CSS propagates to the CDN in seconds). Compare it
against the design tokens from step 3 and iterate: adjust the SCSS, push the
complete file again, screenshot again.

- Also check `viewport: "mobile"` (rendered at 2× scale, retina-sharp) — spacing
  and typography issues often show up only on small screens.
- The tool also returns a public CDN URL of the screenshot — share it with the
  user so they can see what you see.
- Finish by asking the user to confirm on their live e-shop (hard refresh,
  Ctrl/Cmd+Shift+R).

**Cascade fallback:** if the user reports no visible change even though the push
succeeded, the `:root` overrides may be losing the cascade against `generic.css` on
some platforms. Increase specificity — use `:root:root { ... }` for the variable
overrides, or as a last resort set the properties directly on the `.widget-*` blocks.

### 8. Rollback

If the user wants to revert: call `list_asset` to get the AI asset id, then
`delete_asset` with `eshop_id` and `asset_id`. The CDN bundle regenerates
automatically. The user can also delete the asset themselves in the Pobo Page Builder admin.
Confirm with the user before deleting.

## Error handling

All tools return human-readable MCP errors — read them and react (fix the SCSS, pick
another e-shop, retry later). Common ones:

- `Eshop not found.` — wrong `eshop_id`; re-run `list_eshop`.
- `Asset not found.` — the asset id is not an AI asset of that e-shop; re-run `list_asset`.
- `Page returned HTTP {status}.` / `Page redirected outside the eshop domain` /
  `Too many redirects.` — try another path or ask the user for a working URL path.
- `Theming contract is temporarily unavailable ...` — retry shortly.
- `Content not found.` — wrong `page_id`/`content` pair or the page does not belong
  to this e-shop; re-read the data attributes from `fetch_eshop_page`.
- `Screenshot failed: ...` / `Screenshot service is not configured.` — verification
  screenshots are unavailable; fall back to asking the user to check with a hard
  refresh, do not block the workflow on it.
- 401 — see Prerequisites & auth above.
