# Tool reference

This is the complete reference of every tool the two Pobo MCP servers expose. The
skills in `skills/` teach the workflows built on top of these tools; this file is
the lookup for what each tool takes and returns.

Two servers:

- **merchant** — `https://api.pobo.space/mcp/client`
- **white label** — `https://api.pobo.space/mcp/whitelabel`

White label tools only work for eshops on the `b2b` platform — calling one against
an eshop on any other platform returns "Eshop not found."

Every parameter, return value and constraint documented below comes from the
server's own tool schema and description, not from guessing at intended behavior.

## Contents

- [Styling, products, labels and designs](#styling-products-labels-and-designs)
  - [`assign_product_label`](#assign_product_label)
  - [`create_label`](#create_label)
  - [`delete_asset`](#delete_asset)
  - [`fetch_eshop_page`](#fetch_eshop_page)
  - [`find_product`](#find_product)
  - [`get_content_html`](#get_content_html)
  - [`get_product_analytics`](#get_product_analytics)
  - [`get_theming_contract`](#get_theming_contract)
  - [`list_asset`](#list_asset)
  - [`list_category`](#list_category)
  - [`list_design`](#list_design)
  - [`list_design_widget`](#list_design_widget)
  - [`list_eshop`](#list_eshop)
  - [`list_label`](#list_label)
  - [`list_product`](#list_product)
  - [`push_asset_css`](#push_asset_css)
  - [`screenshot_eshop_page`](#screenshot_eshop_page)
  - [`set_product_status`](#set_product_status)
- [Blog, prompts, generation and diagnostics](#blog-prompts-generation-and-diagnostics)
  - [`add_blog_product_carousel`](#add_blog_product_carousel)
  - [`add_blog_widget`](#add_blog_widget)
  - [`convert_blog_widget`](#convert_blog_widget)
  - [`create_blog`](#create_blog)
  - [`create_prompt`](#create_prompt)
  - [`delete_prompt`](#delete_prompt)
  - [`edit_blog_widget`](#edit_blog_widget)
  - [`generate_blog_article`](#generate_blog_article)
  - [`get_blog_content`](#get_blog_content)
  - [`get_blog_generate_status`](#get_blog_generate_status)
  - [`get_blog_widget_catalog`](#get_blog_widget_catalog)
  - [`get_credit`](#get_credit)
  - [`get_export_status`](#get_export_status)
  - [`get_generation`](#get_generation)
  - [`get_preview_status`](#get_preview_status)
  - [`get_product_research`](#get_product_research)
  - [`get_prompt`](#get_prompt)
  - [`get_prompt_history`](#get_prompt_history)
  - [`list_blog`](#list_blog)
  - [`list_generation`](#list_generation)
  - [`list_prompt`](#list_prompt)
  - [`move_blog_widget`](#move_blog_widget)
  - [`preview_generation`](#preview_generation)
  - [`remove_blog_widget`](#remove_blog_widget)
  - [`review_blog`](#review_blog)
  - [`set_blog_widget_image`](#set_blog_widget_image)
  - [`set_widget_prompt`](#set_widget_prompt)
  - [`update_blog_title`](#update_blog_title)
  - [`update_prompt`](#update_prompt)
- [White label content](#white-label-content)
  - [`add_entity_widget`](#add_entity_widget)
  - [`compose_entity_content`](#compose_entity_content)
  - [`copy_entity_widget`](#copy_entity_widget)
  - [`edit_entity_widget`](#edit_entity_widget)
  - [`fill_design_content`](#fill_design_content)
  - [`get_design_structure`](#get_design_structure)
  - [`get_entity_content`](#get_entity_content)
  - [`get_entity_history`](#get_entity_history)
  - [`get_entity_html`](#get_entity_html)
  - [`get_entity_widget_catalog`](#get_entity_widget_catalog)
  - [`list_entity`](#list_entity)
  - [`list_entity_batch`](#list_entity_batch)
  - [`move_entity_widget`](#move_entity_widget)
  - [`remove_entity_widget`](#remove_entity_widget)
  - [`revert_entity`](#revert_entity)
- [Description automation](#description-automation)
  - [`create_automation_rule`](#create_automation_rule)
  - [`get_automation_rule`](#get_automation_rule)
  - [`list_automation_rule`](#list_automation_rule)
  - [`list_automation_run`](#list_automation_run)
  - [`preview_automation_rule`](#preview_automation_rule)
  - [`update_automation_rule`](#update_automation_rule)

## Styling, products, labels and designs

### `assign_product_label`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Attach or detach one label on a batch of products, so they can be filtered in the admin grid.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `label_id` | integer | yes | Label id (see `list_label` / `create_label`). |
| `product_id` | array\<integer\> | yes | Product ids to label (see `find_product`). Min 1, max 500. |
| `action` | string enum (`attach`, `detach`) | no | Default `attach`. `detach` removes the label instead. |

**Returns:** `action`, `label_id`, `label_name`, `product_count` (number of products processed).

**Careful:** strict ownership — the label and every product must belong to the eshop, and a single foreign or unknown product id fails the whole call (no silent partial writes); the error lists the offending ids. Idempotent — attaching an already-attached label, or detaching an unattached one, is a no-op.

### `create_label`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Create a label on an eshop, used to organize products (and categories/blogs) in the Pobo admin grid.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `name` | string | yes | Min 2, max 255 chars. |
| `color` | string | no | Hex color, pattern `^#[0-9A-Fa-f]{6}$`. Default `#3B82F6`. |
| `description` | string | no | Max 1000 chars. |

**Returns:** `action` (`created` or `existing`), `id`, `name`, `color`.

**Careful:** idempotent by name within the eshop — calling it again with the same name returns the existing label instead of creating a duplicate.

### `delete_asset`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Delete an AI-managed style asset of an eshop and regenerate its CDN CSS bundle.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `asset_id` | integer | yes | Asset id (see `list_asset` or the `push_asset_css` response). |

**Returns:** `action: "deleted"`, `id` (the deleted asset id).

**Careful:** destructive and irreversible — only assets with `source = AI` can be deleted here (manually uploaded or CLI assets are out of scope). Deletion triggers a live CDN regeneration.

### `fetch_eshop_page`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Server-side fetch of the HTML of one page of the caller's own eshop (homepage, a product page), used to extract design tokens (colors, fonts, border radii, button styles) before styling.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `path` | string | no | Must start with `/`. Max 2048 chars. Default `/`. |

**Returns:** the raw page HTML as text, truncated with a comment marker past 500,000 bytes.

**Careful:** not an open proxy — only the eshop's own host (and its www/non-www twin) may be reached; redirects are followed up to 3 hops with the host re-validated on every hop, and a redirect leaving the eshop domain aborts the fetch with an error.

### `find_product`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Resolve a batch of client-supplied product identifiers (EAN, product code, product URL, or product name) to Pobo products in one call — the typical entry point before labeling or status changes.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `identifier` | array\<string\> | yes | Each item max 2048 chars. Min 1, max 100 identifiers, freely mixed types. |
| `type` | string enum (`auto`, `ean`, `code`, `url`, `name`) | no | Default `auto`: detects per identifier — `http(s)`/`www` prefix = url, 8 or 13 digits = ean, otherwise exact code match with a name-search fallback. |

**Returns:** `result` — one entry per identifier, each with `identifier`, `type`, and `status` of `matched` (with `product`: id/name/code/url), `ambiguous` (with up to 5 `candidate` entries), or `not_found`.

### `get_content_html`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Get the rendered HTML of the Pobo widget content of one page (product, category or blog), clean of shop chrome (menus, footer, cookie bars) — the styling target for `push_asset_css`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `content` | string enum (`product`, `category`, `blog`) | yes | The `data-pobo-content` attribute value. |
| `page_id` | integer | yes | Pobo page id — the `data-pobo-page-id` attribute value. |
| `lang` | string enum (language codes) | no | Default `default`. |

**Returns:** `html` (truncated past 200,000 bytes with a comment marker), `widget_class` (distinct `widget-*` CSS classes found in the HTML), `lang`, `truncated` (boolean).

### `get_product_analytics`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Read-only performance analytics for one product: page views / add-to-cart / order totals, a before/after comparison around the first Pobo description detection, engagement (scroll depth funnel, reading time, top FAQ questions), and a content heatmap of which description sections visitors see, dwell on, and add to cart after.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `product_id` | integer | yes | Product id (see `find_product`). |
| `period` | string enum (`30d`, `60d`, `90d`) | no | Analysed period ending today. Default `90d`. |

**Returns:** `product_id`, `period`, `totals` (page_views, unique_visitors, add_to_cart_count, order_count summed over the period), `pobo_content`, `before_after_summary`, `engagement`, `section_map`, `meta`. The raw daily series is not returned, only period totals.

**Careful:** `cart_rate_percent` and the before/after change are correlations, not causality; order counts cover only orders where the customer viewed the product page first, and only on Shoptet; `null` values mean not enough data yet, not zero performance.

### `get_theming_contract`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Get the Pobo theming contract: the native `.widget-*` BEM blocks and the `--pobo-widget-*`/`--pobo-global-*` CSS variables with their default values — read this before writing SCSS for `push_asset_css`.

No parameters beyond the implicit authenticated request.

**Returns:** the theming contract structure (blocks and variables with defaults), or an error if the contract source (`generic.css`) could not be fetched.

### `list_asset`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

List the AI-managed style assets of an eshop.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `asset` — array of `id`, `name`, `type`, `size` (bytes), `updated_at` (ISO 8601).

**Careful:** only shows assets created via this MCP channel (`source = AI`) — assets from the admin UI or CLI are not visible here.

### `list_category`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Paginated list of categories of an eshop with the same filters as the admin category grid — mainly used to resolve a category name to its id for `list_product`'s `category_id` filter.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `filter` | string enum (`all`, `without_description`, `edited`, `favourite`, `waiting_for_approval`, `recently_edited`) | no | Default `all`. `waiting_for_approval` = status draft; `recently_edited` orders the full list. |
| `query` | string | no | Full-text search in category name. Max 255 chars. |
| `limit` | integer | no | Min 1, max 100. Default 50. |
| `page` | integer | no | Min 1. Default 1 (1-based). |

**Returns:** `category` — array of `id`, `name`, `url`, `status`, `is_visible`, `is_favourite`, `has_content`, `product_count`; plus `total`, `page`, `limit`.

### `list_design`

**Server:** merchant (`/mcp/client`) and white label (`/mcp/whitelabel`)
**Costs credits:** no

List the designs (widget templates) an eshop can use as a prompt's `design_id` — public Pobo templates plus the eshop's own custom templates.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `design` — array of `id`, `name`, `type`, `widget_count`.

### `list_design_widget`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

List the widgets of a design in display order — enough for an agent to target `set_widget_prompt` by id.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `design_id` | integer | yes | Design id (see `list_design`). |

**Returns:** `design_id`, `design_name`, `widget` — the design's widget summary (id, position, widget name); `id` here is the `design_widget_id` used by `set_widget_prompt`.

**Careful:** this is a light listing only (id/position/name) — the fully rendered widget HTML used for the admin UI preview is not available through this tool.

### `list_eshop`

**Server:** merchant (`/mcp/client`) and white label (`/mcp/whitelabel`)
**Costs credits:** no

List the active eshops of the authenticated user — the entry point for every other tool's `eshop_id`.

No parameters beyond the implicit authenticated request.

**Returns:** `eshop` — array of `id`, `url`, `platform`, limited to active eshops, ordered by URL.

### `list_label`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

List the labels of an eshop.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `label` — array of `id`, `name`, `color`, `description`, `is_active`, `product_count` (usage count among products).

### `list_product`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Paginated list of products of an eshop with the same filters as the admin product grid — closes the loop of filtering a batch and then acting on it (e.g. with `set_product_status`) without hand-feeding product ids.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `filter` | string enum (`all`, `without_description`, `edited`, `favourite`, `waiting_for_approval`, `recently_edited`, `most_visited`, `most_added_to_cart`) | no | Default `all`. `without_description` = no Pobo content yet, `edited` = has Pobo content, `waiting_for_approval` = status draft. |
| `query` | string | no | Full-text search in product name, code and EAN. Max 255 chars. |
| `label_id` | array\<integer\> | no | Filter by label ids (see `list_label`). |
| `category_id` | array\<integer\> | no | Filter by category ids. |
| `brand_id` | array\<integer\> | no | Filter by brand ids. |
| `is_visible` | string enum (`all`, `visible`, `hidden`) | no | Default `all`. |
| `limit` | integer | no | Min 1, max 100. Default 50. |
| `page` | integer | no | Min 1. Default 1 (1-based). |

**Returns:** `product` — array of `id`, `name`, `code`, `url`, `status`, `is_visible`, `is_favourite`, `has_content`; plus `total`, `page`, `limit`.

### `push_asset_css`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Create or replace the eshop's AI style asset from SCSS source overriding the theming contract's variables. The server compiles, security-checks and deploys it to the eshop's CDN bundle.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `name` | string | yes | Max 255 chars. Human-readable asset name shown in the merchant admin. |
| `scss` | string | yes | Max 262,144 bytes (256 KB). No `@import`. `url(...)` only as https on an allowlisted CDN host (Pobo CDN, platform asset CDN), a relative path, or a `data:image` URI. |

**Returns:** `action` (`created` or `updated`), `id`, `name`, `size` (bytes), `cdn_url`.

**Careful:** each eshop has at most one AI style asset — pushing always replaces the previous content (upsert), it does not create a second asset. Compilation errors and forbidden CSS constructs (`@import`, `expression()`, `javascript:`, disallowed `url(...)` origins) are rejected with an error before anything is written.

### `screenshot_eshop_page`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Take a browser-rendered screenshot of a page of the caller's own eshop — the visual verification step after `push_asset_css`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `path` | string | no | Must start with `/`. Max 2048 chars. Default `/`. |
| `viewport` | string enum (`desktop`, `mobile`) | no | Default `desktop` (1520×1080). `mobile` is 475×812 rendered at 2× scale (950×1624 image). |
| `selector` | string | no | Max 256 chars. CSS selector to capture just that element (e.g. `#pobo-all-content` for the Pobo widget area) instead of the full viewport. |

**Returns:** the screenshot as an inline image, plus — best-effort — a public CDN URL to the same image as a text block (upload failure never fails the tool; the inline image is still returned).

**Careful:** screenshots larger than 3,000,000 bytes are rejected with an error suggesting the `selector` argument to narrow the capture; the target host is always the eshop's own stored URL, only a path is accepted as input.

### `set_product_status`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Switch the status of a batch of products between `ready` (approved for export) and `draft` (waiting for approval).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `product_id` | array\<integer\> | yes | Product ids to update (see `find_product`). Min 1, max 500. |
| `status` | string enum (`ready`, `draft`) | yes | Target status. |

**Returns:** `status` (the status applied), `product_count` (number updated).

**Careful:** restricted in both directions — only `ready`/`draft` can be set, and only products currently in `ready` or `draft` can be switched; products in other pipeline states (`review`, `generate`) are refused. A single foreign, unknown, or non-switchable product id fails the whole call with no partial writes; the error lists the offending ids. Idempotent — setting the current status again is a no-op.
## Blog, prompts, generation and diagnostics

### `add_blog_product_carousel`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Inserts a product carousel into a blog article, filled from real shop data (photo, name, price, link). Products resolve by Pobo id or a pasted public product URL.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `product_id` | array of integer | no | Pobo product ids (see `find_product`); max 10 items. |
| `product_url` | array of string | no | Public product URLs on the eshop; max 10 items. |
| `position` | integer | no | 1-based position in the article; omit to append. Min 1. |
| `lang` | string (enum) | no | Content language; omit for the eshop default. |

**Returns:** `blog_id` and a `summary` string describing what was inserted.

**Careful:** at least one of `product_id` or `product_url` must be given — the tool errors if both are empty; this is enforced in `handle()`, not in the schema.

### `add_blog_widget`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Inserts a widget into a blog article filled with content the agent authors itself (the Claude-as-author path — deterministic filler, no server-side LLM call). Content is keyed by the roles from `get_blog_widget_catalog`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog` / `create_blog`). |
| `widget_id` | integer | yes | Template widget id from `get_blog_widget_catalog`. |
| `content` | object | yes | Values keyed by role, e.g. `{"paragraph": "<h2>Heading</h2><p>Body</p>"}`. Repeatable widgets: `{"items": [{role: value}, ...]}` with distinct items. |
| `image_url` | array of string | no | CDN image URLs assigned sequentially to the widget's image slots. |
| `position` | integer | no | 1-based position in the article; omit to append. Min 1. |
| `lang` | string (enum) | no | Content language; omit for the eshop default. |

**Returns:** `blog_id`, `widget_instance_id`, and the widget's 1-based `position`.

### `convert_blog_widget`

**Server:** merchant (`/mcp/client`)
**Costs credits:** yes — only when `image_source=ai`: 1 photo × 2 credits = 2 credits. The first call without `cost_confirmed=true` returns a `confirm_required` quote instead of executing; the agent must relay the price and re-call with `cost_confirmed=true`. Sources `stock`, `uploaded`, `image_bank` are free.

Converts a text widget of a blog article into an image+text layout: its texts carry over verbatim and a photo is resolved from the chosen source.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `widget_instance_id` | integer | yes | Widget instance id from `get_blog_content`. |
| `image_query` | string | yes | Photo search query (`stock`) or generation prompt (`ai`). Max 300 chars. |
| `image_side` | string (enum: `left`, `right`) | no | Where the photo should sit. Default `right`. |
| `image_source` | string (enum: `stock`, `ai`, `uploaded`, `image_bank`) | no | Photo source. Default `stock`. |
| `lang` | string (enum) | no | Content language; omit for the eshop default. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted AI photo cost. Default false. |

**Returns:** `blog_id`, a `summary` string, and `widget` (a snapshot of the article's widgets).

**Careful:** replaces the widget's layout (text preserved, but the shape changes from text-only to image+text); with `image_source=ai` it is paid and refused with `Not enough credits...` if the eshop's balance is below the quoted cost.

### `create_blog`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Creates a new empty blog article in Pobo (a manual article, not paired with the shop platform). Returns `blog_id` for use with the other blog tools.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `name` | string | yes | Article title. Max 255 chars. |

**Returns:** `blog_id` and `name`.

### `create_prompt`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Creates a custom AI generation prompt profile on an eshop. Always creates `type=custom` (the MCP channel can never create gallery-type prompts). Optionally links a design; per-widget instructions are set separately via `set_widget_prompt`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `name` | string | yes | Profile name shown in the merchant admin. Min 5, max 255 chars. |
| `prompt` | string | yes | The generation prompt text — general instructions for the whole product description. Min 20, max 1,000,000 chars. |
| `design_id` | integer | no | Optional design (template) the prompt generates into (see `list_design`). Required before per-widget instructions can be set. |
| `icon_category_id` | integer | no | Optional icon category restricting AI icon selection. |

**Returns:** `action: "created"` plus the prompt detail — `id`, `name`, `prompt`, `design_id`, `icon_category_id`, `icon_mode`, `widget_prompt` (array of `{design_widget_id, prompt}`), `created_at`, `updated_at`.

### `delete_prompt`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Deletes a custom prompt profile from an eshop. Its per-widget instructions are removed with it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `prompt_id` | integer | yes | Prompt id (see `list_prompt`). |

**Returns:** `action` (`"deleted"` if the prompt belonged to no other eshop, `"detached"` if it is shared and only the link to this eshop is removed) and `id`.

**Careful:** destructive — when `action` is `"deleted"` the prompt itself is soft-deleted, not just unlinked.

### `edit_blog_widget`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Rewrites the text content of an existing widget in a blog article with values the agent authors, keyed by role. A deterministic, structure-preserving rewrite: only the target language key of matched value maps changes — other translations, images and non-matched nodes stay untouched.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `widget_instance_id` | integer | yes | Widget instance id from `get_blog_content`. |
| `content` | object | yes | New values keyed by role, e.g. `{"paragraph": "<p>New body</p>"}`. Repeatable widgets: `{"items": [{role: value}, ...]}`. |
| `lang` | string (enum) | no | Language key to rewrite; omit for the default content. Other languages stay untouched. |

**Returns:** `blog_id`, `widget_instance_id`, `updated: true`.

**Careful:** returns an error if no editable field matched the given content roles — check `get_blog_widget_catalog` for the widget's actual roles.

### `generate_blog_article`

**Server:** merchant (`/mcp/client`)
**Costs credits:** yes — 1 credit per article, always. When `photo_source=ai`, add 2 credits per AI photo (`image_count × 2`); `image_count` is required in that case so the cost can be quoted, and the first call without `cost_confirmed=true` returns a `confirm_required` quote that must be relayed to the user before re-calling with `cost_confirmed=true`. Non-AI photo sources do not require confirmation, but the 1-credit article cost still applies and the call is refused if the eshop cannot cover it. Free alternative: author the texts with `add_blog_widget`.

Generates a whole blog article server-side from a brief. Asynchronous — dispatches a job on the `generator` queue; poll with `get_blog_generate_status`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog` / `create_blog`). |
| `brief` | string | yes | What the article should be about. Min 5, max 5000 chars. |
| `normpages` | integer | no | Article length in standard pages. Min 1, max 10, default 3. |
| `structure_mode` | string (enum: `ai`, `template`) | no | `ai` lets the planner pick the structure; `template` follows a blog design (requires `design_id`). Default `ai`. |
| `design_id` | integer | no | Blog design id; required when `structure_mode=template`. Min 1. |
| `photo_source` | string (enum: `none`, `ai`, `stock`, `uploaded`, `product`) | no | Photo strategy. `stock`/`uploaded`/`product`/`none` are free; `ai` is paid. Default `stock`. |
| `image_count` | integer | no | Number of image widgets; omit to size automatically. Required for `photo_source=ai`. Min 0, max 10. |
| `product_id` | array of integer | no | Related Pobo product ids to weave into the article; max 10. |
| `product_url` | array of string | no | Related public product URLs; max 10. |
| `inspiration_url` | array of string | no | Pages to distill as factual inspiration; max 3. |
| `tone` | string | no | Optional tone of voice. Max 100 chars. |
| `remove_exist_widget` | boolean | no | Replace the current article content instead of appending. Default false. |
| `lang` | string (enum) | no | Article language; omit for the eshop default. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost (needed for `photo_source=ai`). Default false. |

**Returns:** `job_id`, `blog_id`, `status`, and a message pointing to `get_blog_generate_status` for polling.

**Careful:** `remove_exist_widget=true` replaces the article's existing content instead of appending to it. The tool also refuses to run when the AI-content-generation kill switch is off, and when `structure_mode=template` without `design_id`.

### `get_blog_content`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Numbered snapshot of one blog article: title plus the ordered widget list (position, widget_instance_id, name, text preview, image slot count). Call before editing widgets so ids and positions resolve without guessing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |

**Returns:** `blog_id`, `name`, and `widget` (the ordered snapshot array).

### `get_blog_generate_status`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Polls a running article generation job started by `generate_blog_article`. Steps: `preparing`, `writing`, `selecting_photos`, `composing`, `done`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id the job was started for. |
| `job_id` | integer | yes | Job id returned by `generate_blog_article`. |

**Returns:** `job_id`, `blog_id`, `status`, `progress`, `step`, `widgets_created`, `error_message`, `title` (from the generated response, when available).

### `get_blog_widget_catalog`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Lists the widgets an eshop can place into a blog article, with their AI slot schema: text roles (role, hint, max_length), image slot count and item container count.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `widget` — array of widget slot descriptions (one per placeable widget).

### `get_credit`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Credit balance of an eshop plus the price list of paid operations. Meant to be checked before offering a paid action (`generate_blog_article`, an AI photo) so the merchant is not promised something the balance cannot cover.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `credit` (`remaining`, `total_added`, `total_spent`) and `price` (the price list of paid operations).

### `get_export_status`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Recent exports of content to the eshop platform: progress, completion, and whether an export failed. A failed run is never retried, so its products stay unexported — the usual reason approved content is not visible on the eshop.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `platform`, `accepts_new_export` (bool), and `export` (array of `{id, created_at, finished_at, is_complete, is_running, export_total, export_complete, failed_at}`).

### `get_generation`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Detail of one AI content generation run of the merchant's eshop: the settings it ran with, what the web research found per provider (with a coverage summary), and the model's answer.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `job_id` | integer | yes | Generation run id (see `list_generation`). |
| `include_response` | boolean | no | Include the raw model answer. Default true. |
| `search` | string | no | Case-insensitive keyword. When given, the model answer comes back as up to 5 match windows (600 chars of context each) instead of full text. Min 2, max 255 chars. |

**Returns:** `generation` — settings summary, per-provider research with a coverage summary, and the model answer (unless `include_response=false` or narrowed by `search`).

**Careful:** the assembled prompt sent to the model (`gpt_text_prompt`) is never returned — only Pobo staff can see it on the internal server. `research_status.reason="no_ean"` means research never ran for lack of an EAN; `is_complete=false` without a `skip_reason` means the run is stuck, not failed.

### `get_preview_status`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Polls dry-run previews started by `preview_generation`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `token` | array of string | yes | Tokens returned by `preview_generation`, max 3; each max 64 chars. |

**Returns:** `preview` — one entry per token, `status` of `pending`, `complete`, or `not_found` (expired after 30 minutes, never existed, or belongs to another eshop). A complete entry includes the rendered HTML, generated SEO fields, and any error.

### `get_product_research`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

What the web research found for one of the merchant's products in its recent generation runs, per provider, plus a coverage summary of which facts turned up and which sources were cited.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `product_id` | integer | yes | Product id (see `find_product` / `list_product`). |
| `limit` | integer | no | How many recent generation runs with research to return, 1-10. Default 3. |

**Returns:** `research` — array of `{job_id, created_at, search_web, search_model, provider: [{provider, mode, confidence, response}], coverage}`.

### `get_prompt`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Full detail of a custom prompt profile: prompt text, linked design and explicit per-widget instructions.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `prompt_id` | integer | yes | Prompt id (see `list_prompt`). |

**Returns:** `id`, `name`, `prompt`, `design_id`, `icon_category_id`, `icon_mode`, `widget_prompt` (array of `{design_widget_id, prompt}`), `created_at`, `updated_at`.

### `get_prompt_history`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Lists historical versions of a custom prompt's text (newest first) — useful to check what was there before overwriting it. Per-widget prompts are not versioned.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `prompt_id` | integer | yes | Prompt id (see `list_prompt`). |
| `limit` | integer | no | Maximum number of snapshots to return. Min 1, max 50, default 10. |

**Returns:** `prompt_id` and `history` — array of `{id, prompt, user, created_at}`, newest first.

### `list_blog`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Lists the blog articles of an eshop (id, name, widget count, platform pairing). Use the id as `blog_id` for the other blog tools.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `query` | string | no | Optional name search. |
| `limit` | integer | no | Page size. Min 1, max 50, default 20. |
| `page` | integer | no | 1-based page number. Min 1, default 1. |

**Returns:** `blog` — array of `{id, name, widget_count, is_visible, platform_article_id}` — plus `total`, `page`, `limit`.

### `list_generation`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Lists the AI content generation runs of an eshop, newest first, with their settings summary, completion state and research row count. Filter by `product_id` to see every run of one product, or `only_incomplete` to find what is stuck.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `product_id` | integer | no | Only runs of this product. |
| `query` | string | no | Product name fragment the run was started with. Max 255 chars. |
| `date_from` | string | no | Only runs created at or after this date (Y-m-d or ISO 8601). |
| `date_to` | string | no | Only runs created at or before this date (Y-m-d or ISO 8601). |
| `only_incomplete` | boolean | no | Only runs that never completed — stuck or failed generations. Default false. |
| `limit` | integer | no | Rows per page, 1-100. Default 20. |
| `page` | integer | no | Page number, from 1. Default 1. |

**Returns:** `generation` — array of run summaries (`id`, `product_id`, `name`, timestamps, `is_complete`, `is_running`, `skip_reason`, `design_id`/`design_name`, `prompt_text_id`/`prompt_text_name`, research settings, `research_row_count`) — plus `meta` (`total`, `page`, `per_page`, `last_page`).

### `list_prompt`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Lists the custom AI generation prompt profiles of an eshop (id, name, linked design, per-widget prompt count). Gallery-backed prompt types (design/image/scss) are invisible on this channel.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `prompt` — array of `{id, name, design_id, design_name, widget_prompt_count, updated_at}`.

**Careful:** the full prompt text is deliberately excluded (it can be up to 1 MB) — use `get_prompt` to read it.

### `move_blog_widget`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Moves a widget of a blog article to another 1-based position and resequences the article.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `widget_instance_id` | integer | yes | Widget instance id from `get_blog_content`. |
| `position` | integer | yes | Target 1-based position. Min 1. |

**Returns:** `blog_id`, `widget_instance_id`, `position`, and `widget` (a snapshot of the article's widgets).

### `preview_generation`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Dry-runs a product description generation against a design and a prompt without saving anything — no content is written, no credits are charged, even when web research or AI fact-finder providers are used. Queues one preview per product (max 3) and returns tokens; poll with `get_preview_status`. Use it to test a prompt before `update_prompt` makes it permanent.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `design_id` | integer | yes | Design (widget template) to generate into (see `list_design`). |
| `product_id` | array of integer | yes | Products to preview, 1-3 (see `find_product` / `list_product`). |
| `prompt` | string | no | The prompt text being tested. Omit to preview with the design defaults. Max 1,000,000 chars. |
| `paragraph_length` | integer | no | Target paragraph length in characters, 50-500. |
| `generate_short_description` | string (enum: `never`, `when_missing`, `always`) | no | Whether the preview also writes a short description. Default `never`. |
| `generate_seo_meta` | boolean | no | Generate SEO title/description in the preview. |
| `generate_entity_name` | boolean | no | Generate an improved product name in the preview. |
| `use_ai_profile` | boolean | no | Include the eshop AI profile (tone of voice) in the prompt. |
| `use_serp_context` | boolean | no | Fetch Google/SerpAPI context. Free in a dry-run. |
| `use_web_research` | boolean | no | Fetch web research by EAN. Free in a dry-run. |
| `search_web` | array of string | no | Restrict web research to these domains, e.g. `["https://yoggies.cz"]`. Max 20. |
| `search_model` | array of string (enum: `openai`, `gemini`, `perplexity`, `anthropic`) | no | Fact-finders to run. Free in a dry-run. Max 4. |

**Returns:** `preview` — one entry per `product_id` (`{product_id, token, status}` or an error entry when the product does not belong to the eshop), `poll_with: "get_preview_status"`, `expires_in_minutes: 30`.

**Careful:** refused with an error when the AI-content-generation kill switch is off — this can look like a bug rather than a maintenance window.

### `remove_blog_widget`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Removes a widget from a blog article and resequences the remaining positions.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `widget_instance_id` | integer | yes | Widget instance id from `get_blog_content`. |

**Returns:** `blog_id`, `widget_instance_id`, `removed: true`.

**Careful:** deletes the widget outright — there is no undo tool in this set.

### `review_blog`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Renders a blog article and returns its quality report (rendered size, widget count, empty slots). Use as the final check after authoring or editing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `lang` | string (enum) | no | Language to render; omit for the default. |

**Returns:** `lang`, `html` (rendered output), `byte_size`, `byte_limit`, `over_limit` (bool), `widget_count`, `empty_text_slot`, `empty_image_slot`, `warning` (array of human-readable issues, e.g. over the size limit or empty slots).

### `set_blog_widget_image`

**Server:** merchant (`/mcp/client`)
**Costs credits:** yes — only when `image_source=ai`: 2 credits per photo slot replaced (1 slot if `image_index` is given, otherwise every slot in the widget). The first call without `cost_confirmed=true` returns a `confirm_required` quote; re-call with `cost_confirmed=true` after the user confirms. Sources `stock`, `uploaded`, `image_bank` are free.

Replaces the photo(s) of an existing widget in a blog article without touching its texts.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `widget_instance_id` | integer | yes | Widget instance id from `get_blog_content`. |
| `image_query` | string | yes | Photo search query (`stock`) or generation prompt (`ai`). Max 300 chars. |
| `image_source` | string (enum: `stock`, `ai`, `uploaded`, `image_bank`) | no | Photo source. Default `stock`. |
| `image_index` | integer | no | 1-based photo slot to replace; omit to replace all of them. Min 1. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted AI photo cost. Default false. |

**Returns:** `blog_id`, `widget_instance_id`, and a `summary` string.

**Careful:** replaces the existing photo(s) outright (no side-by-side compare); with `image_source=ai` it is paid and refused if the eshop's credit balance cannot cover the quoted cost.

### `set_widget_prompt`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Replaces ALL explicit per-widget instructions of a custom prompt profile in one call. Each instruction targets one widget of the prompt's design and overrides the general prompt for that widget.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `prompt_id` | integer | yes | Prompt id (see `list_prompt`). The prompt must have a `design_id`. |
| `widget_prompt` | array of object `{design_widget_id: integer (required), prompt: string (required, max 50000)}` | yes | The complete new set of per-widget instructions — replaces all existing ones. Empty array clears them. Max 30 items. |

**Returns:** `action: "replaced"`, `prompt_id`, `widget_prompt_count`.

**Careful:** replace-all semantics — every call defines the *complete* set; anything omitted is deleted, not left alone. Fails if the prompt has no `design_id` (set one first via `update_prompt`).

### `update_blog_title`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Sets the title (and optionally the SEO title) of a blog article.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `blog_id` | integer | yes | Blog id (see `list_blog`). |
| `title` | string | yes | New article title. Max 255 chars. |
| `seo_title` | string | no | Optional SEO title. Max 255 chars. |
| `lang` | string (enum) | no | Language of the title; omit for the default. |

**Returns:** `blog_id`, `changed` (bool), `summary`.

### `update_prompt`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Partial update of a custom prompt profile — only the fields sent are changed (unlike the REST endpoint, which requires the full payload).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `prompt_id` | integer | yes | Prompt id (see `list_prompt`). |
| `name` | string | no | New profile name. Min 5, max 255 chars. |
| `prompt` | string | no | New generation prompt text (full replacement of the text). Min 20, max 1,000,000 chars. |
| `design_id` | integer | no | New design id (see `list_design`), or null to unlink. |
| `icon_category_id` | integer | no | New icon category id, or null to unlink. |

**Returns:** `action: "updated"`, `widget_prompt_cleared` (bool), plus the prompt detail (`id`, `name`, `prompt`, `design_id`, `icon_category_id`, `icon_mode`, `widget_prompt`, `created_at`, `updated_at`).

**Careful:** changing `design_id` clears all existing per-widget instructions of the prompt (they referenced widgets of the previous design) — `widget_prompt_cleared` in the response confirms whether this happened. Errors if no field is sent to change.
## White label content

All tools in this section live on the white label server (`/mcp/whitelabel`) and are gated to eshops with `platform = b2b` — calling any of them against an eshop on another platform returns "Eshop not found."

### `add_entity_widget`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Adds a widget template to one entity or up to 50 at once, either with the same content for every entity (`content`) or different content per entity (`content_per_entity`). A non-repeatable widget holds one item, so five FAQ questions require five calls.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | array of string | yes | Host ids of the entities to add the widget to, 50 at most. Each string max 64 chars. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `widget_id` | integer | yes | Widget template to place (see `get_entity_widget_catalog`). |
| `content` | object | no | Texts keyed by role, e.g. `{"question": "…", "answer": "<p>…</p>"}`. Used for every entity not named in `content_per_entity`. A repeatable widget carries its values in `content.item`. |
| `content_per_entity` | object | no | Per-entity content: `{"<entity_id>": {"<role>": "text"}}`. Overrides `content` for the entities it names. |
| `image` | array of string | no | Up to 20 own https image URLs, filled into image slots in order. Each must be an https URL, max 255 chars. |
| `position` | integer | no | Where to place the widget, 1 = first. Omitted appends at the end. |

At least one of `content` or `content_per_entity` is required; if both are empty the call errors.

**Returns:** `batch_id`, a `summary` sentence, and a per-entity `entity` array with `status` (`added`, `error`, or `not_found`), `widget_instance_id` and `position` on success. If `content_per_entity` names an id not present in `entity_id`, the response adds `unmatched_content_key` and a `warning` — those entities silently received the shared `content` instead.

**Careful:** an entity with no applicable content (neither in `content` nor `content_per_entity`) is reported as `error`, not silently skipped, but the batch as a whole still runs for the other entities. A widget instance that gets text longer than its `max_length` hint is written anyway (length is advisory) and flagged with a `warning` per entity.

### `compose_entity_content`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Builds the whole description of one entity in a single call — the agent picks widgets from `get_entity_widget_catalog`, orders them, and writes their texts, without any design template. Creates the entity if it does not exist yet.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | yes | Own id of the entity, max 64 chars. Created automatically if unknown. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `widget` | array of object | yes | Widgets in display order, 30 at most. Each item: `{widget_id, content, image}` — `content` holds texts by role, `image` holds up to 20 items, each a plain https URL string or `{url, alt}`. |
| `mode` | string | no | `replace` (default) or `append`. |
| `title` | string | no | Entity name used by `list_entity` search, max 255 chars. Only applied on creation or when the entity already exists (title is then also updated). |

**Returns:** `entity_id`, `entity_created` (bool), `widget_count`, `mode`, `batch_id`, and a `message`. Includes `warning` when any widget's content exceeded its length hint.

**Careful:** `mode=replace` is the default and force-deletes every existing widget of the entity before writing the new ones. The whole composition is validated up front — if any widget fails validation, nothing is written at all (no partial replace that could leave the entity with a half-empty description). `mode=append` instead adds the new widgets after the current content and does not delete anything.

### `copy_entity_widget`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Copies widgets from one source entity to up to 50 target entities, optionally restricted to specific widget instances of the source.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `source_entity_id` | string | yes | Host id of the entity to copy FROM, max 64 chars. |
| `target_entity_id` | array of string | yes | Host ids of the entities to copy TO, 50 at most. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page` — shared by source and targets. |
| `mode` | string | no | `append` (default) keeps target content and adds to it; `replace` throws the target content away first. |
| `widget_instance_id` | array of integer | no | Copy only these widgets of the source (see `get_entity_content`), 50 at most. Omit to copy all of them. |

**Returns:** `batch_id`, `mode`, a `summary` sentence, and a per-target `entity` array with `status` (`copied` or `error`) and `widget_copied` count.

**Careful:** `mode=replace` force-deletes the target entity's existing widgets before copying in the source's widgets — the tool description explicitly tells the caller to state which mode it is about to use before calling. A target that is the same entity as the source is rejected per-entity with an error, not treated as a no-op.

### `edit_entity_widget`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Rewrites the texts and/or media of widgets that already exist — either one specific widget instance, or every instance of one template across up to 50 entities.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | array of string | yes | Host ids of the entities to edit, 50 at most. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `content` | object | no | New texts keyed by role. A repeatable widget carries its values in `content.item`. Only the roles sent are changed. |
| `image` | object | no | Photo slots to change, keyed by the slot index from `get_entity_content`, e.g. `{"1": {"url": "https://…", "alt": "…"}}`. `url` and `alt` are independent — sending `alt` alone describes the photo without replacing it. |
| `icon` | object | no | Icon slots, keyed the same way. Icons take `alt` only; sending a `url` for an icon is rejected as an error. |
| `widget_instance_id` | integer | no | Edit exactly this one widget. Requires exactly one `entity_id`. Mutually exclusive with `widget_id`. |
| `widget_id` | integer | no | Edit every instance of this template on the given entities. Mutually exclusive with `widget_instance_id`. |

Exactly one of `widget_instance_id` or `widget_id` must be given, and at least one of `content`/`image`/`icon` must be non-empty.

**Returns:** `batch_id`, a `summary`, and a per-entity `entity` array with `status` (`edited`, `error`, or `not_found`), `widget_changed` and `field_written` counts. Includes `warning` when content exceeded its length hint.

**Careful:** a slot index outside the widget's actual image/icon count is rejected as an error rather than silently ignored. If nothing sent actually matches anything on the widget, the call reports `error` ("Nothing was written") rather than a false `edited`. Only the roles/slots explicitly sent are touched — everything else on the widget is left as is.

### `fill_design_content`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Fills a design template with the agent's own texts, images and icons and returns the rendered HTML. No AI runs on the server. Call `get_design_structure` first to learn the slots.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | yes | Own id of the item, max 64 chars. Registered automatically on first use. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `design_id` | integer | yes | Design template to fill (see `list_design`, `get_design_structure`). |
| `widget` | array of object | yes | Widgets to fill, 50 at most. Each: `{design_widget_id, content, image, icon}` — `content` holds texts by role, a repeatable widget carries its values in `content.item` as an array of objects (max 20 entries); `image` and `icon` are own https URLs (max 20 each), filled in the order given, max 255 chars per URL. |
| `lang` | string | no | Enum of `AllowedLanguageEnum` values. Which language of the rendered HTML to return; content is written into every language of the eshop regardless. |
| `title` | string | no | Max 255 chars. Name of the item, stored on the entity when it is registered by this call. Ignored for an entity that already exists. |

**Returns:** `entity_id`, `entity_type`, `design_id`, `widget_created` (count), `html` (keyed by the requested language, or all languages), and `warning` when any text exceeded its `max_length` hint.

**Careful:** every call REPLACES the entity's whole content, not just the widgets sent — confirmed in code (`EntityDesignFillService::fill()`, and the class docblock states the tool is deliberately not marked idempotent because "volání nahrazuje celý obsah entity"). A widget template not included in `widget` is not rendered at all, so a second call with a shorter widget list drops whatever was left out the first time; there is no append mode. The entity is registered automatically (`WhiteLabel::registerEntity()`) if `entity_id`/`entity_type` do not resolve to an existing one — this tool never errors with "entity not found."

### `get_design_structure`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Describes what a design template accepts: for every widget its `design_widget_id`, roles with format and `max_length`, item counts for repeatable widgets, and image/icon slot counts. Read before `fill_design_content`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `design_id` | integer | yes | Design template id (see `list_design`). |

**Returns:** the template's widget/slot description (via `EntityDesignFillService::describe()`) — widgets that cannot be filled are still listed, with a reason.

### `get_entity_content`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Reads a numbered snapshot of what is written on one entity: ordered widget list with position, `widget_instance_id`, `widget_id`, template name, image slot count and texts keyed by role.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | yes | Host id of the entity (see `list_entity`), max 64 chars. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `lang` | string | no | Enum of `AllowedLanguageEnum` values. Omit for the default language. |

**Returns:** `entity_id`, `entity_type`, `title`, `design_id`, `lang`, and `widget` (the ordered snapshot). Repeatable widgets list their roles once per item, in document order.

### `get_entity_history`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Lists the versions of one entity, newest first — the input to `revert_entity`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | yes | Host id of the entity (see `list_entity`), max 64 chars. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `limit` | integer | no | How many versions to return, 50 at most (default 20). |

**Returns:** `entity_id`, `entity_type`, and `version`, an array of `{version_id, created_at, source, tool, user_id, batch_id, widget_count}`. Each version is the state BEFORE the write that produced it, so reverting to it undoes that write.

### `get_entity_html`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Reads the current rendered HTML of a white label entity as stored in Pobo right now — the same render path used by `fill_design_content`, not regenerated.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | yes | The host's own id of the item, max 64 chars. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `lang` | string | no | Enum of `AllowedLanguageEnum` values. Omit for the default language. |

**Returns:** `entity_id`, `entity_type`, `html` (keyed by language, or just the requested one). Errors with "Entity has no content yet" if the entity has never been filled.

### `get_entity_widget_catalog`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Lists widget templates that can be placed on a white label entity, with their slots: text roles (role, hint, max_length), image slot count and item container count. Filtered to the ~40 (of ~95 public) templates that declare AI-fillable roles.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |

**Returns:** `widget`, an array of widget slot descriptions (via `WidgetSchemaService::describeSlots()`).

### `list_entity`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Finds white label entities of an eshop. Without a query it lists the newest ones; with a query it searches the host id, entity name and widget texts at once (case/diacritics-insensitive), and says which matched.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `query` | string | no | Searched in host entity id, entity name and widget texts. Max 255 chars. |
| `type` | string | no | Restrict to one of `product`, `category`, `blog`, `page`. |
| `limit` | integer | no | How many entities to return, 50 at most (default 20). |
| `offset` | integer | no | How many entities to skip, for paging. |
| `lang` | string | no | Language of returned names/snippets. Enum of `AllowedLanguageEnum` values. Omit for default. |

**Returns:** `total`, `limit`, `offset`, and `entity` (array of `{entity_id, type, title, design_id, widget_count, updated_at}`, plus `match` info when a query was given). If the eshop holds more than `EntityContentService::MAX_SEARCHABLE_WIDGET` widgets, widget text is not searched, and the response adds `content_searched: false` with a `note` explaining only ids/names were searched.

### `list_entity_batch`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Lists recent bulk write runs on this eshop, newest first — used to find a run to undo with `revert_entity`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `tool` | string | no | Only runs made by this tool, e.g. `"add_entity_widget"`. Max 64 chars. |
| `limit` | integer | no | How many runs to return, 50 at most (default 20). |

**Returns:** `batch`, an array of `{batch_id, tool, source, user_id, entity_count, created_at, entity_id_sample}` — `source` is the channel that made the run, `entity_count` is how many distinct entities it touched, and `entity_id_sample` is up to 5 of their ids.

### `move_entity_widget`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Moves a widget to another position inside its own entity; the rest of the widgets shift to close the gap.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | yes | Host id of the entity, max 64 chars. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `widget_instance_id` | integer | yes | Widget to move (see `get_entity_content`). |
| `position` | integer | yes | Where to put it, 1 = first. Minimum 1. |

**Returns:** `entity_id`, `widget_instance_id`, and `widget` — the full updated widget snapshot of the entity (default language) after the move.

### `remove_entity_widget`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Removes widgets from up to 50 entities, matched by one widget instance, by every instance of a template, or by a text query. Two-phase: the first call previews, the second (with `confirmed: true`) actually deletes.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | array of string | yes | Host ids of the entities to remove from, 50 at most. |
| `entity_type` | string | yes | One of `product`, `category`, `blog`, `page`. |
| `widget_instance_id` | integer | no | Remove exactly this one widget. Exactly one of the three match parameters is required. |
| `widget_id` | integer | no | Remove every instance of this template on the given entities. |
| `query` | string | no | Remove every widget whose text contains this (case/diacritics-insensitive). Max 255 chars. |
| `confirmed` | boolean | no | Pass true only after showing the preview to the merchant and getting agreement. |

**Returns (preview, `confirmed` not true):** `confirm_required: true`, `would_remove` (total widget count), `entity_affected`, a `sample` (up to 5 matched widgets with entity_id, widget_instance_id, widget_id, and up to 120 chars of text), and a `note` that nothing was removed yet.

**Returns (confirmed run):** `batch_id`, `widget_removed` (total), a `summary`, and per-entity `entity` array with `status: removed` and `widget_removed` count.

**Careful:** the first call with `confirmed` omitted or false performs no deletion regardless of how broad the filter is — this is a deliberate two-phase gate specifically because a typo in `query` could otherwise strip content from hundreds of entities in one call. The actual deletion (second call) is a soft delete of the matched widget rows (the row is marked deleted, not physically removed), but it is reversible via `revert_entity` using the returned `batch_id`, since a version snapshot is captured per entity before deleting.

### `revert_entity`

**Server:** white label (`/mcp/whitelabel`)
**Costs credits:** no

Puts entity content back to an earlier state — either one entity to a specific version, or every entity touched by one batch run.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `entity_id` | string | no | Host id of the entity to revert, max 64 chars. Give together with `entity_type` and `version_id`. |
| `entity_type` | string | no | One of `product`, `category`, `blog`, `page`. |
| `version_id` | integer | no | Version to go back to (see `get_entity_history`). |
| `batch_id` | string | no | Revert a whole run instead of one entity, max 26 chars. Mutually exclusive path with `entity_id`/`entity_type`/`version_id`. |

Either `batch_id` alone, or `entity_id` + `entity_type` + `version_id` together, must be given.

**Returns (batch revert):** `batch_id`, `entity_reverted` (count), `widget_restored` (count). Errors if no entity of the eshop was touched by that `batch_id`.

**Returns (single entity revert):** `entity_id`, `entity_type`, `version_id`, `widget_restored`.

**Careful:** reverting overwrites the entity's current widget content with the older version's content — the current state is saved as a new version first (so a revert is itself revertible), but the effect is still a full replace of what's there now.

## Description automation

### `create_automation_rule`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no (the rule itself is free to create; the nightly runs it schedules later spend credits per product generated)

Creates a scheduled automation that generates AI descriptions for products that still have none, running on its own cron schedule.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `name` | string | yes | Max 120 chars. Shown in the merchant admin. |
| `prompt_id` | integer | yes | Prompt profile defining HOW to generate (design, languages, SEO). See `list_prompt`. Must belong to this eshop and be a type that generates product text, or the call errors. |
| `schedule_cron` | string | yes | Cron expression in the eshop timezone, e.g. `"0 2 * * *"`. Must be valid (`ValidCronExpression`) and at most once per day. |
| `batch_size` | integer | yes | Max products per run. Min 1, max `config('automation.product.batch_size_max')` (default 500). Each product costs credits — this is the nightly ceiling. |
| `description` | string | no | Note for the merchant, max 2000 chars. |
| `category_id` | array of integer | no | Limit to these categories (must belong to the eshop). Omit for the whole catalogue. |
| `brand_id` | array of integer | no | Limit to these brands (must belong to the eshop). Omit for the whole catalogue. |
| `is_paused` | boolean | no | Defaults to true (created paused). Pass false to start it running immediately. |

**Returns:** `action: "created"` plus the full rule detail — `id`, `name`, `description`, `is_paused`, `priority`, `prompt` (`{id, name}`), `category` list, `brand` list, `scope` (`"whole catalogue"` or `"limited to the listed categories and brands"`), `batch_size`, `schedule_cron`, `schedule_timezone`, `last_run_at`, `next_run_at`, `consecutive_failure`, `created_at`.

**Careful:** the rule is created **paused by default** — `is_paused` defaults to `true` regardless of whether the caller set a schedule, so it will not run and spend credits until it is explicitly unpaused (via `update_automation_rule` with `is_paused: false`, or `is_paused: false` at creation). This is deliberate: an agent that just invented a cron expression is not also meant to start it running unattended. Once unpaused, the rule runs nightly on its own and spends credits per product generated, with no further confirmation per run.

### `get_automation_rule`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Full detail of one AI description automation, including its 5 most recent runs.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `rule_id` | integer | yes | Automation id (see `list_automation_rule`). |

**Returns:** the full rule detail (same shape as `create_automation_rule`'s response) plus `recent_run` (up to 5 most recent runs, each with `id`, `rule_id`, `rule_name`, `triggered_by`, `status`, `started_at`, `finished_at`, `candidate_count`, `job_created`, `skipped_no_credit`, `skipped_recently_generated`, `error_message`) and `auto_pause_warning` (true when `consecutive_failure > 0` — three failed nights in a row auto-pause the rule).

### `list_automation_rule`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Lists the scheduled AI description automations of an eshop.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `is_paused` | boolean | no | Filter: true for only paused automations, false for only active ones. Omit for all. |

**Returns:** `rule`, an array of rule details (same shape as `create_automation_rule`'s response, one per rule).

### `list_automation_run`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

History of automation runs across an eshop's automations, newest first.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `rule_id` | integer | no | Restrict to runs of this one automation (see `list_automation_rule`). |
| `limit` | integer | no | How many runs to return, max 100 (default 20). |

**Returns:** `run` (array of run details — same shape as `get_automation_rule`'s `recent_run` entries) and `total` (count across the filtered set).

### `preview_automation_rule`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Dry run of the rule's product selector: how many products it would pick and actually queue on its next run, a sample, and whether the credit balance covers it. Writes nothing and spends no credits itself.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `rule_id` | integer | yes | Automation id (see `list_automation_rule`). |

**Returns:** `rule_id`, `rule_name`, `is_paused`, `next_run_at`, `candidate_count` (total matching products), `would_queue_next_run` (min of candidate_count and batch_size), `batch_size`, `credit_available`, `selector` (a fixed description: "Only products without a description, not generated in the last 30 days."), `sample_product` (up to 5 `{id, name}`).

### `update_automation_rule`

**Server:** merchant (`/mcp/client`)
**Costs credits:** no

Partial update of an automation — only the fields sent are changed. This is also the pause/resume switch via `is_paused`.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see `list_eshop`). |
| `rule_id` | integer | yes | Automation id (see `list_automation_rule`). |
| `is_paused` | boolean | no | true pauses the automation (stops running at night), false resumes it. |
| `name` | string | no | New name, max 120 chars. |
| `description` | string | no | New note, max 2000 chars. |
| `prompt_id` | integer | no | Different prompt profile (see `list_prompt`); validated the same way as on create. |
| `schedule_cron` | string | no | New cron expression in the eshop timezone, at most once per day. |
| `batch_size` | integer | no | New per-run product ceiling, min 1, max `config('automation.product.batch_size_max')` (default 500). |
| `category_id` | array of integer | no | REPLACES the category scope entirely. An empty array widens the automation to the whole catalogue. |
| `brand_id` | array of integer | no | REPLACES the brand scope entirely. An empty array widens the automation to the whole catalogue. |

**Returns:** `action: "updated"` plus the full rule detail (same shape as `create_automation_rule`).

**Careful:** `category_id` and `brand_id`, when sent, REPLACE the whole scope rather than adding to it — sending one category drops any others previously set. Setting `is_paused: false` also silently resets `consecutive_failure` to 0, so resuming a rule that had auto-paused after 3 failed nights clears its failure streak and gives it a fresh 3-strike count.
