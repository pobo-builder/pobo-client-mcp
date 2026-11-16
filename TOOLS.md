# Tool reference

This is the complete reference of every tool the two Pobo MCP servers expose. The
skills in `skills/` teach the workflows built on top of these tools; this file is
the lookup for what each tool takes and does.

Two servers:

- **merchant** — `https://api.pobo.space/mcp/client`
- **white label** — `https://api.pobo.space/mcp/whitelabel`

173 tools in total — 154 on the merchant server, 21 on the white label one, with
`list_eshop` and `list_design` served by both.

Every description and parameter below is what the server itself sends to your
agent in its tool list — when the two ever disagree, the server wins.

**Changed on 2026-11-16:** the reference now covers the tools added since September —
the full product and catalogue workflow (reading, creating, updating, uploading to the
eshop), content checks and fixes, translation checks, A/B tests, Pobo's agent, AI
visibility, plans and credits, and the tools for eshops on Pobo's own storefront
platform (orders, stock, prices, branches, add-ons). Several existing content tools
gained parameters too, most visibly `reason` on writes and batch forms of
`compose_content`.

## Rules that hold for every merchant tool

- **Pobo LLM must be enabled.** Every tool that takes `eshop_id` refuses an eshop
  where the merchant has not switched Pobo LLM on, and does nothing. `list_eshop`
  shows `pobo_llm_enabled` and the `pobo_llm_setting_url` where the merchant turns
  it on.
- **Write in the eshop's language.** Tools and hints are in English; the content you
  write goes in the eshop's `ai_lang` from `list_eshop`.
- **Calls per minute are limited by the eshop's plan.** Writes count towards the
  limit; most read-only tools do not (each entry says **Rate limit: counted / not
  counted**). Content writes are also capped per day by the number of entities
  changed. When a limit is hit the tool says so — wait, do not retry in a loop.
- **Paid tools quote first.** A tool that spends credits answers the first call with
  the price and does nothing. Repeat it with `cost_confirmed=true` only after the
  merchant agreed. `get_credit` shows the balance and the price list.
- **Destructive calls preview first.** Calls that delete content answer with a
  preview and a `confirm_token`; the second call sends the token with
  `confirmed=true`. If the content changed in between, the token no longer matches
  and the call is refused — preview again.
- **Plan gates.** AI generation needs the PROFI plan or higher; some tools need
  PREMIUM or SAAS and say so when refused. `list_tariff` lists what each plan
  includes.
- **Pobo storefront tools** (marked **Pobo storefront only**) work only on eshops on
  Pobo's own platform (`pobo`) and refuse any other eshop before they run.
- **Nothing reaches the eshop by itself.** Content written through these tools stays
  in Pobo until it is uploaded — by the merchant in the administration, or with
  `push_product`.

## Contents

- [Eshop, plans, credits and help](#eshop-plans-credits-and-help) — 7 tools
- [Styling](#styling) — 7 tools
- [Products, categories and the catalogue](#products-categories-and-the-catalogue) — 27 tools
- [Content — products, categories and blog articles](#content--products-categories-and-blog-articles) — 30 tools
- [Content quality — checks, fixes and translations](#content-quality--checks-fixes-and-translations) — 12 tools
- [Blog articles](#blog-articles) — 4 tools
- [Prompts, designs and generation](#prompts-designs-and-generation) — 13 tools
- [Diagnostics](#diagnostics) — 4 tools
- [Description automation](#description-automation) — 6 tools
- [Pobo agent](#pobo-agent) — 4 tools
- [A/B tests](#ab-tests) — 6 tools
- [AI visibility](#ai-visibility) — 1 tool
- [Pobo storefront — orders, stock, prices, branches and add-ons](#pobo-storefront--orders-stock-prices-branches-and-add-ons) — 33 tools
- [White label content](#white-label-content) — 19 tools

## Eshop, plans, credits and help

Start here. `list_eshop` gives the `eshop_id` every other tool takes and the `ai_lang` every text you write must be in. `get_help` answers how-to questions from the Pobo help centre, `list_tariff` and `get_credit` answer what the eshop may use and what it can afford, and the AI profile (brand, audience, tone, translation rules) is what every description is written against.

Tools: [`get_credit`](#get_credit), [`get_eshop_ai_profile`](#get_eshop_ai_profile), [`get_help`](#get_help), [`list_eshop`](#list_eshop), [`list_tariff`](#list_tariff), [`transfer_credit`](#transfer_credit), [`update_eshop_ai_profile`](#update_eshop_ai_profile)

### `get_credit`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How many credits do I have / where did my credits go?" — balance (remaining, total added, total spent), the price list of paid operations, transferable = what transfer_credit may move to another eshop of the merchant. period sums credits added and spent per category over the last day days (generate, translate, ai_image, prompt_test, translation_check, content_check, order = bought, bonus, grant, transfer = moved between the merchant's eshops, …); movement lists the newest rows with date, amount, category, note and the product, category, article or shared description they went on. Check it before offering a paid action so the merchant is not promised what the balance cannot cover.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `day` | integer | no | Period of `period` and `movement` in days, 1-365. Default 30. Default `30`. |
| `movement_limit` | integer | no | Newest movements to list, 0-100. Default 20. Default `20`. |

**Returns:** `credit` (`remaining`, `total_added`, `total_spent`) and `price` (the price list of paid operations).

### `get_eshop_ai_profile`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How should texts of this eshop sound?" — the AI profile: brand story, certifications, origin and quality of products, audience, their pains and what they value, frequent questions, shipping, returns and support, competitive advantage, tone, preferred and forbidden phrases, emoji, SEO keywords. Read it before writing any description and follow it. Blank fields are null and listed in empty_field: unknown, never fill them with a guess. The profile may be in another language — take facts and style, write in the target language. language says which language keys the content tools accept and which language AI writes in. translate_prompt = the merchant's rules for translations (null = none), set with update_eshop_ai_profile.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `get_help`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Where is … / how do I … in Pobo?" — Pobo's help articles, the ones the help drawer shows: where a feature is, how it works, what a setting means. Without slug it lists every article (title, slug); pick the ones that fit by meaning and read each with slug. Answer from the text; never invent features the articles do not describe.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `slug` | string | no | The article to read; omit to list them all. |
| `lang` | string | no | Language of the articles; cs when omitted. One of: `cs`, `sk`, `en`. |

### `list_eshop`

**Server:** merchant (`/mcp/client`) and white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The merchant's active eshops (id, url, platform, ai_lang, pobo_llm_enabled, pobo_llm_setting_url); id is eshop_id for every other tool. Write every text you put on an eshop in its ai_lang — tools and hints are in English, content is not. Every tool taking eshop_id refuses an eshop with pobo_llm_enabled=false: send the merchant to pobo_llm_setting_url first.

No parameters.

### `list_tariff`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What does a plan cost / what does it include?" — the Pobo plans of the eshop's platform as the plan page shows them: monthly and yearly price without VAT, free credits, limits of edited products, categories, articles and AI visibility queries, features (each plan has everything of the one before), the plan the eshop pays for and the one it may use now (SaaS during the 7-day trial). Quote only what it returns.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `transfer_credit`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** see the description  
**Rate limit:** counted

"Move credits to my other eshop" — transfers credits between the merchant's own eshops. Only purchased credits (install bonus and gifts stay) — check transferable in get_credit first. The source eshop needs PROFI or higher (the trial counts). Cannot be undone; a transfer back is a new transfer. Confirm the amount and target eshop with the merchant before calling.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Source eshop id (see list_eshop) — the credits leave this eshop. |
| `target_eshop_id` | integer | yes | Target eshop id (see list_eshop) — must be another eshop of the same merchant. |
| `count` | integer | yes | Number of credits, at most `transferable` from get_credit. |
| `note` | string | no | Optional note shown with the transfer. |

### `update_eshop_ai_profile`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Remember how my eshop sounds / save my translation rules" — changes fields of the AI profile, the brand, audience, services and style every description is written against (by you and by the Pobo generator), and translate_prompt, the rules every translation follows from then on (terminology, address, what stays untranslated; existing translations are not redone). Only the fields sent change; an empty string clears one. Show the merchant what you are about to save and get their agreement — never fill it with guesses; a good source is the eshop's own pages (fetch_eshop_page: about us, shipping, returns), say where each value came from. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `company_brand_story` | string | no | Story of the brand: who they are, since when, why. Max 5000 characters. |
| `company_certification` | string | no | Certifications and awards. Max 500 characters. |
| `company_country_origin` | string | no | Where the company is from and where it produces. Max 255 characters. |
| `product_origin` | string | no | Where the products come from (own production, import, local farmers). Max 500 characters. |
| `product_quality` | string | no | Quality and materials of the products. Max 5000 characters. |
| `product_unique` | string | no | What is unique about the products. Max 5000 characters. |
| `customer_target_audience` | string | no | Who the customers are. Max 5000 characters. |
| `customer_pain` | string | no | Problems of the customers the products solve. Max 5000 characters. |
| `customer_value` | string | no | What customers appreciate about the eshop. Max 5000 characters. |
| `customer_faq` | string | no | Questions customers ask most often. Max 5000 characters. |
| `service_shipping` | string | no | Shipping: speed, price, free shipping threshold, pickup. Max 500 characters. |
| `service_return_policy` | string | no | Returns and warranty. Max 500 characters. |
| `service_support` | string | no | Customer support: channels and hours. Max 500 characters. |
| `competitive_advantage` | string | no | Why buy here and not at a competitor. Max 5000 characters. |
| `style_communication_tone` | string | no | Tone of the texts, including formal or informal address. Max 255 characters. |
| `style_preferred_phrase` | string | no | Words and phrases the texts should use. Max 5000 characters. |
| `style_forbidden_phrase` | string | no | Words, phrases and topics the texts must avoid. Max 5000 characters. |
| `seo_keyword` | string | no | SEO keywords of the eshop. Max 5000 characters. |
| `style_use_emoji` | boolean | no | Whether the texts may use emoji. |
| `translate_prompt` | string | no | Instruction for translating the eshop content: terminology, address, what stays untranslated. Max 2000 characters. |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |

## Styling

AI styling of Pobo widgets: read the eshop's design, push SCSS that overrides the `--pobo-widget-*` / `--pobo-global-*` variables, and look at the result in a real browser. Each eshop has at most one AI style asset — `push_asset_css` replaces it.

Tools: [`delete_asset`](#delete_asset), [`fetch_eshop_page`](#fetch_eshop_page), [`get_content_html`](#get_content_html), [`get_theming_contract`](#get_theming_contract), [`list_asset`](#list_asset), [`push_asset_css`](#push_asset_css), [`screenshot_eshop_page`](#screenshot_eshop_page)

### `delete_asset`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

Deletes the AI style asset of an eshop (the CSS pushed with push_asset_css) and regenerates its CDN bundle — "put the original look back". Only AI-managed assets.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `asset_id` | integer | yes | Asset id (see list_asset or the push_asset_css response). |

**Returns:** `action: "deleted"`, `id` (the deleted asset id).

**Careful:** destructive and irreversible — only AI style assets (pushed with `push_asset_css`) can be deleted here — assets uploaded in the admin or by the CLI are out of scope. Deletion triggers a live CDN regeneration.

### `fetch_eshop_page`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** counted

"What does my eshop look like / what does this page say?" — downloads the live HTML of a page of the merchant's eshop (homepage, a product page, about us) to read its design tokens (colors, fonts, radii, buttons) or its texts (shipping, returns, about us for the AI profile).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `path` | string | no | Page path starting with "/", e.g. "/" for the homepage or a product detail path. Default `"/"`. |

**Returns:** the raw page HTML as text, truncated with a comment marker past 500,000 bytes.

**Careful:** not an open proxy — only the eshop's own host (and its www/non-www twin) may be reached; redirects are followed up to 3 hops with the host re-validated on every hop, and a redirect leaving the eshop domain aborts the fetch with an error.

### `get_content_html`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The rendered HTML of the Pobo widget content of one page (product, category or blog), clean, without shop chrome, plus the widget classes it uses. Address it with the data-pobo-content and data-pobo-page-id attributes from fetch_eshop_page.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `content` | string | yes | Content type — the data-pobo-content attribute value. One of: `product`, `category`, `blog`. |
| `page_id` | integer | yes | Pobo page id — the data-pobo-page-id attribute value. |
| `lang` | string | no | Language mutation of the content. Default "default". One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. Default `"default"`. |

**Returns:** `html` (truncated past 200,000 bytes with a comment marker), `widget_class` (distinct `widget-*` CSS classes found in the HTML), `lang`, `truncated` (boolean).

### `get_theming_contract`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How can the widgets be styled?" — the theming contract: native .widget-* BEM blocks and --pobo-widget-* / --pobo-global-* CSS variables with defaults. Variables live on :root — write overrides on :root in the SCSS for push_asset_css. The storefront assistant widget has its own --pobo-assistant-* variables (assistant_variable), allowed on any selector.

No parameters.

### `list_asset`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The AI-managed style assets of an eshop (what push_asset_css deployed). Assets from the admin or the CLI are not listed.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

**Returns:** `asset` — array of `id`, `name`, `type`, `size` (bytes), `updated_at` (ISO 8601).

**Careful:** only shows AI style assets pushed with `push_asset_css` — assets from the admin UI or the CLI are not visible here.

### `push_asset_css`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Make the descriptions match my eshop's look" — creates or updates the AI style asset of an eshop from SCSS: compiled, validated and deployed to the eshop CDN bundle. One AI asset per eshop — a push replaces the previous one. Override only --pobo-widget-* / --pobo-global-* variables and native .widget-* blocks (get_theming_contract); no @import; url() only to the Pobo or platform CDN over https, a relative path or a data:image URI. Then screenshot_eshop_page to see it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `name` | string | yes | Human-readable asset name shown in the merchant admin, e.g. "AI design unification". Max length 255. |
| `scss` | string | yes | SCSS source overriding --pobo-widget-*/--pobo-global-* variables on :root and styling native .widget-* blocks. Max 256 KB. No @import. url(...) only as https on an allowlisted CDN host (Pobo CDN, the platform asset CDN), a relative path, or a data:image URI. |

**Returns:** `action` (`created` or `updated`), `id`, `name`, `size` (bytes), `cdn_url`.

**Careful:** each eshop has at most one AI style asset — pushing always replaces the previous content (upsert), it does not create a second asset. Compilation errors and forbidden CSS constructs (`@import`, `expression()`, `javascript:`, disallowed `url(...)` origins) are rejected with an error before anything is written.

### `screenshot_eshop_page`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** counted

"Show me how it looks" — a screenshot of a page of the merchant's eshop rendered by a real browser; after push_asset_css to SEE the styling and iterate. selector "#pobo-all-content" captures just the Pobo widget area.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `path` | string | no | Page path starting with "/", e.g. "/" or a product detail path. Default `"/"`. |
| `viewport` | string | no | Viewport to render in. Default "desktop" (1520×1080); "mobile" is 475×812 rendered at 2× scale (950×1624 image, retina-sharp). One of: `desktop`, `mobile`. Default `"desktop"`. |
| `selector` | string | no | Optional CSS selector — capture just that element instead of the viewport. Use "#pobo-all-content" for the Pobo widget area. |

**Returns:** the screenshot as an inline image, plus — best-effort — a public CDN URL to the same image as a text block (upload failure never fails the tool; the inline image is still returned).

**Careful:** screenshots larger than 3,000,000 bytes are rejected with an error suggesting the `selector` argument to narrow the capture; the target host is always the eshop's own stored URL, only a path is accepted as input.

## Products, categories and the catalogue

Finding products (`find_product` for a client's list of EANs, codes, URLs or names, `list_product` for the admin grid filters, `search_product` for full text), reading one in full (`get_product`), labelling, switching draft ⇄ ready, and uploading Pobo content to the eshop (`push_product` → `get_push_status`).

On eshops whose catalogue lives in Pobo (platforms `pobo` and `b2b`) the agent can also maintain the catalogue itself: `create_product`, `update_product`, `create_category`, `update_category` and, on `pobo`, `update_product_pricing`. Every catalogue write takes a `reason` and is recorded field by field (before → after) in the overview of agent changes (`list_agent_change`).

`start_import` / `get_import_status` keep Pobo's copy of a Shoptet, Upgates, Shopify, WooCommerce or PrestaShop catalogue fresh; they never upload anything.

Tools: [`assign_product_label`](#assign_product_label), [`assign_product_type`](#assign_product_type), [`assign_share_product`](#assign_share_product), [`create_category`](#create_category), [`create_label`](#create_label), [`create_product`](#create_product), [`enhance_product_image`](#enhance_product_image), [`find_product`](#find_product), [`get_import_status`](#get_import_status), [`get_product`](#get_product), [`get_product_analytics`](#get_product_analytics), [`get_push_status`](#get_push_status), [`list_category`](#list_category), [`list_label`](#list_label), [`list_product`](#list_product), [`list_product_type`](#list_product_type), [`list_share`](#list_share), [`push_product`](#push_product), [`research_product`](#research_product), [`save_product_type`](#save_product_type), [`search_product`](#search_product), [`set_product_status`](#set_product_status), [`start_import`](#start_import), [`sync_product_image`](#sync_product_image), [`update_category`](#update_category), [`update_product`](#update_product), [`update_product_pricing`](#update_product_pricing)

### `assign_product_label`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Tag these products" — attaches a label to a batch of products, or detaches it (action=detach). Attaching an attached label does nothing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `label_id` | integer | yes | Label id (see list_label / create_label). |
| `product_id` | array of integer | yes | Product ids to label (see find_product). Min items 1. Max items 500. |
| `action` | string | no | attach adds the label, detach removes it. One of: `attach`, `detach`. Default `"attach"`. |

**Returns:** `action`, `label_id`, `label_name`, `product_count` (number of products processed).

**Careful:** strict ownership — the label and every product must belong to the eshop, and a single foreign or unknown product id fails the whole call (no silent partial writes); the error lists the offending ids. Idempotent — attaching an already-attached label, or detaching an unattached one, is a no-op.

### `assign_product_type`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Files products under one of your product types (list_product_type). A product has one type; assigning moves it from its previous type. get_product then reports it, and the description should follow its structure. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `product_type_id` | integer | yes | The type (see list_product_type). |
| `product_id` | array of integer | yes | Pobo product ids, 50 at most. |

### `assign_share_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Add / remove this shared block on these products" — attaches a shared description (list_share) to a batch of products, or detaches it (action=remove). Only the link changes; the shared description and the rest of each product stay. To swap one for another, remove the old and add the new. The eshop shows it after push_product of those products.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `share_id` | integer | yes | Shared description id (see list_share). |
| `product_id` | array of integer | yes | Product ids (see find_product, list_product). Min items 1. Max items 500. |
| `action` | string | no | add attaches the shared description, remove detaches it. One of: `add`, `remove`. Default `"add"`. |

### `create_category`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Add / move categories" on an eshop with its own catalogue (pobo or b2b), up to 50 of each per call. A new category gets its parent as parent_id (list_category) or as path — names from the root down; existing levels (same name, ignoring accents and case) are reused, missing ones created; a category with the same name under the same parent is reported as existing with its id. move: [{category_id, parent_id (null = top level), position}], a key left out keeps its value; a move under its own descendant is refused. Renaming is update_content_meta. Every change is audited with your reason (required).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | yes | Why — kept in the audit. |
| `category` | array of object | no | Max items 50. |
| `category[].name` | string | yes | Max length 250. |
| `category[].parent_id` | integer | no |  |
| `category[].path` | array of string | no | parent names from the root down |
| `category[].is_visible` | boolean | no |  |
| `category[].position` | integer | no |  |
| `move` | array of object | no | Max items 50. |
| `move[].category_id` | integer | yes |  |
| `move[].parent_id` | integer | no | null = top level |
| `move[].position` | integer | no |  |

### `create_label`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Create a label" to group products in the admin grid — or returns the existing label of that name.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `name` | string | yes | Label name shown in the merchant admin, e.g. "Copy update 07/2026". Min length 2. Max length 255. |
| `color` | string | no | Hex color of the label, e.g. "#3B82F6". Default `"#3B82F6"`. Pattern `^#[0-9A-Fa-f]{6}$`. |
| `description` | string | no | Optional description of the label. Max length 1000. |

**Returns:** `action` (`created` or `existing`), `id`, `name`, `color`.

**Careful:** idempotent by name within the eshop — calling it again with the same name returns the existing label instead of creating a duplicate.

### `create_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Add these products to my catalogue" — creates up to 50 products by hand on an eshop with its own catalogue (pobo or b2b). Each product: name, url (required on b2b, optional on pobo), remote_id, is_visible, short_description, old_description (HTML, sanitized), image (https URLs), parameter (label/value), category_id (from list_category), brand (name, created when missing), variant (code + optional EAN). On pobo a variant may carry price, sale price and stock (default price list and warehouse; list_price_list says whether prices include VAT); a product with prices needs vat_rate. Items are validated first, invalid ones reported and skipped. Give a reason — it is kept in the audit. On Shoptet and Upgates with the SaaS plan the product is created HIDDEN on the platform (up to 20 per call on Shoptet, 50 on Upgates) with price, stock and existing categories; the price there is the gross price including VAT in the eshop currency (vat_rate is not taken), each product needs a variant — exactly one on Shoptet — and Shoptet also a category; url, remote_id (Shoptet) and sale prices are not taken, the merchant publishes the product on the platform. Not on other imported platforms.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why these products are created — shown to the merchant in the change overview. |
| `product` | array of object | yes | Max items 50. |
| `product[].name` | string | yes | Max length 250. |
| `product[].url` | string | no | https address of the product page; required on b2b, optional on pobo |
| `product[].remote_id` | string | no | your own product id; generated when left out |
| `product[].is_visible` | boolean | no |  |
| `product[].short_description` | string | no | HTML |
| `product[].old_description` | string | no | HTML — the original description of the shop |
| `product[].image` | array of string | no |  |
| `product[].parameter` | array of object | no |  |
| `product[].parameter[].label` | string | yes |  |
| `product[].parameter[].value` | string | yes |  |
| `product[].category_id` | array of integer | no |  |
| `product[].brand` | string | no | brand name; created when the eshop has no brand of that name; pobo and b2b only Max length 250. |
| `product[].vat_rate` | number | no | VAT rate in percent, e.g. 21 — required when a variant has a price on pobo; on Shoptet and Upgates not taken (the eshop default rate applies) |
| `product[].variant` | array of object | no |  |
| `product[].variant[].code` | string | yes |  |
| `product[].variant[].ean` | string | no | 8–14 digits |
| `product[].variant[].price` | number | no | pobo: price in the default price list, with or without VAT as that list says (see list_price_list); Shoptet and Upgates: gross price including VAT in the eshop currency |
| `product[].variant[].action_price` | number | no | sale price, lower than price |
| `product[].variant[].action_from` | string | no | sale start, ISO date; empty = now |
| `product[].variant[].action_to` | string | no | sale end, ISO date; empty = open-ended |
| `product[].variant[].quantity` | number | no | stock in the default warehouse; pobo, Shoptet, Upgates |
| `product[].variant[].allow_backorder` | boolean | no | sell without stock; pobo, Shoptet, Upgates |
| `product[].variant[].weight` | number | no | kg of one piece, for the parcel; pobo only |

### `enhance_product_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Make a nicer photo of this product" — from one of the product's own photos makes a better one and adds it to its gallery in Pobo (on pobo the storefront gallery; elsewhere Pobo descriptions can use it — the Shoptet or Upgates gallery is not changed). Up to 10 photos per call. mode retouch = the same product on a clean white studio background; lifestyle = the product in the setting described in scene. image_url picks the source (an address from get_product), otherwise the first shop photo. replace=true (pobo and b2b only) puts the new photo in place of the source. PAID, two steps: the first call only quotes; repeat it with cost_confirmed=true. Made in the background within about a minute, then get_product lists them; only stored photos are billed. An image model redraws the picture — look at it before using it first or in a description.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `image` | array of object | yes | Max items 10. |
| `image[].product_id` | integer | yes |  |
| `image[].mode` | string | yes | One of: `retouch`, `lifestyle`. |
| `image[].scene` | string | no | lifestyle only: where the product is, in a sentence Max length 500. |
| `image[].image_url` | string | no | source photo, one of the addresses get_product lists |
| `image[].replace` | boolean | no | pobo and b2b only: the new photo replaces the source photo in the gallery |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost. |

### `find_product`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Find these products" for a pasted list of EANs, product codes, URLs or names — resolves them to Pobo products in one call, each matched, ambiguous (with candidates) or not_found. For products of a kind (brand, property) use search_product.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `identifier` | array of string | yes | Identifiers to resolve — EANs, product codes, product URLs or product names, freely mixed. Min items 1. Max items 100. Each item max length 2048. |
| `type` | string | no | Force one identifier type for the whole batch. Default "auto" detects per identifier: http(s)/www prefix = url, 8 or 13 digits = ean, otherwise exact code match with name search fallback. One of: `auto`, `ean`, `code`, `url`, `name`. Default `"auto"`. |

**Returns:** `result` — one entry per identifier, each with `identifier`, `type`, and `status` of `matched` (with `product`: id/name/code/url), `ambiguous` (with up to 5 `candidate` entries), or `not_found`.

### `get_import_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Pobo does not show my new or changed products" — when Pobo last downloaded products, categories and articles from the eshop platform and how the newest downloads went: status (waiting, running, complete, failed), what they covered (all, product, category, blog), manual or background (webhooks, daily refresh), progress and times; last_complete_at = last finished download. Changes appear in Pobo only after a completed download — a failed one changed nothing. start_import downloads again. Free, changes nothing. The other direction (Pobo content not on the eshop) is get_export_status.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `get_product`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Tell me about this product" — everything the eshop knows: name, short description, SEO fields, URL, the platform description today (plain text), code, EAN, brand, categories, parameters, variants with price and stock, the photo gallery (image_count 0 usually means never downloaded — sync_product_image) and its product type. Read it before writing a description: these are the facts to write from; add no product fact that is not here or in get_product_research. Lists are capped and report their full count. lang picks the language version, lang_used says which one came back. product_id reads one, product_id_list up to 20 (unknown ids in not_found).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_id` | integer | no | One Pobo product id (see find_product / list_product). Give this or product_id_list. |
| `product_id_list` | array of integer | no | Several Pobo product ids, 20 at most. |
| `lang` | string | no | Language version of the texts; omit for the default (see lang_key in get_eshop_ai_profile). One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

### `get_product_analytics`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** counted

"How does this product perform? / Did the new description help?" — page views, add-to-cart and orders for the period, before/after around the first Pobo description, engagement (scroll depth, reading time, top FAQ questions), the content heatmap (which sections visitors see, how long, and the share that adds to cart after) and content_change: when the description went live and what changed it (generation, editor, agent, translation, A/B test winner), is_not_live = visitors kept getting the previous version. IMPORTANT: cart rate and before/after are CORRELATIONS, not causality (engaged visitors scroll further and buy more; seasons and campaigns move numbers); orders count only when the customer viewed the product page first (Shoptet only); null = not enough data, never zero; content_change carries no effect per change on purpose — do not attribute traffic moves to it, propose an A/B test to measure one.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_id` | integer | yes | Product id (see find_product). |
| `period` | string | no | Analysed period ending today. Default 90d. One of: `30d`, `60d`, `90d`. |

**Returns:** `product_id`, `period`, `totals` (page_views, unique_visitors, add_to_cart_count, order_count summed over the period), `pobo_content`, `before_after_summary`, `engagement`, `section_map`, `meta`. The raw daily series is not returned, only period totals.

**Careful:** `cart_rate_percent` and the before/after change are correlations, not causality; order counts cover only orders where the customer viewed the product page first, and only on Shoptet; `null` values mean not enough data yet, not zero performance.

### `get_push_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Did the upload to the eshop go through?" — a push started by push_product: status pending, running, complete or failed, counters, and per product ok, failed (with the platform message), skipped (no_content or not_approved — nothing broke) or not_found. An ok product lists in untranslated_lang the languages whose description stayed out because it is not translated — the eshop keeps its previous text there; translate them and push again. A token older than 30 minutes, or of another eshop, comes back as not_found.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `token` | string | yes | Token returned by push_product. |

### `list_category`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

Browse categories the way the admin category grid does: filter (all, without_description, edited, favourite, waiting_for_approval = drafts, concept = saved but not published, recently_edited), full-text, label (or without_label), with or without products, and the translation check (translation = ok / attention); each carries its translation state per language. Also turns category names into ids for list_product's category_id.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `filter` | string | no | Grid filter: without_description = no Pobo content yet, edited = has Pobo content, favourite, waiting_for_approval = status draft, concept = an edit saved but not published yet (status concept); recently_edited orders the full list. One of: `all`, `without_description`, `edited`, `favourite`, `waiting_for_approval`, `concept`, `recently_edited`. Default `"all"`. |
| `query` | string | no | Full-text search in category name. Max length 255. |
| `label_id` | array of integer | no | Filter by label ids (see list_label). |
| `without_label` | boolean | no | Only categories with no label. Wins over label_id, like in the admin grid. Default `false`. |
| `product` | string | no | with = categories that hold an active product, without = empty categories. One of: `all`, `with`, `without`. Default `"all"`. |
| `translation` | string | no | Translation check filter, as in the admin grid: ok = translated, nothing missing, no open finding of the AI translation check; attention = anything else (not translated, partly, out of date, or an open finding). Read the findings of one item with get_translation_finding. One of: `ok`, `attention`. |
| `translation_lang` | string | no | Language of the translation filter (a translation language of the eshop, see list_eshop), or "any". With "any", ok needs every language in order and attention any one of them. Default `"any"`. |
| `limit` | integer | no | Categories per page. Default `50`. Min 1. Max 100. |
| `page` | integer | no | 1-based page number. Default `1`. Min 1. |

**Returns:** `category` — array of `id`, `name`, `url`, `status`, `is_visible`, `is_favourite`, `has_content`, `product_count`; plus `total`, `page`, `limit`.

### `list_label`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The labels of an eshop (id, name, color, products using it); labels group products, categories and articles in the admin grids.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

**Returns:** `label` — array of `id`, `name`, `color`, `description`, `is_active`, `product_count` (usage count among products).

### `list_product`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

Browse products the way the admin product grid does: filter (all, without_description, edited, favourite, waiting_for_approval = drafts, recently_edited, most_visited, most_added_to_cart), full-text over names, label (or without_label), category, brand, visibility, content (without / thin / done), shared description, AI visibility, findings of a content check and the translation check (translation = ok / attention); each product carries its translation state per language. Not for questions about properties (search_product) or traffic (get_optimize_candidate) — never page through it to count or to find what to improve.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `filter` | string | no | Grid filter: without_description = no Pobo content yet, edited = has Pobo content, favourite, waiting_for_approval = status draft, concept = an edit the merchant saved but has not published yet (status concept); recently_edited / most_visited / most_added_to_cart order the full list. One of: `all`, `without_description`, `edited`, `favourite`, `waiting_for_approval`, `concept`, `recently_edited`, `most_visited`, `most_added_to_cart`. Default `"all"`. |
| `query` | string | no | Full-text search in product name, code and EAN. Max length 255. |
| `label_id` | array of integer | no | Filter by label ids (see list_label). |
| `without_label` | boolean | no | Only products with no label. Wins over label_id, like in the admin grid. Default `false`. |
| `category_id` | array of integer | no | Filter by category ids. |
| `brand_id` | array of integer | no | Filter by brand ids. |
| `is_visible` | string | no | Filter by product visibility on the eshop. One of: `all`, `visible`, `hidden`. Default `"all"`. |
| `content` | array of string | no | Own widgets of the description, as the admin "Content" filter: without = none, thin = 1–2 widgets, done = more. Several combine with OR. One of: `without`, `thin`, `done`. |
| `share` | string | no | with = shows a shared description (see list_share), without = none. One of: `with`, `without`. |
| `ai_visibility` | string | no | Result of the latest finished AI visibility run: mentioned, missed, error (the answer failed), unmeasured (no answered result). One of: `mentioned`, `missed`, `error`, `unmeasured`. |
| `content_check_id` | integer | no | Only products with a finding of this bulk content check (check_id of start_content_bulk_check / get_content_bulk_check). |
| `content_check_severity` | string | no | With content_check_id: only findings of this severity. One of: `high`, `medium`, `low`. |
| `translation` | string | no | Translation check filter, as in the admin grid: ok = translated, nothing missing, no open finding of the AI translation check; attention = anything else (not translated, partly, out of date, or an open finding). Read the findings of one item with get_translation_finding. One of: `ok`, `attention`. |
| `translation_lang` | string | no | Language of the translation filter (a translation language of the eshop, see list_eshop), or "any". With "any", ok needs every language in order and attention any one of them. Default `"any"`. |
| `limit` | integer | no | Products per page. Default `50`. Min 1. Max 100. |
| `page` | integer | no | 1-based page number. Default `1`. Min 1. |

**Returns:** `product` — array of `id`, `name`, `code`, `url`, `status`, `is_visible`, `is_favourite`, `has_content`; plus `total`, `page`, `limit`.

### `list_product_type`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The product types derived for this eshop earlier: how to recognise each, product count, and the description structure every product of the type follows (widget_id + purpose of each block, in order) with the reason. Read it before writing — a product of a known type gets that structure, so descriptions of one type stay consistent. next_structure_change_at says until when the structure is locked. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `list_share`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which shared descriptions do I have / where is this one used?" — shared descriptions: a block of widgets written once on the Shared descriptions page and shown in every product it is attached to. id, name, product_count, widget_count; with share_id that one with the ids of all its products. The block itself is edited by the merchant in Pobo; attach or detach with assign_share_product.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `query` | string | no | A fragment of the name. |
| `share_id` | integer | no | One shared description, returned with its product ids. |

### `push_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Upload it to the eshop" — sends the Pobo content of up to 50 products to the platform so customers see it; content written with the content tools stays in Pobo until then. Runs in the background: returns a token for get_push_status. It overwrites what the eshop shows today for those products. Skipped, not failed: no_content (no Pobo content), not_approved (waiting for approval, set_product_status). Run check_content on every product first: push repeats its rules but not the fact and spelling review — a product that failed, was never checked, or changed since (not_reviewed) comes back in refused with its issues. The fact review reads the platform data, the AI profile and the research. acknowledge_fact=true with a reason lets fact issues pass (they come back in acknowledged) — only when the merchant confirmed those claims are true, never on your own judgement; spelling, filler, the rules and a missing review still refuse.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `product_id` | array of integer | yes | Pobo product ids to push, 50 at most. |
| `acknowledge_fact` | boolean | no | The merchant confirmed the claims check_content reported as fact are true: send despite them. Requires reason. |

### `research_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — 1 credit per provider that found something; the same question within a day is free  
**Rate limit:** counted

"Find facts about this product on the web" — OpenAI web search by EAN and Perplexity by name: facts (label and value), a short summary, how it is used and the source URLs. Call it when get_product has no description and few parameters — without facts a description is only filler. Up to a minute. 1 credit per provider that found something (free inside a Pobo agent task); the same question within a day comes from the cache for free. Writes nothing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_id` | integer | yes | Pobo product id (see find_product / list_product). |

### `save_product_type`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Creates a product type (omit product_type_id) or changes one. A type comes from the products themselves — "rubber car mats", "kitchen knives" —, not from an eshop category. structure = the ordered blocks every description of the type follows: widget_id from get_content_widget_catalog plus the purpose of the block for this type. reason is required with a structure, one sentence the merchant reads. A structure may change once in 30 days; the first one is free. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_type_id` | integer | no | The type to change; omit to create one. |
| `name` | string | no | Short name of the type, unique on the eshop. Max 100 characters. Required when creating. |
| `description` | string | no | How to recognise a product of this type, and what a buyer of it needs to know. Max 2000 characters. Required when creating. |
| `structure` | array of object | no | Ordered blocks of the description, 20 at most. |
| `structure[].widget_id` | integer | yes |  |
| `structure[].purpose` | string | yes |  |
| `reason` | string | no | Why this structure. Required with structure. Max 1000 characters. |

### `search_product`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How many / which products are vegan, in capsules, size 42, for sensitive skin, of brand X?" — finds products by brand, parameters and what a shopper would type, with the exact number of matches. Use it, not list_product, for every question about what products are like: properties live in parameters, list_product searches names only. Two steps: call with query — the answer carries parameter_facet (parameter names and values of the products found) — then again WITHOUT query, only with the matching parameter (and brand) filter. total_kind: "filtered" = the exact number of products of that kind, the only number to answer "how many" with; "related" = everything related to the free text, never a count. Brand matches exactly by name, ignoring case and accents; brand_suggestion lists similar brands when nothing matches, related_brand lists sub-brands with counts — mention them. Without free text products come ordered by add-to-cart. Needs the eshop's search index; without it the call says so — use list_product then.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `brand` | string | no | Brand name, e.g. "Aqualung". Give this and/or query. Max length 100. |
| `query` | string | no | What to look for, e.g. "neoprenový oblek". Give this and/or brand. Max length 200. |
| `category_id` | array of integer | no | Only products in these categories (ids from list_category). |
| `parameter` | array of string | no | Parameter filters as "Name: value", e.g. ["Forma: kapsle", "Vlastnosti: vegan"] — all must hold (at most 5). The name is matched exactly (ignoring case and accents), the value as words. When nothing matches, parameter_suggestion lists the parameter names and common values of the products that match the rest. Max items 5. |
| `in_stock` | boolean | no | true = only products in stock, false = only out of stock. |
| `limit` | integer | no | How many products to return, 50 at most (default 20). total_count is exact regardless. |

### `set_product_status`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Approve these products" / "put them back to drafts" — switches a batch between ready (approved for export) and draft (waiting for approval). Only these two statuses, and only products currently ready or draft. Setting the current status again does nothing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `product_id` | array of integer | yes | Product ids to update (see find_product). Min items 1. Max items 500. |
| `status` | string | yes | Target status: ready = approved for export, draft = waiting for approval. One of: `ready`, `draft`. |

**Returns:** `status` (the status applied), `product_count` (number updated).

**Careful:** restricted in both directions — only `ready`/`draft` can be set, and only products currently in `ready` or `draft` can be switched; products in other pipeline states (`review`, `generate`) are refused. A single foreign, unknown, or non-switchable product id fails the whole call with no partial writes; the error lists the offending ids. Idempotent — setting the current status again is a no-op.

### `start_import`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Download my products from the eshop now" — the "download from the eshop" button: reads the platform and refreshes Pobo's copy of products, categories and articles; never uploads anything. Use it when products added or changed on the platform are missing in Pobo. A download already waiting or running is returned instead of a new one. Minutes for a large catalogue — follow it with get_import_status. Free. Not for platforms pobo and b2b, whose catalogue lives in Pobo.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `sync_product_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"The product has no photos in Pobo" — downloads the photo gallery of up to 20 products from the platform into Pobo so get_product lists them and picture widgets can use them; call it when get_product reports image_count 0. Reads the eshop, writes nothing to it. Each product returns image_count after the sync, or why it failed.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `product_id` | array of integer | yes | Pobo product ids, 20 at most. |

### `update_category`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Hide / show a category" or make it reservation-only on an eshop with its own catalogue (pobo or b2b), up to 50 per call: is_visible, and on pobo is_reservation_only — goods of the category and all its subcategories are only reserved for a visit at a branch, the cart refuses them. Only what is sent changes. Name, heading and SEO are update_content_meta, the place in the tree create_category (move), pictures set_storefront_image. Each comes back updated or failed with its current is_reservation_only and reservation_only_from when inherited. Every change is audited before and after with your reason (required).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | yes | Why — kept in the audit. |
| `category` | array of object | yes | Max items 50. |
| `category[].category_id` | integer | yes | see list_category |
| `category[].is_visible` | boolean | no |  |
| `category[].is_reservation_only` | boolean | no | pobo only |

### `update_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Hide / show / move / change these products" on an eshop with its own catalogue (pobo or b2b), up to 50 per call: is_visible, category_add / category_remove (from list_category), image (https addresses — replaces the list, photos already downloaded stay), parameter (label/value — replaces the ones added by hand), remote_id, brand (name, created when missing; empty string removes it), variant_add (code + EAN; on pobo also price, sale price and stock). Name and SEO fields are update_content_meta, prices of existing variants update_product_pricing. Only what is sent changes. Every change is audited before and after with your reason (required); each product updated or failed.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | yes | Why — kept in the audit. |
| `product` | array of object | yes | Max items 50. |
| `product[].product_id` | integer | yes |  |
| `product[].is_visible` | boolean | no |  |
| `product[].category_add` | array of integer | no |  |
| `product[].category_remove` | array of integer | no |  |
| `product[].image` | array of string | no | https addresses; replaces the list |
| `product[].parameter` | array of object | no | replaces the parameters added by hand |
| `product[].parameter[].label` | string | yes |  |
| `product[].parameter[].value` | string | yes |  |
| `product[].remote_id` | string | no |  |
| `product[].brand` | string | no | brand name, created when missing; empty string removes the brand Max length 250. |
| `product[].variant_add` | array of object | no |  |
| `product[].variant_add[].code` | string | yes |  |
| `product[].variant_add[].ean` | string | no | 8–14 digits |
| `product[].variant_add[].price` | number | no | pobo only — default price list |
| `product[].variant_add[].action_price` | number | no | pobo only |
| `product[].variant_add[].action_from` | string | no |  |
| `product[].variant_add[].action_to` | string | no | a date alone is the end of that day |
| `product[].variant_add[].quantity` | number | no | pobo only — default warehouse |
| `product[].variant_add[].allow_backorder` | boolean | no |  |

### `update_product_pricing`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Change prices / stock" on a pobo eshop — price, sale price, sale dates, stock and backorders of existing variants addressed by variant code (up to 100 per call), and the VAT rate of their products. Only what is sent changes; "action_price": null ends a sale. The default price list unless price_list_code names another (list_price_list: VAT included or not). A date-only action_to means the end of that day. Every change is audited before and after with your reason (required). Each item updated or failed; one failure does not stop the others.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | yes | Why — shown to the merchant in the change overview and kept in the audit. |
| `variant` | array of object | no | Max items 100. |
| `variant[].variant_code` | string | yes |  |
| `variant[].price_list_code` | string | no | default price list when left out |
| `variant[].price` | number | no |  |
| `variant[].action_price` | number | no | sale price, lower than the price; null ends the sale |
| `variant[].action_from` | string | no | ISO date |
| `variant[].action_to` | string | no | ISO date; a date alone is the end of that day |
| `variant[].quantity` | number | no | stock in the default warehouse |
| `variant[].allow_backorder` | boolean | no |  |
| `variant[].weight` | number | no | kg of one piece, for the parcel; null clears it |
| `product` | array of object | no | Max items 100. |
| `product[].variant_code` | string | yes | any variant of the product |
| `product[].vat_rate` | number | yes |  |

## Content — products, categories and blog articles

One set of tools writes the Pobo description of a **product, a category or a blog
article** — every one of them takes `entity_type` (`product` | `category` | `blog`) and
`entity_id`, the Pobo id from `list_product` / `list_category` / `list_blog`.

They replaced the eleven `*_blog_*` widget tools on 2026-09-15. If an older integration
still calls `add_blog_widget`, `get_blog_content`, `set_blog_widget_image`, `review_blog`,
`update_blog_title` or their siblings, it will get "Tool not found" — the replacement is
the same operation with `entity_type: "blog"`.

Three things hold for the whole set:

- **You write the texts, and that is free.** Nothing here calls a model. The only paid
  thing is `image_source="ai"`, which quotes its price first and writes nothing until you
  repeat the call with `cost_confirmed=true`.
- **Every write is versioned.** `get_content_history` lists the versions of one entity,
  `list_content_batch` the recent runs, and `revert_content` puts either back. A revert
  saves the current state first, so it can itself be reverted.
- **Nothing is published to the eshop platform.** The content stays in Pobo and the
  merchant pushes it from the administration.

Three calls destroy content, and all three answer the first call with a preview instead
of doing it: `remove_content_widget`, and `copy_content_widget` / `compose_content` in
`mode="replace"`. The preview hands back a **`confirm_token`**, and the second call has to
send it together with `confirmed=true`. The token is a fingerprint of that run, recomputed
when you confirm — so if the entity changed in between, it no longer matches and the call is
refused. That is not a permission error; it means what you were shown is no longer what
would happen, and the answer is to preview again.

Tools: [`add_content_product_carousel`](#add_content_product_carousel), [`add_content_widget`](#add_content_widget), [`check_content`](#check_content), [`compose_content`](#compose_content), [`convert_content_widget`](#convert_content_widget), [`copy_content_widget`](#copy_content_widget), [`crop_content_image`](#crop_content_image), [`delete_content_image`](#delete_content_image), [`edit_content_product_card`](#edit_content_product_card), [`edit_content_widget`](#edit_content_widget), [`generate_content_icon`](#generate_content_icon), [`generate_content_video`](#generate_content_video), [`get_content_history`](#get_content_history), [`get_content_icon_status`](#get_content_icon_status), [`get_content_video_status`](#get_content_video_status), [`get_content_widget`](#get_content_widget), [`get_content_widget_catalog`](#get_content_widget_catalog), [`get_platform_content`](#get_platform_content), [`import_content_image`](#import_content_image), [`list_content_batch`](#list_content_batch), [`list_content_image`](#list_content_image), [`list_content_media`](#list_content_media), [`list_icon`](#list_icon), [`move_content_widget`](#move_content_widget), [`remove_content_widget`](#remove_content_widget), [`render_content_html`](#render_content_html), [`revert_content`](#revert_content), [`search_content`](#search_content), [`set_content_widget_image`](#set_content_widget_image), [`update_content_meta`](#update_content_meta)

### `add_content_product_carousel`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Show related products in the description" — puts live product cards of real eshop products into the description of a product, category or blog article. layout: carousel (default, a sliding row), single (one large card per product under each other — best for 1–3 products), grid (cards in a grid), card_left / card_right (the card left or right of a text column — send text: <h2> heading and <p> paragraphs presenting the product, written by you; best with one product). Products by Pobo id (find_product / list_product) or public URL, at most 10; names, prices, photos and links come from the eshop data, never from you, and stay live — price and stock follow the eshop. position is 1-based, omit it to append. Products that do not match are reported, the rest goes in. Free, versioned, undoable with revert_content. Active HTML in text is removed; content_sanitized=true means what got stored differs from what you sent.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity the carousel goes into is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of that entity. |
| `product_id` | array of integer | no | Pobo ids of the products to show, 10 at most together with product_url; card_left and card_right take exactly one. |
| `product_url` | array of string | no | Public URLs of the products to show, as an alternative to product_id. |
| `layout` | string | no | How the cards are laid out: carousel (default), single, grid, card_left or card_right. One of: `carousel`, `single`, `grid`, `card_left`, `card_right`. |
| `text` | string | no | HTML of the text column next to the card (<h2>, <h3>, <p>). Required by card_left and card_right, not allowed with the other layouts. |
| `position` | integer | no | 1-based position in the description; omit to append at the end. Min 1. |

### `add_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Inserts one widget, filled with text you write, into a product, category or blog article. Content is keyed by the roles from get_content_widget_catalog; a repeatable widget takes {"items": [{role: value}, ...]} with DISTINCT items. position is 1-based and the rest moves down; omit it to append. The text is stored in EVERY language of the entity — edit_content_widget with lang gives one language its own wording. Active HTML (script, iframe, img, event attributes) is removed, only formatting tags survive; content_sanitized=true means what got stored differs from what you sent — check it before telling the merchant an image or embed made it in. Free, recorded under one batch_id. A whole description at once: compose_content. Hotspot widget (ai_widget_type hotspot): each item also takes x, y (0–100, percent of the photo from the left/top) and placement (above|below); 1–8 items, one point each. Look at the photo before you place points.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity. |
| `widget_id` | integer | yes | Template widget id from get_content_widget_catalog. |
| `content` | object | yes | Values keyed by role, e.g. {"paragraph": "<h2>Heading</h2><p>Body</p>"}. Repeatable widgets: {"items": [{"author": "...", "text": "..."}, ...]}. |
| `image_url` | array of string | no | https image URLs, assigned to the widget image slots in order. |
| `icon_url` | array of string | no | https icon URLs, assigned to the widget icon slots in order. |
| `position` | integer | no | 1-based position inside the entity; omit to append. Min 1. |

### `check_content`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** yes — 2 credits per checked product; the same text over the same facts again is free  
**Rate limit:** counted

"Is this text ready to go live?" — the quality gate for a few stored descriptions before push_product. Rules: no widget, widgets without text, picture or icon slots with no image (broken image on the shop), the Shoptet size limit, phrases the AI profile forbids, placeholder text (lorem ipsum, "[název produktu]"). A product that passes is read by a model against the platform data (get_product), the fact fields of the AI profile and the research: fact = a concrete product claim none of them supports, spelling = a misspelled word, filler = a sentence that tells the shopper nothing; each quotes the words and suggests a fix. passed=false lists every issue. A claim true for the whole eshop belongs in the AI profile (update_eshop_ai_profile); then check again. push_product sends only products whose current text passed here over the current facts — check again after any edit. review_unavailable = the model could not be reached, try later. 2 credits per product like generating a description (not_enough_credit when none); the same text over the same facts again is free, a review that did not run is not charged. Counts against the tariff limit. Not for the whole catalogue: start_content_bulk_check.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entities are. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | One entity. Give this or entity_id_list. |
| `entity_id_list` | array of integer | no | Several entities, 10 at most. |
| `lang` | string | no | Language version to check; omit for the default. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

### `compose_content`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

Builds the whole description of one product, category or blog article in one call: pick widgets with get_content_widget_catalog, order them, write their texts. mode="replace" (default) DELETES every widget the entity holds and leaves exactly what you send — on an entity that holds widgets it writes NOTHING until repeated with confirmed=true; an empty entity is written at once. mode="append" keeps the content and adds yours after it. The text is stored in EVERY language; edit_content_widget with lang gives one language its own wording. Active HTML (script, iframe, img, event attributes) is removed, only formatting tags survive; content_sanitized=true means what got stored differs from what you sent. The whole plan of an entity is written or none of it, under one batch_id. Several entities at once: entity — up to 20 items {"entity_id": int, "widget": [...]} instead of entity_id and widget; each reported as written, refused (with reason) or not_found, one failure does not stop the others, revert_content with the batch_id undoes the whole run. Hotspot widget (ai_widget_type hotspot): each item also takes x, y (0–100, percent of the photo from the left/top) and placement (above|below); 1–8 items, one point each. Look at the photo before you place points.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the entity. It must exist — this tool never creates one. Give entity_id + widget, or entity. |
| `widget` | array of object | no | The blocks in order, 30 at most, each {"widget_id": int, "content": {role: value}, "image": [url], "icon": [url]}. |
| `entity` | array of object | no | Several entities at once, 20 at most, each with its own blocks (same shape as widget). |
| `entity[].entity_id` | integer | yes |  |
| `entity[].widget` | array of object | yes |  |
| `mode` | string | no | replace = the entity ends up holding exactly your widgets (default; needs confirmed=true when it already holds widgets); append = your widgets go after what is already there. One of: `append`, `replace`. |
| `confirm_token` | string | no | The confirm_token the preview returned. Required together with confirmed=true. Max length 32. |
| `confirmed` | boolean | no | Required by mode=replace. Without it the call only reports how many widgets would be deleted. |

### `convert_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Use a different layout for this block" — replaces one widget with another template at the same position, carrying its texts over word for word. Read get_content_widget_catalog and pass the target widget_id — it never picks a layout for you. image_query fills the new picture slots: stock (free), uploaded (the entity library, free), image_bank (free), ai (PAID — the first call only quotes and changes nothing). Texts whose role the new template lacks are dropped and listed. Versioned, undoable with revert_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity. |
| `widget_instance_id` | integer | yes | Which widget to convert (see get_content_widget). |
| `widget_id` | integer | yes | Template to convert it into (see get_content_widget_catalog). |
| `image_query` | string | no | Photo search query (stock) or generation prompt (ai) for the new layout's picture slots. Omit to leave them empty. Max length 300. |
| `image_source` | string | no | Where the pictures come from; stock by default. One of: `stock`, `ai`, `uploaded`, `image_bank`. |
| `cost_confirmed` | boolean | no | Required by image_source="ai" after the user confirmed the quoted cost. |

### `copy_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

"Put the same blocks on these products too" — copies widgets from one product, category or blog article onto others of the same type. mode="append" adds them at the end; mode="replace" DELETES everything the target holds first and REMOVES NOTHING until repeated with confirmed=true. widget_instance_id copies only some widgets. Up to 25 product or 50 category/blog targets per call, each reported on its own; a locked or refused one does not stop the rest. One batch_id for the run. One block shared by many products that the merchant edits in one place is a shared description (list_share), not a copy.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entities are — the source and every target share it. One of: `product`, `category`, `blog`. |
| `source_entity_id` | integer | yes | Pobo id of the entity to copy FROM. |
| `target_entity_id` | array of integer | yes | Pobo ids to copy TO: 25 at most for products, 50 for categories and blogs. |
| `mode` | string | no | append = keep what the target has (default); replace = DELETE the target content first, needs confirmed=true. One of: `append`, `replace`. |
| `widget_instance_id` | array of integer | no | Copy only these widgets of the source (see get_content_widget). Omit to copy all of them. |
| `confirm_token` | string | no | The confirm_token the preview returned. Required together with confirmed=true. Max length 32. |
| `confirmed` | boolean | no | Required by mode=replace. Without it the call only reports what would be deleted. |

### `crop_content_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

"Crop the picture in this block" — crops one picture of one widget on a product, category or blog article without touching the file: the crop belongs to that slot only (the same photo elsewhere stays whole) and the CDN cuts it on the page. mode: trim = cut the plain light background around the product; aspect = the largest box of a ratio ("1:1", "4:3", "16:9") aimed at focus (center, top, bottom, left, right, or focus_x/focus_y in percent); box = an exact box in percent of the whole photo {x, y, width, height}; reset = the whole photo again. Boxes are always in percent of the whole photo, also when the slot is cropped already. image_index counts pictures as get_content_widget reports image_slot; omit it for a widget with one picture. A hotspot photo moves its points along; a crop that would leave a point out is refused. A before/after photo cannot be cropped. Free, versioned, undoable with revert_content. Returns preview_url — look at it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity. |
| `widget_instance_id` | integer | yes | Which widget (see get_content_widget). |
| `image_index` | integer | no | 1-based picture slot; omit for a widget with one picture. Min 1. |
| `mode` | string | yes | trim, aspect, box or reset. One of: `trim`, `aspect`, `box`, `reset`. |
| `aspect` | string | no | mode=aspect: width:height, e.g. "1:1", "4:3", "16:9", "3:4". |
| `focus` | string | no | mode=aspect: where the box is aimed; center by default. One of: `center`, `top`, `bottom`, `left`, `right`. |
| `focus_x` | number | no | mode=aspect: exact aim in percent from the left; wins over focus. Min 0. Max 100. |
| `focus_y` | number | no | mode=aspect: exact aim in percent from the top; wins over focus. Min 0. Max 100. |
| `box` | object | no | mode=box: percent of the whole photo from the top left corner. |
| `box.x` | number | yes |  |
| `box.y` | number | yes |  |
| `box.width` | number | yes |  |
| `box.height` | number | yes |  |

### `delete_content_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

Removes a picture from the media library of one product, category or blog article (e.g. one imported by mistake). Refused while a widget of that entity uses it — change or remove that widget first (set_content_widget_image, remove_content_widget). The file stays on the CDN, widgets elsewhere may point at it; it only leaves the list list_content_image shows.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity the picture belongs to. |
| `image_id` | integer | yes | Which picture (see list_content_image). |

### `edit_content_product_card`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Change the products in this block" — replaces the products of a product card block (carousel, single, grid, or a card next to text) already on a product, category or blog article. get_content_widget shows the block's widget_instance_id and its current product_id list; send the complete new list in the order to show (add = current list + new id, remove = list without it, reorder = same ids in a new order). Products by Pobo id or public URL, at most 10; products that do not match are reported and left out, at least one must match. The layout stays — to change it, remove the block and add_content_product_carousel with another layout. Free, versioned, undoable with revert_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity holding the block is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of that entity. |
| `widget_instance_id` | integer | yes | The product card block to change (get_content_widget). |
| `product_id` | array of integer | no | Pobo ids of the products the block shows after the edit, in order, 10 at most together with product_url; a card next to text (card_left, card_right) holds exactly one. |
| `product_url` | array of string | no | Public URLs of products, appended after product_id. |

### `edit_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Change the text of this block" — rewrites the texts of widgets already on one product, category or blog article: one by widget_instance_id, or every instance of a template by widget_id. Only the roles sent change; structure, images and other roles stay. lang picks the language and the others keep their text — this is how a translation gets its own wording. Active HTML (script, iframe, img, event attributes) is removed, only formatting tags survive; content_sanitized=true means what got stored differs from what you sent. Free, one batch_id. Several entities at once: edit — up to 50 items {"entity_id": int, "content": {...}, "widget_instance_id" or "widget_id"} instead of entity_id, content and the widget id; lang applies to all, each reported as written, refused (with reason) or not_found, revert_content with the batch_id undoes the run. Hotspot widget (ai_widget_type hotspot): each item also takes x, y (0–100, percent of the photo from the left/top) and placement (above|below); 1–8 items, one point each. Look at the photo before you place points.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the entity. Give entity_id + content + a widget id, or edit. |
| `content` | object | no | New values keyed by role. Repeatable widgets: {"items": [{role: value}, ...]} in template order. |
| `edit` | array of object | no | Several edits at once, 50 at most, each shaped like a single call. |
| `edit[].entity_id` | integer | yes |  |
| `edit[].content` | object | yes |  |
| `edit[].widget_instance_id` | integer | no |  |
| `edit[].widget_id` | integer | no |  |
| `widget_instance_id` | integer | no | Rewrite this one widget (from get_content_widget). Give this or widget_id, not both. |
| `widget_id` | integer | no | Rewrite every instance of this template on the entity. Give this or widget_instance_id, not both. |
| `lang` | string | no | Language key to rewrite; omit for the default. The other languages are left untouched. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

### `generate_content_icon`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** **yes** — one credit price per subject, quoted first, nothing drawn until `cost_confirmed=true`  
**Available:** PROFI plan or higher (AI generation)  
**Rate limit:** counted

Draws new icons for the subjects you name — the rows of a benefits or parameters widget. Check list_icon first: there is no cache, a subject the ready-made set covers is paid twice. PAID, two steps: the first call only quotes; repeat it with cost_confirmed=true (inside a Pobo agent task icons are part of the description price: no quote, no charge). Queued — returns a token, get_content_icon_status says when the files are there. A subject already drawn for this eshop comes back as existing and is not billed again.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `subject` | array of string | yes | What each icon should depict, one short noun phrase each, 10 at most. Derive them from the text the icons will stand next to, not from a guess about the product. |
| `style` | string | no | How the whole icon set should look. It is stored on the eshop set, not on this run — one eshop, one style — so sending it changes the look of icons drawn from now on. Max length 500. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost. |

### `generate_content_video`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Make a video from this photo" — turns a photo from the media library (PNG or JPEG, from list_content_media) into a short clip for a video widget. PAID, two steps: the first call only quotes; repeat it with cost_confirmed=true. A described scene (scene_wish) costs more than a camera move. Queued and takes minutes — returns a job id, get_content_video_status says when it is ready; its video_guid goes into a stream_video slot.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `media_file_id` | integer | yes | The photo to animate — a file id from list_content_media. |
| `preset` | string | no | Camera move: orbit, push_in, parallax, rotate, float. Leave it out to let the planner pick one from the photo. |
| `prompt` | string | no | Your own wording of the motion, instead of a preset. Max length 500. |
| `scene_wish` | string | no | What should happen in the scene (for example "model takes a sip"). Sending it makes this a scene run, which costs more. Max length 500. |
| `resolution` | string | no | One of 480p, 540p, 720p, 768p, 1080p. Defaults to 720p. |
| `loop` | boolean | no | Whether the clip should loop seamlessly. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost. |

### `get_content_history`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What did this description look like before?" — versions of one product, category or blog article, newest first: version_id, when, which channel and tool caused it, widget count, batch_id. A version is the state BEFORE that write, so reverting to it undoes the write; the editor writes versions too. Pass version_id to revert_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the product (list_product), category (list_category) or blog article (list_blog). |
| `limit` | integer | no | How many versions to return, 50 at most (default 20). |

### `get_content_icon_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

A run of generate_content_icon: icons drawn and their URL, ones already in the set, failures and why, and credits used. A drawn icon goes into a widget as icon_url (add_content_widget) or icon (compose_content). Only drawn icons are billed. wait (up to 60 s) holds the answer until the run is done.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `token` | string | yes | The token generate_content_icon returned. Max length 64. |
| `wait` | integer | no | Seconds to wait for the run to finish before answering, 0-60. Default 0 answers at once. |

### `get_content_video_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

A clip started by generate_content_video. Once usable=true put its video_guid into a stream_video slot (add_content_widget or compose_content). Completed but not usable = still transcoding — wait, a video that is not ready renders as an empty player.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `id` | integer | yes | Id returned by generate_content_video. |

### `get_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What is in the description now?" — what a product, category or blog article holds: ordered widgets with position, widget_instance_id, widget_id, template name, image slot count and texts by role. Read it before editing, moving or removing — widget_instance_id comes from here only. entity_id for one, entity_id_list for up to 25 (to look at a batch before copying onto it). Each entity reports per language how many text values are filled and how many empty.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the product (list_product), category (list_category) or blog article (list_blog). Give this or entity_id_list. |
| `entity_id_list` | array of integer | no | Several entities at once, 25 at most — the alternative to entity_id. |
| `lang` | string | no | Which language of the texts to show; omit for the default. Reading never writes. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

### `get_content_widget_catalog`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The widget templates this eshop can place on a product, category or blog article (the same list for all three). Each entry: text roles (role, hint, max_length), image and icon slot count, item container count. Pass its widget_id to add_content_widget or compose_content; never guess one. Write the texts in write_in_language — roles and hints are in English, the content is not. With product_id the catalog leaves out widgets with more picture slots than the product has gallery photos (empty slots are broken images); run sync_product_image first if the gallery is empty.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What you are going to build. The catalog is identical for all three. One of: `product`, `category`, `blog`. |
| `product_id` | integer | no | The product you are writing; limits picture widgets to what its gallery can fill. |

### `get_platform_content`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What does the customer read on the eshop today?" — the description the entity has on the shop platform itself, as opposed to Pobo content. Read it before rewriting: it often holds facts that exist nowhere else (dimensions, materials, compatibility, what is in the box) and a rewrite that ignores them drops them. It is Pobo's copy from the last download (get_import_status), not the live page (fetch_eshop_page). Tags stripped, text shortened: original_length and truncated say how much you see. Also the platform name and SEO fields (what update_content_meta often refuses to change) and for a category the page heading name_heading (null = the name).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity. |
| `lang` | string | no | Which language version to read; omit for the default. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `length` | integer | no | How many characters of the description to return, 40000 at most (default 6000). Raise it only when the shortened text is not enough. Min 500. |

### `import_content_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Copies a picture from a public https URL into the media library of one product, category or blog article and returns its Pobo CDN url — files cannot be sent to this server, name the source. Up to 3 redirects, each checked; private addresses refused; max 10 MB. Then set_content_widget_image with image_source="uploaded" and that url puts it into a widget.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity the picture belongs to. |
| `image_url` | string | yes | https URL of the image file. A short redirect chain (for instance a shop CDN sending you on to the shop domain) is followed. |

### `list_content_batch`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Undo what was changed" — recent runs of content changes on products, categories and blog articles, newest first: batch_id, tool and channel, when, how many entities and a few of them. Editor runs are listed too. revert_content with the batch_id puts every entity of the run back.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `tool` | string | no | Only runs made by this tool, e.g. "compose_content". Max length 64. |
| `limit` | integer | no | How many runs to return, 50 at most (default 20). |

### `list_content_image`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

Pictures stored on one product, category or blog article: id, CDN url, format — what image_source="uploaded" puts into a widget. Add one with import_content_image.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the product (list_product), category (list_category) or blog article (list_blog). |

### `list_content_media`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which photos and videos did I upload?" — folders and files of the eshop media library (list_content_image is one entity's pictures). Without folder_id the folders with file counts, with one its files and URLs; videos carry their encoding status. Prefer a picture the merchant owns over stock or a paid generation.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `folder_id` | integer | no | List the files of this folder. Omit it to list the folders themselves. |
| `limit` | integer | no | How many files to return, 200 at most (default 50). |

### `list_icon`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The ready-made icons this eshop can use, free curated files. Widgets with icon slots (icon_slot in get_content_widget_catalog) take their URL as icon_url (add_content_widget) or icon (compose_content). Without category_id the categories with a sample each, with one that category; icon_name finds by name across categories. Pick by name (Czech, as the merchant sees it — "Lupa", "Garance kvality"), never by URL. Draw new ones only when nothing fits: generate_content_icon.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `category_id` | integer | no | List one category. Omit it to get every category with a sample of its icons. |
| `query` | string | no | Only categories whose name contains this. Max length 64. |
| `icon_name` | string | no | Only icons whose name contains this, searched across every category (names are Czech, e.g. "auto", "kufr", "voda"). Max length 64. |
| `limit` | integer | no | How many icons per category, 200 at most (default 60). |

### `move_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Moves one widget to another place in its product, category or blog article. Positions are 1-based, position 1 puts it first; nothing is deleted, no text changes. One batch_id.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity. |
| `widget_instance_id` | integer | yes | Widget instance id from get_content_widget. |
| `position` | integer | yes | 1-based target position. A number past the end lands the widget last. Min 1. |

### `remove_content_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

"Remove this block" — DELETES widgets from one or more products, categories or blog articles: by widget_instance_id (one), widget_id (every instance of a template) or query (every widget whose text contains it); entity_id for one or entity_id_list for up to 10, the selector applied to each. The first call REMOVES NOTHING — it answers with what would go; repeat with confirmed=true AND its confirm_token. A locked entity is reported and skipped. One batch_id; the remaining widgets close the gap.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the entity. Give this or entity_id_list. |
| `entity_id_list` | array of integer | no | Several entities at once, 10 at most — the alternative to entity_id. |
| `widget_instance_id` | integer | no | Remove this one widget (from get_content_widget). |
| `widget_id` | integer | no | Remove every instance of this template on the entity. |
| `query` | string | no | Remove every widget whose text contains this, ignoring case and diacritics. Max length 255. |
| `confirm_token` | string | no | The confirm_token the preview returned. Required together with confirmed=true. Max length 32. |
| `confirmed` | boolean | no | false or omitted = preview only, nothing is deleted. true = delete. |

### `render_content_html`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Is the description OK?" — renders the stored description of one product, category or blog article and reports its quality: byte size, widget count, widgets without text, picture or icon slots with no image (a broken image on the shop — set_content_widget_image or a widget without that slot), and whether a Shoptet product is over the 65 000 byte limit that blocks writes. The final check after building or editing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | yes | Pobo id of the entity. |
| `include_html` | boolean | no | false returns only the report, without the rendered HTML — enough for a final check and much shorter. Default true. |
| `lang` | string | no | Language to render; omit for the default. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

### `revert_content`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

"Undo it / put it back" — one entity to a version_id from get_content_history, or a whole run by batch_id from list_content_batch (every product, category and article it touched goes back). The current state is saved first and the revert has its own batch_id, so a revert can be reverted. A locked entity (a cross-eshop mirror target) is refused on its own and skipped inside a batch, never overwritten.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | no | What the entity is. Give it together with entity_id and version_id. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the entity to revert. |
| `version_id` | integer | no | Version to go back to (see get_content_history). |
| `batch_id` | string | no | Revert a whole run instead of one entity — every entity it touched goes back to the state before it. Max length 26. |

### `search_content`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which descriptions still mention …?" — finds products, categories or blog articles whose Pobo description (widget text, which list_product does not search) contains a phrase, ignoring case and accents, in every language. scope "all" also searches the platform description, name, short description, category, brand, variant names and parameter values — use it when the question is about the product, not the text Pobo wrote. total_count, when present, is the exact number even past the limit; without it very large eshops are refused rather than half-searched. Products of a kind (brand, property): search_product.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What to search through. One of: `product`, `category`, `blog`. |
| `query` | string | yes | The phrase to look for. Case and accents are ignored. Max length 200. |
| `scope` | string | no | "pobo" (default): only the Pobo description. "all": also the platform description, name, categories, brand, variants and parameters. One of: `pobo`, `all`. |
| `limit` | integer | no | How many entities to return, 100 at most (default 25). |

### `set_content_widget_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Change the picture in this block" — replaces the picture(s) of one widget without touching its texts. Sources: stock (Pexels, free), uploaded (the entity library, free — image_query names pictures by id, url or file name from list_content_image, several comma-separated fill the slots in order; free text takes the newest), image_bank (eshop public directory, free), ai (generated, PAID — the first call only quotes and writes nothing). image_index = one slot, counted as get_content_widget reports image_slot (every picture in document order, row icons included); omit it to refill all. Versioned, undoable with revert_content. Several widgets at once: item — up to 20 {"entity_id", "widget_instance_id", "image_query", "image_index"?} instead of the single fields; image_source applies to all, an ai batch is quoted as a whole, each reported as written, refused or not_found. A hotspot widget keeps its points where they were — check them against the new photo and move them with edit_content_widget.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the entity. Give entity_id + widget_instance_id + image_query, or item. |
| `widget_instance_id` | integer | no | Which widget to change (see get_content_widget). |
| `image_query` | string | no | Photo search query (stock), generation prompt (ai), or for uploaded the pictures to use: id, url or file name from list_content_image, comma-separated for several slots. image_bank ignores it. Max length 300. |
| `item` | array of object | no | Several widgets at once, 20 at most. |
| `item[].entity_id` | integer | yes |  |
| `item[].widget_instance_id` | integer | yes |  |
| `item[].image_query` | string | yes |  |
| `item[].image_index` | integer | no |  |
| `image_source` | string | no | Where the pictures come from; stock by default. One of: `stock`, `ai`, `uploaded`, `image_bank`. |
| `image_index` | integer | no | 1-based picture slot to replace; omit to replace all of them. Min 1. |
| `cost_confirmed` | boolean | no | Required by image_source="ai" after the user confirmed the quoted cost. |

### `update_content_meta`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

"Rename it / change the SEO title" — sets name, seo_title, seo_description and (products only) short_description of one product, category or blog article in one language; on platform pobo a category also takes name_heading, the H1 of its page (longer than the menu name; empty string clears it). A field the eshop platform owns in the export settings is refused — the next import would overwrite it; refused fields are named, the rest is written. The URL is never writable. Nothing goes to the eshop until push_product. Several entities at once: entity — up to 50 items {"entity_id", "name"?, "seo_title"?, "seo_description"?, "short_description"?, "name_heading"?}; each reported as written, refused or not_found.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`. |
| `entity_id` | integer | no | Pobo id of the entity. Give entity_id with the fields, or entity. |
| `entity` | array of object | no | Several entities at once, 50 at most, each with its own fields. |
| `entity[].entity_id` | integer | yes |  |
| `entity[].name` | string | no |  |
| `entity[].seo_title` | string | no |  |
| `entity[].seo_description` | string | no |  |
| `entity[].short_description` | string | no |  |
| `entity[].name_heading` | string | no |  |
| `lang` | string | no | Which language to write; omit for the default one. Other languages keep their own text. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `name` | string | no | Entity name. Max length 255. |
| `seo_title` | string | no | Title tag. Max length 255. |
| `seo_description` | string | no | Meta description. Max length 500. |
| `short_description` | string | no | Short description; products and categories only. Max length 2000. |
| `name_heading` | string | no | Storefront page heading (H1); categories on platform pobo only. Empty string = show the name. Max length 255. |

## Content quality — checks, fixes and translations

Three paid, quote-first runs over the catalogue. **Content check** reads descriptions against the merchant's own rules (and optional legal packages) and reports findings with quoted words; **content fix** rewrites only the quoted words of those findings and leaves the products as drafts; **translation check** compares every translation with its source. `estimate_content_bulk_check` and the first call of every `start_*` tool only quote the price — nothing is spent until the call is repeated with `cost_confirmed=true`.

`check_content` is different: a free check of one entity before `push_product`, which refuses products that failed it or were never checked.

Tools: [`estimate_content_bulk_check`](#estimate_content_bulk_check), [`fix_translation`](#fix_translation), [`get_content_bulk_check`](#get_content_bulk_check), [`get_content_fix`](#get_content_fix), [`get_translation_check`](#get_translation_check), [`get_translation_finding`](#get_translation_finding), [`list_content_bulk_check`](#list_content_bulk_check), [`list_content_check_preset`](#list_content_check_preset), [`recheck_content_fix`](#recheck_content_fix), [`start_content_bulk_check`](#start_content_bulk_check), [`start_content_fix`](#start_content_fix), [`start_translation_check`](#start_translation_check)

### `estimate_content_bulk_check`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How much would it cost to check my texts?" — the price of a bulk content check before it starts: entities and items (entity × language) in the scope and the most it costs (1 credit per 50 items read, items without text free; a rule with a legal pack 1 credit per 3 items — pass instruction for that price, credit_without_legal is the price without them). Free, changes nothing. Next: start_content_bulk_check with the same scope and expected_credit = credit. Not for a few texts before upload (check_content) and not for translations (start_translation_check).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What to check: products, categories or articles. The whole catalogue of that kind is checked. One of: `product`, `category`, `blog`. |
| `field` | array of string | no | Texts to read: name, short_description (products only), description (the description the shop shows — the Pobo one when it is live there, otherwise the original), seo_title, seo_description, platform_description (the original description even when a Pobo one replaced it on the shop — only when asked for). Omit for all but platform_description. One of: `name`, `short_description`, `description`, `platform_description`, `seo_title`, `seo_description`. |
| `lang` | array of string | no | Languages to read; "default" is the main language of the eshop. Omit for the main language only. Each language of each entity counts as one item. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `with_pobo_content` | boolean | no | true = only entities that have a description made in Pobo. Default false. |
| `category_id` | array of integer | no | Products only: check just these categories (see list_category). |
| `with_fact` | boolean | no | Products only: send the parameters and variants along, so a rule can ask about contradictions with them. Default false. |
| `instruction` | string | no | Optional: the rules the check will use, one per line — a ready-made legal rule among them changes the price. |

### `fix_translation`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Translate what is missing / fix the bad translations" — per item and language translates again the areas (name, short description, SEO, description) that miss a translation or have an open finding of the AI translation check, and closes those findings; products, categories, blog articles or shared descriptions. Stale translations (the main text changed since) may hold the merchant's own edits — only with include_stale=true. Items: entity_id (from list_product / list_category / list_blog with translation=attention), or all=true for the next 500 items per kind with something missing or an open finding of the AI translation check — "translate the whole eshop into Slovak" or "fix what the check found": leave entity_type out to take products, categories, blog articles and shared descriptions in ONE call (one price, one approval; by_type splits it), and repeat until fix_count is 0; items still being translated come back as running_count — wait, then call again. PAID, two steps: without cost_confirmed it only quotes (fix_count, translate_count, credit_required, stale_count, running_count); call again with the same arguments, expected_credit = credit_required and cost_confirmed=true. Charged like a translation from the editor, after it ran.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | no | What the items are. Leave out with all=true to take every kind of content at once. One of: `product`, `category`, `blog`, `share`. |
| `entity_id` | array of integer | no | Pobo ids of the items, 500 at most. Leave out with all=true. |
| `all` | boolean | no | Instead of entity_id: the next 500 items of entity_type with something missing or an open finding of the AI translation check. Default `false`. |
| `lang` | array of string | no | Languages to fix; without it every translation language of the eshop. One of: `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `include_stale` | boolean | no | Translate again also stale translations — they may carry manual edits. Only when the user asked for it. Default `false`. |
| `expected_credit` | integer | no | credit_required of the first call, which the user agreed to. |
| `cost_confirmed` | boolean | no | true once the user agreed to the price. Default `false`. |

### `get_content_bulk_check`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What did the content check find?" — state and findings of a bulk content check; without check_id the latest one. status: pending, running, finished, finished_partial (some batches unread and not charged), failed, cancelled. Done: score (share of items without a finding), grade 1–5, the most common problems, and a page of findings — entity, field, language, severity, rule line, the exact quote, what is wrong, why it matters, a suggested fix, legal_ref for a legal rule; widget_instance_id is the description block for get_content_widget / edit_content_widget. Only the first 10 findings of a check are ever served (finding_visible_limit); finding_total and severity_count are the full numbers. wait (up to 60 s) holds the answer until the check is done.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `check_id` | integer | no | The check_id start_content_bulk_check returned; omit for the latest check. |
| `severity` | string | no | Only findings of this severity. One of: `high`, `medium`, `low`. |
| `offset` | integer | no | Findings to skip. Default 0. |
| `limit` | integer | no | Findings per page, 1-100. Default 50. |
| `wait` | integer | no | Seconds to wait for the check to finish before answering, 0-60. Default 0 answers at once. |

### `get_content_fix`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How did the fix go?" — state and result of a fix (start_content_fix): fix_id, or check_id for the latest fix of that check. status: pending, running, finished, failed, cancelled; item_done of item_count. Done: result.fixed and result.skipped (per reason), credit_spent, and a page of changes — product, field, language, old_text, new_text. comparison is the check of the fixed products again (recheck_content_fix): before, resolved, persisted, new — new findings are the check reading the same text differently, not caused by the fix. wait (up to 60 s) holds the answer until the fix is done.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `fix_id` | integer | no | The fix_id start_content_fix returned. |
| `check_id` | integer | no | Instead of fix_id: the latest fix of this check. |
| `offset` | integer | no | Changes to skip. Default 0. |
| `limit` | integer | no | Changes per page, 1-100. Default 20. |
| `wait` | integer | no | Seconds to wait for the fix to finish before answering, 0-60. Default 0 answers at once. |

### `get_translation_check`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How far is the translation check / what did it find?" — progress and outcome of the AI translation check. With run_id one run: status (pending, running, finished, failed, cancelled), units done of unit_count, findings, credit reserved and spent, kind_count = findings per kind (wrong_language, partial, missing_sentence, extra_sentence, translated_term), entity_with_finding = how many products, categories, articles and shared descriptions still have an open finding — the number of items to report; never take it from the totals of list_product or list_category, those count the whole catalogue. Without run_id the latest runs. Free, changes nothing. The findings per item: list_product / list_category / list_blog with translation=attention, then get_translation_finding.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `run_id` | integer | no | Run id from start_translation_check; without it the latest runs. |

### `get_translation_finding`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What is wrong with the translation of this item?" — the translation state of one product, category, blog article or shared description per language: state (translated, partial, stale = the main text changed since, none), units translated, what is missing (field, widget, kind) and the open findings of the AI translation check — the translated words (quote), the main-language words they belong to (source_quote) and the kind: wrong_language, partial, missing_sentence, extra_sentence, translated_term (a name that should stay untranslated). Free, changes nothing. fix_translation translates it again.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `share`. |
| `entity_id` | integer | yes | Pobo id of the entity. |

### `list_content_bulk_check`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which checks did I run?" — the eshop's bulk content checks, newest first: check_id, status, scope, items and findings, score and grade once done, and the latest fix of each (fix_id, status, findings fixed); recheck_of_fix_id marks a check that verified a fix. Findings of one check: get_content_bulk_check; fixing them: start_content_fix.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `offset` | integer | no | Checks to skip. Default 0. |
| `limit` | integer | no | Checks per page, 1-50. Default 20. |

### `list_content_check_preset`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What should I check my texts for?" — the ready-made rules the content check page offers, grouped by what they protect (geo = AI assistant recommendations, seo, law = legal risk, brand = tone and trust, data = catalogue data such as photos): rule text for one instruction line, why it matters, when it fits, usual severity for rule_severity. needs_fact=true compares texts with product parameters (start the check with with_fact=true). A rule with legal_pack is checked against the official text of the law and its findings carry legal_ref; a rule with data_rule (e.g. missing_image) is answered from catalogue data, free and exact — pass both kinds unchanged on their own line, or they are not recognised. industry says which rules fit a kind of shop. Offer the rules that fit, then pass the chosen texts as instruction.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `recheck_content_fix`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Did the fix work?" — checks again only the products, fields and languages a finished fix changed, with the same rules, and says for every earlier finding whether it is resolved or still there; anything else is reported as new. This, not a new start_content_bulk_check, measures a fix: a fresh check of the same text reports different findings run to run. PAID, two steps: without cost_confirmed it only quotes (1 credit per started 50 items); call again with expected_credit = the quote and cost_confirmed=true. Result in get_content_fix → comparison; its findings in get_content_bulk_check with the returned check_id.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `fix_id` | integer | yes | The finished fix to verify. |
| `expected_credit` | integer | no | The price the user confirmed. Second call only. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted price. Second call only. |

### `start_content_bulk_check`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Check my products / texts" — AI reads every text of the scope against the merchant's rules in plain words and reports each place that breaks them: the exact quote, what is wrong, why it matters, a suggested fix. The default for a general "check my products". PAID, two steps: without cost_confirmed it only quotes (as estimate_content_bulk_check); call again with the same scope, expected_credit = the quote and cost_confirmed=true. Credits are reserved at the start, what was not read comes back. Rules go in instruction, one per line (list_content_check_preset has tested ones); rule_severity gives each line high, medium or low; with_ai_profile adds the rules of the AI profile. One check per eshop at a time, it takes minutes — follow it with get_content_bulk_check. Not for translations: start_translation_check tells text left in another language from a wrong translation and fix_translation translates again, while a content fix only swaps quoted words. Not for a few texts before upload: check_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | What to check: products, categories or articles. The whole catalogue of that kind is checked. One of: `product`, `category`, `blog`. |
| `field` | array of string | no | Texts to read: name, short_description (products only), description (the description the shop shows — the Pobo one when it is live there, otherwise the original), seo_title, seo_description, platform_description (the original description even when a Pobo one replaced it on the shop — only when asked for). Omit for all but platform_description. One of: `name`, `short_description`, `description`, `platform_description`, `seo_title`, `seo_description`. |
| `lang` | array of string | no | Languages to read; "default" is the main language of the eshop. Omit for the main language only. Each language of each entity counts as one item. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `with_pobo_content` | boolean | no | true = only entities that have a description made in Pobo. Default false. |
| `category_id` | array of integer | no | Products only: check just these categories (see list_category). |
| `with_fact` | boolean | no | Products only: send the parameters and variants along, so a rule can ask about contradictions with them. Default false. |
| `instruction` | string | yes | The rules to check against, one per line, in the merchant's words. |
| `rule_severity` | array of string | no | Optional fixed severity per line of instruction, in the same order; leave a line out by ending the list early. One of: `high`, `medium`, `low`. |
| `with_ai_profile` | boolean | no | true = add the rules the eshop AI profile already states (forbidden phrases, tone, emoji, delivery and return terms) to instruction. The answer lists the added rules. |
| `expected_credit` | integer | no | The price the user confirmed. Second call only. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted price. Second call only. |

### `start_content_fix`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Fix the errors the check found" — rewrites only the quoted words of each finding of a finished product check; the rest of the text stays and every fixed product is saved as a draft for the merchant to approve (nothing reaches the eshop). Skipped with a reason: the platform's own description (never changed), missing data (never made up), a field the eshop manages itself, a quote changed since the check, a suggested fix equal to the quote. Findings in a translation that is a copy of the main language are not rewritten word by word: translate=true (default) translates those products again (priced in translate.credit, charged as translations), translate=false skips them. research=true also looks up missing data on the web into the parameter block (research.credit, at most 2 credits per product). PAID, two steps: without cost_confirmed it only quotes — what is fixed, skipped and price; call again with expected_credit = price and cost_confirmed=true. 1 credit per 10 fixed findings, the rest of the reservation comes back. One fix per eshop at a time; follow it with get_content_fix.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `check_id` | integer | yes | The finished check to fix (list_content_bulk_check, get_content_bulk_check). |
| `research` | boolean | no | true = also fill in missing data from web research into the parameter block. Off by default. |
| `translate` | boolean | no | true (default) = translate again the products whose translation is a copy of the main language; false = skip those findings. |
| `expected_credit` | integer | no | The price the user confirmed. Second call only. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted price. Second call only. |

### `start_translation_check`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Rate limit:** counted

"Check my translations" — AI compares every translated product, category, blog article or shared description × language with its main-language text and finds text left in another language, partial translations, missing or extra sentences and names that should stay untranslated; the findings land on the items. Use it for anything about translations or languages, not start_content_bulk_check. PREMIUM and up. PAID, two steps: without cost_confirmed it only quotes (unit_count, credit, credit_remaining); call again with the same arguments, expected_credit = credit and cost_confirmed=true. Only units sent are charged; only_changed=true skips units unchanged since the last check (the quote stays the full price, the unsent part comes back). One check per eshop at a time. Then get_translation_check; once finished list_product / list_category / list_blog with translation=attention, get_translation_finding per item and fix_translation to translate again.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | array of string | no | What to check; without it products, categories, blog articles and shared descriptions. One of: `product`, `category`, `blog`, `share`. |
| `lang` | array of string | no | Translation languages to check; without it every translation language of the eshop. One of: `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `field` | array of string | no | Areas to check (name, short_description, seo_title, seo_description, content); without it all of them. One of: `name`, `short_description`, `seo_title`, `seo_description`, `content`. |
| `only_changed` | boolean | no | Skip units whose texts did not change since the last check. Default `false`. |
| `expected_credit` | integer | no | credit of the first call, which the user agreed to. |
| `cost_confirmed` | boolean | no | true once the user agreed to the price. Default `false`. |

## Blog articles

Creating and listing blog articles, and Pobo's own server-side article generator (`generate_blog_article`, paid). Writing an article yourself goes through the content tools above with `entity_type: "blog"` and is free.

Tools: [`create_blog`](#create_blog), [`generate_blog_article`](#generate_blog_article), [`get_blog_generate_status`](#get_blog_generate_status), [`list_blog`](#list_blog)

### `create_blog`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Start a new blog article" — creates an empty article in Pobo (manual, not paired with the platform) and returns blog_id; write it with compose_content (entity_type=blog). To have the whole article written for you: generate_blog_article.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `name` | string | yes | Article title. Max length 255. |

**Returns:** `blog_id` and `name`.

### `generate_blog_article`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Available:** PROFI plan or higher (AI generation)  
**Rate limit:** counted

"Write a blog article about …" — generates a whole article from a brief on the server (async — poll get_blog_generate_status). 1 credit per article + 2 credits per AI photo (photo_source=ai needs image_count and cost confirmation). Free alternative: create_blog and write the texts yourself with compose_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `blog_id` | integer | yes | Blog id (see list_blog / create_blog). |
| `brief` | string | yes | What the article should be about. Min length 5. Max length 5000. |
| `normpages` | integer | no | Article length in standard pages. Default `3`. Min 1. Max 10. |
| `structure_mode` | string | no | ai lets the planner pick the structure; template follows a blog design (requires design_id). One of: `ai`, `template`. Default `"ai"`. |
| `design_id` | integer | no | Blog design id, required for structure_mode=template. Min 1. |
| `photo_source` | string | no | Photo strategy: none, stock (free), uploaded (free), product (free), ai (PAID — requires image_count and cost confirmation). One of: `none`, `ai`, `stock`, `uploaded`, `product`. Default `"stock"`. |
| `image_count` | integer | no | Number of image widgets; omit to size automatically. Required for photo_source=ai. Min 0. Max 10. |
| `product_id` | array of integer | no | Related Pobo product ids to weave into the article. Max items 10. |
| `product_url` | array of string | no | Related public product URLs. Max items 10. |
| `inspiration_url` | array of string | no | Pages to distill as factual inspiration. Max items 3. |
| `tone` | string | no | Optional tone of voice. Max length 100. |
| `remove_exist_widget` | boolean | no | Replace the current article content instead of appending. Default `false`. |
| `lang` | string | no | Article language; omit for the eshop default. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost (needed for photo_source=ai). Default `false`. |

**Returns:** `job_id`, `blog_id`, `status`, and a message pointing to `get_blog_generate_status` for polling.

**Careful:** `remove_exist_widget=true` replaces the article's existing content instead of appending to it. The tool also refuses to run when the AI-content-generation kill switch is off, and when `structure_mode=template` without `design_id`.

### `get_blog_generate_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Is the article done?" — a generation started by generate_blog_article, step by step: preparing, writing, selecting_photos, composing, done.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `blog_id` | integer | yes | Blog id the job was started for. |
| `job_id` | integer | yes | Job id returned by generate_blog_article. |

**Returns:** `job_id`, `blog_id`, `status`, `progress`, `step`, `widgets_created`, `error_message`, `title` (from the generated response, when available).

### `list_blog`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

Browse blog articles the way the admin blog grid does: id, name, widget count, platform pairing and translation state per language; filters filter (all, without_description, edited, favourite, waiting_for_approval = drafts, concept = saved but not published, recently_edited), name search, label (or without_label), origin, author (or without_author) and the translation check (translation = ok / attention). The id is blog_id for the other tools. Which articles mention something: search_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `filter` | string | no | Grid filter: without_description = no Pobo content yet, edited = has Pobo content, favourite, waiting_for_approval = status draft, concept = an edit saved but not published yet (status concept); recently_edited orders the full list. One of: `all`, `without_description`, `edited`, `favourite`, `waiting_for_approval`, `concept`, `recently_edited`. Default `"all"`. |
| `query` | string | no | Optional name search. |
| `label_id` | array of integer | no | Filter by label ids (see list_label). |
| `without_label` | boolean | no | Only articles with no label. Wins over label_id, like in the admin grid. Default `false`. |
| `origin` | array of string | no | imported = loaded from the eshop platform, manual = written in Pobo. One of: `imported`, `manual`. |
| `author_id` | array of integer | no | Filter by author ids. |
| `without_author` | boolean | no | Only articles no author signs. Default `false`. |
| `translation` | string | no | Translation check filter, as in the admin grid: ok = translated, nothing missing, no open finding of the AI translation check; attention = anything else (not translated, partly, out of date, or an open finding). Read the findings of one item with get_translation_finding. One of: `ok`, `attention`. |
| `translation_lang` | string | no | Language of the translation filter (a translation language of the eshop, see list_eshop), or "any". With "any", ok needs every language in order and attention any one of them. Default `"any"`. |
| `limit` | integer | no | Page size. Default `20`. Min 1. Max 100. |
| `page` | integer | no | 1-based page number. Default `1`. Min 1. |

**Returns:** `blog` — array of `{id, name, widget_count, is_visible, platform_article_id}` — plus `total`, `page`, `limit`.

## Prompts, designs and generation

Prompt profiles decide how Pobo generates descriptions; designs are the widget templates the generation writes into. `preview_generation` tries a prompt on 1–3 real products without writing anything to them, `start_generation` runs the real thing. Both are AI generation and need the PROFI plan or higher; an extended template (`custom` prompt or one with a design) needs SAAS.

Tools: [`create_prompt`](#create_prompt), [`delete_prompt`](#delete_prompt), [`get_generation_status`](#get_generation_status), [`get_preview_status`](#get_preview_status), [`get_prompt`](#get_prompt), [`get_prompt_history`](#get_prompt_history), [`list_design`](#list_design), [`list_design_widget`](#list_design_widget), [`list_prompt`](#list_prompt), [`preview_generation`](#preview_generation), [`set_widget_prompt`](#set_widget_prompt), [`start_generation`](#start_generation), [`update_prompt`](#update_prompt)

### `create_prompt`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Make a prompt in my style" — creates a custom AI generation prompt on an eshop, optionally with a design (list_design); per-widget instructions come after with set_widget_prompt. Try it with preview_generation.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `name` | string | yes | Profile name shown in the merchant admin. Min length 5. Max length 255. |
| `prompt` | string | yes | The generation prompt text — general instructions applied to the whole product description. Min length 20. Max length 1000000. |
| `design_id` | integer | no | Optional design (template) the prompt generates into (see list_design). Required before per-widget instructions can be set. |
| `icon_category_id` | integer | no | Optional icon category restricting AI icon selection. |
| `photo_mode` | string | no | How a template generation fills its photos: legacy = gallery order (default); smart = AI picks photos by what they show, places the hotspot points and draws only missing photos; profi = smart, plus AI sharpens and fits the eshop photos to each block and checks the drawn icons (each photo the model draws costs an image credit). Only for a prompt with a design. One of: `legacy`, `smart`, `profi`. |
| `fact_coverage` | boolean | no | Keep the facts of the original description: AI checks the new text against the product's original description (from 1 000 characters) and writes lost facts back into the blocks. About half a minute longer per product. Only for a prompt with a design. |

**Returns:** `action: "created"` plus the prompt detail — `id`, `name`, `prompt`, `design_id`, `icon_category_id`, `icon_mode`, `widget_prompt` (array of `{design_widget_id, prompt}`), `created_at`, `updated_at`.

### `delete_prompt`

**Server:** merchant (`/mcp/client`)  
**Access:** writes — destructive  
**Costs credits:** no  
**Rate limit:** counted

Deletes a custom prompt and its per-widget instructions.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `prompt_id` | integer | yes | Prompt id (see list_prompt). |

**Returns:** `action` (`"deleted"` if the prompt belonged to no other eshop, `"detached"` if it is shared and only the link to this eshop is removed) and `id`.

**Careful:** destructive — when `action` is `"deleted"` the prompt itself is soft-deleted, not just unlinked.

### `get_generation_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How far is the generation?" — a generation started by start_generation: products generated, running or skipped (list_generation / get_generation for one product), or for categories generated, running or failed. status: draft (never started), running, done. wait (up to 60 s) holds the answer until it is done.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `generation_id` | string | yes | The generation_id start_generation returned. |
| `wait` | integer | no | Seconds to wait for the generation to finish before answering, 0-60. Default 0 answers at once. |

### `get_preview_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Show me the prompt try-out" — a try-out started by preview_generation, per product pending, running, complete or failed; a complete one returns the rendered HTML, SEO fields and name, AI images still coming and the credits its text and images cost (icons and research are charged on their own). Try-outs are kept 30 days.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `token` | array of string | yes | Tokens returned by preview_generation, max 3. |

**Returns:** `preview` — one entry per token, `status` of `pending`, `complete`, or `not_found` (expired after 30 minutes, never existed, or belongs to another eshop). A complete entry includes the rendered HTML, generated SEO fields, and any error.

### `get_prompt`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

One custom prompt in full: text, linked design and its per-widget instructions.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `prompt_id` | integer | yes | Prompt id (see list_prompt). |

**Returns:** `id`, `name`, `prompt`, `design_id`, `icon_category_id`, `icon_mode`, `widget_prompt` (array of `{design_widget_id, prompt}`), `created_at`, `updated_at`.

### `get_prompt_history`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

Earlier versions of a custom prompt's text, newest first — check what was there before overwriting it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `prompt_id` | integer | yes | Prompt id (see list_prompt). |
| `limit` | integer | no | Maximum number of snapshots to return. Default `10`. Min 1. Max 50. |

**Returns:** `prompt_id` and `history` — array of `{id, prompt, user, created_at}`, newest first.

### `list_design`

**Server:** merchant (`/mcp/client`) and white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which description templates can I use?" — designs (widget templates) usable as a prompt's design_id: public Pobo templates and the eshop's own.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

**Returns:** `design` — array of `id`, `name`, `type`, `widget_count`.

### `list_design_widget`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

The blocks of one design in display order (id, position, widget name); id is the design_widget_id set_widget_prompt takes.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `design_id` | integer | yes | Design id (see list_design). |

**Returns:** `design_id`, `design_name`, `widget` — the design's widget summary (id, position, widget name); `id` here is the `design_widget_id` used by `set_widget_prompt`.

**Careful:** this is a light listing only (id/position/name) — the fully rendered widget HTML used for the admin UI preview is not available through this tool.

### `list_prompt`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which prompts do I have?" — the eshop's custom generation prompts: id, name, linked design, for_category (its template writes category descriptions), per-widget instruction count. Full text: get_prompt.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

**Returns:** `prompt` — array of `{id, name, design_id, design_name, widget_prompt_count, updated_at}`.

**Careful:** the full prompt text is deliberately excluded (it can be up to 1 MB) — use `get_prompt` to read it.

### `preview_generation`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Available:** PROFI plan or higher (AI generation)  
**Rate limit:** counted

"Try this prompt on a few products first" — generates 1–3 products exactly as a real generation (text, SEO, AI images, icons) WITHOUT writing anything to the products. Without design_id it uses the template of prompt_id, without both a basic Pobo layout. PAID, two steps: without cost_confirmed it only quotes the most it can cost; call again with the same settings, expected_credit = the quote and cost_confirmed=true. One try-out per eshop at a time; poll the token with get_preview_status. Use it before update_prompt makes a change permanent.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `design_id` | integer | no | Design (widget template) to generate into (see list_design). Omit to use the template of prompt_id; omit both for a basic template. |
| `prompt_id` | integer | no | The saved prompt being tuned, if any (see list_prompt). Its text is used when prompt is omitted. |
| `product_id` | array of integer | yes | Products to try, 1-3 (see find_product / list_product). |
| `prompt` | string | no | The prompt text being tested. |
| `paragraph_length` | integer | no | Target paragraph length in characters, 50-500. |
| `generate_count_section` | integer | no | Basic template only: paragraphs, 0-10. |
| `generate_count_benefit` | integer | no | Basic template only: benefits, 0-10. |
| `generate_count_faq` | integer | no | Basic template only: questions and answers, 0-10. |
| `generate_seo_meta` | boolean | no | Generate SEO title/description. |
| `generate_entity_name` | boolean | no | Generate an improved product name. |
| `generate_image_variation` | boolean | no | Let AI create missing photos (paid per photo). |
| `use_ai_profile` | boolean | no | Include the eshop AI profile (tone of voice) in the prompt. |
| `use_serp_context` | boolean | no | Fetch Google/SerpAPI context (paid). Templates only — a basic template runs no research. |
| `use_web_research` | boolean | no | Fetch web research by EAN (paid). Templates only. |
| `search_web` | array of string | no | Restrict web research to these domains, e.g. ["https://yoggies.cz"]. Max 20. Templates only. |
| `search_model` | array of string | no | Fact-finders to run: openai / gemini / perplexity / anthropic (each paid). Templates only. One of: `openai`, `gemini`, `perplexity`, `anthropic`. |
| `expected_credit` | integer | no | The price the user confirmed. Second call only. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted price. Second call only. |

**Returns:** `preview` — one entry per `product_id` (`{product_id, token, status}` or an error entry when the product does not belong to the eshop), `poll_with: "get_preview_status"`, `expires_in_minutes: 30`.

**Careful:** refused with an error when the AI-content-generation kill switch is off — this can look like a bug rather than a maintenance window.

### `set_widget_prompt`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Replaces ALL per-widget instructions of a custom prompt; each targets one widget of its design (list_design_widget) and overrides the general prompt for it. An empty array clears them.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `prompt_id` | integer | yes | Prompt id (see list_prompt). The prompt must have a design_id. |
| `widget_prompt` | array of object | yes | The complete new set of per-widget instructions (replaces all existing ones). Empty array clears them. Max items 30. |
| `widget_prompt[].design_widget_id` | integer | yes | Widget of the prompt's design (see list_design_widget). |
| `widget_prompt[].prompt` | string | yes | Instruction for this widget's content. Max length 50000. |

**Returns:** `action: "replaced"`, `prompt_id`, `widget_prompt_count`.

**Careful:** replace-all semantics — every call defines the *complete* set; anything omitted is deleted, not left alone. Fails if the prompt has no `design_id` (set one first via `update_prompt`).

### `start_generation`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Available:** PROFI plan or higher (AI generation)  
**Rate limit:** counted

"Generate descriptions for these products / categories" — the batch the generation pages start: up to 500 products or categories with a prompt (its design, research and languages). Fast and cheaper; for a few products that deserve research, an FAQ and parameters with icons use start_agent_task. PAID, two steps: called with product_id, or entity_type=category and category_id (prompt_id, or the one used last), it only prepares the batch and returns generation_id, the price and the entities that lack data; call again with generation_id and cost_confirmed=true to start. Categories need a prompt whose template is for categories (list_prompt: for_category=true) and are written from their products; products never use such a prompt. Follow it with get_generation_status.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | no | What to generate; product when omitted. First call only. One of: `product`, `category`. |
| `product_id` | array of integer | no | Products to generate, 500 at most. First call only. |
| `category_id` | array of integer | no | Categories to generate (see list_category), 500 at most; with entity_type=category. First call only. |
| `prompt_id` | integer | no | Prompt to generate with (see list_prompt); for categories one with for_category=true. Omit for the one used last. First call only. |
| `generation_id` | string | no | The generation_id the first call returned. Second call only. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted price. Second call only. |

### `update_prompt`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Changes a custom prompt — only the fields sent. Changing design_id clears its per-widget instructions (set them again with set_widget_prompt). Check get_prompt_history and try the change with preview_generation before it becomes permanent.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `prompt_id` | integer | yes | Prompt id (see list_prompt). |
| `name` | string | no | New profile name. Min length 5. Max length 255. |
| `prompt` | string | no | New generation prompt text (full replacement of the text). Min length 20. Max length 1000000. |
| `design_id` | integer | no | New design id (see list_design), or null to unlink. Clears existing per-widget instructions when changed. |
| `icon_category_id` | integer | no | New icon category id, or null to unlink. |
| `photo_mode` | string | no | How a template generation fills its photos: legacy = gallery order (default); smart = AI picks photos by what they show, places the hotspot points and draws only missing photos; profi = smart, plus AI sharpens and fits the eshop photos to each block and checks the drawn icons (each photo the model draws costs an image credit). Only for a prompt with a design. One of: `legacy`, `smart`, `profi`. |
| `fact_coverage` | boolean | no | Keep the facts of the original description: AI checks the new text against the product's original description (from 1 000 characters) and writes lost facts back into the blocks. About half a minute longer per product. Only for a prompt with a design. |

**Returns:** `action: "updated"`, `widget_prompt_cleared` (bool), plus the prompt detail (`id`, `name`, `prompt`, `design_id`, `icon_category_id`, `icon_mode`, `widget_prompt`, `created_at`, `updated_at`).

**Careful:** changing `design_id` clears all existing per-widget instructions of the prompt (they referenced widgets of the previous design) — `widget_prompt_cleared` in the response confirms whether this happened. Errors if no field is sent to change.

## Diagnostics

Read-only look inside the merchant's own generation runs, the web research behind them and the export that follows — the three places a description can go wrong.

Tools: [`get_export_status`](#get_export_status), [`get_generation`](#get_generation), [`get_product_research`](#get_product_research), [`list_generation`](#list_generation)

### `get_export_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Why is the new description not on my eshop?" — recent exports of Pobo content to the eshop platform: progress, completion, failure. A failed export is never retried and its products stayed unexported — the usual reason approved content is missing on the eshop. The other direction (new products missing in Pobo) is get_import_status.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

**Returns:** `platform`, `accepts_new_export` (bool), and `export` (array of `{id, created_at, finished_at, is_complete, is_running, export_total, export_complete, failed_at}`).

### `get_generation`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Why is something missing in the generated description?" — one AI generation run: its settings, what web research found per provider (with coverage of which facts were available at all) and the model answer. research_status.reason="no_ean" means research never ran for lack of an EAN; is_complete=false without skip_reason means the run is stuck, not failed. Find the run with list_generation.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `job_id` | integer | yes | Generation run id (see list_generation). |
| `include_response` | boolean | no | Include the raw model answer. Default true. Default `true`. |
| `search` | string | no | Case-insensitive keyword. When given, the model answer comes back as match windows (up to 5, 600 chars of context each) instead of full text. |

**Returns:** `generation` — settings summary, per-provider research with a coverage summary, and the model answer (unless `include_response=false` or narrowed by `search`).

**Careful:** the assembled prompt sent to the model is never returned. `research_status.reason="no_ean"` means research never ran for lack of an EAN; `is_complete=false` without a `skip_reason` means the run is stuck, not failed.

### `get_product_research`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Why did the description leave out a fact the manufacturer states?" — what web research found for one product in its recent generation runs, per provider, with coverage of which facts turned up and which sources were cited. A fact no provider returned was never available to the writing model.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_id` | integer | yes | Product id (see find_product / list_product). |
| `limit` | integer | no | How many recent generation runs WITH research to return, 1-10. Default 3. Default `3`. |

**Returns:** `research` — array of `{job_id, created_at, search_web, search_model, provider: [{provider, mode, confidence, response}], coverage}`.

### `list_generation`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which generations ran and which are stuck?" — the eshop's AI generation runs, newest first, with settings summary, completion state and research row count. product_id = every run of one product, only_incomplete = what is stuck. Detail of one run: get_generation.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_id` | integer | no | Only runs of this product — the usual filter when checking one description. |
| `query` | string | no | Product name fragment the run was started with. |
| `date_from` | string | no | Only runs created at or after this date (Y-m-d or ISO 8601). |
| `date_to` | string | no | Only runs created at or before this date (Y-m-d or ISO 8601). |
| `only_incomplete` | boolean | no | Only runs that never completed — stuck or failed generations. Default `false`. |
| `limit` | integer | no | Rows per page, 1-100. Default 20. Default `20`. |
| `page` | integer | no | Page number, from 1. Default 1. Default `1`. |

**Returns:** `generation` — array of run summaries (`id`, `product_id`, `name`, timestamps, `is_complete`, `is_running`, `skip_reason`, `design_id`/`design_name`, `prompt_text_id`/`prompt_text_name`, research settings, `research_row_count`) — plus `meta` (`total`, `page`, `per_page`, `last_page`).

## Description automation

Tools: [`create_automation_rule`](#create_automation_rule), [`get_automation_rule`](#get_automation_rule), [`list_automation_rule`](#list_automation_rule), [`list_automation_run`](#list_automation_run), [`preview_automation_rule`](#preview_automation_rule), [`update_automation_rule`](#update_automation_rule)

### `create_automation_rule`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no (the rule itself is free to create; the nightly runs it schedules later spend credits per product generated)  
**Rate limit:** counted

"Write descriptions of new products automatically every night" — creates a scheduled automation that generates AI descriptions for products still without one. Created paused by default: check preview_automation_rule, then unpause with update_automation_rule — never without saying what it costs per night.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `name` | string | yes | Name shown in the merchant admin, e.g. "Fill in new arrivals". Max length 120. |
| `prompt_id` | integer | yes | Prompt profile that defines HOW to generate (design, languages, SEO). See list_prompt. |
| `schedule_cron` | string | yes | When to run, as a cron expression in the eshop timezone, e.g. "0 2 * * *" for 02:00 nightly. At most once per day. |
| `batch_size` | integer | yes | How many products at most per run. Each product costs credits, so this is the nightly ceiling. |
| `description` | string | no | Optional note for the merchant. Max length 2000. |
| `category_id` | array of integer | no | Optional: limit to these categories (see list_category). Omit for the whole catalogue. |
| `brand_id` | array of integer | no | Optional: limit to these brands. Omit for the whole catalogue. |
| `is_paused` | boolean | no | Defaults to true. Pass false to start the automation right away. |

**Returns:** `action: "created"` plus the full rule detail — `id`, `name`, `description`, `is_paused`, `priority`, `prompt` (`{id, name}`), `category` list, `brand` list, `scope` (`"whole catalogue"` or `"limited to the listed categories and brands"`), `batch_size`, `schedule_cron`, `schedule_timezone`, `last_run_at`, `next_run_at`, `consecutive_failure`, `created_at`.

**Careful:** the rule is created **paused by default** — `is_paused` defaults to `true` regardless of whether the caller set a schedule, so it will not run and spend credits until it is explicitly unpaused (via `update_automation_rule` with `is_paused: false`, or `is_paused: false` at creation). This is deliberate: an agent that just invented a cron expression is not also meant to start it running unattended. Once unpaused, the rule runs nightly on its own and spends credits per product generated, with no further confirmation per run.

### `get_automation_rule`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

One automation in full, with its last runs (products queued, skipped for missing credits, skipped as recently generated).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `rule_id` | integer | yes | Automation id (see list_automation_rule). |

**Returns:** the full rule detail (same shape as `create_automation_rule`'s response) plus `recent_run` (up to 5 most recent runs, each with `id`, `rule_id`, `rule_name`, `triggered_by`, `status`, `started_at`, `finished_at`, `candidate_count`, `job_created`, `skipped_no_credit`, `skipped_recently_generated`, `error_message`) and `auto_pause_warning` (true when `consecutive_failure > 0` — three failed nights in a row auto-pause the rule).

### `list_automation_rule`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which automations do I have?" — the eshop's scheduled description automations: schedule, prompt, scope, batch size, paused or not, last and next run.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `is_paused` | boolean | no | Optional filter: true returns only paused automations, false only active ones. |

**Returns:** `rule`, an array of rule details (same shape as `create_automation_rule`'s response, one per rule).

### `list_automation_run`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What did the automation do last night / why did it fail?" — history of automation runs: when, products queued, skipped for missing credits or as recently generated, and why a run failed (three failed nights in a row pause a rule).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `rule_id` | integer | no | Optional: only runs of this automation (see list_automation_rule). |
| `limit` | integer | no | How many runs to return, newest first. Defaults to 20, max 100. |

**Returns:** `run` (array of run details — same shape as `get_automation_rule`'s `recent_run` entries) and `total` (count across the filtered set).

### `preview_automation_rule`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What would the automation do tonight?" — dry run: how many products it would pick, how many it would queue given its batch size, a sample, and whether the credits cover it. Spends nothing.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `rule_id` | integer | yes | Automation id (see list_automation_rule). |

**Returns:** `rule_id`, `rule_name`, `is_paused`, `next_run_at`, `candidate_count` (total matching products), `would_queue_next_run` (min of candidate_count and batch_size), `batch_size`, `credit_available`, `selector` (a fixed description: "Only products without a description, not generated in the last 30 days."), `sample_product` (up to 5 `{id, name}`).

### `update_automation_rule`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Pauses or resumes an automation (is_paused) or changes its schedule, prompt, batch size or scope; only the fields sent change. Say what a resumed or widened rule will cost per night first.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `rule_id` | integer | yes | Automation id (see list_automation_rule). |
| `is_paused` | boolean | no | true pauses the automation (it stops running at night), false resumes it. |
| `name` | string | no | New name. Max length 120. |
| `description` | string | no | New note. Max length 2000. |
| `prompt_id` | integer | no | Different prompt profile (see list_prompt). |
| `schedule_cron` | string | no | New cron expression in the eshop timezone, at most once per day. |
| `batch_size` | integer | no | New per-run product ceiling. |
| `category_id` | array of integer | no | REPLACES the category scope. An empty array widens the automation to the whole catalogue. |
| `brand_id` | array of integer | no | REPLACES the brand scope. An empty array widens the automation to the whole catalogue. |

**Returns:** `action: "updated"` plus the full rule detail (same shape as `create_automation_rule`).

**Careful:** `category_id` and `brand_id`, when sent, REPLACE the whole scope rather than adding to it — sending one category drops any others previously set. Setting `is_paused: false` also silently resets `consecutive_failure` to 0, so resuming a rule that had auto-paused after 3 failed nights clears its failure streak and gives it a fresh 3-strike count.

## Pobo agent

Pobo's own agent: a background task that optimises descriptions and leaves them as drafts (`start_agent_task`, paid, PROFI and up), the candidates it picks from, and the overview of every change an agent made through MCP — including yours — with its reason.

Tools: [`get_agent_task`](#get_agent_task), [`get_optimize_candidate`](#get_optimize_candidate), [`list_agent_change`](#list_agent_change), [`start_agent_task`](#start_agent_task)

### `get_agent_task`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"How is the agent's task going?" — a task of Pobo's agent (start_agent_task): status (pending, running, completed, failed, stopped), products done and failed, and the agent's summary once it ended. wait (up to 60 s) holds the answer until the task ends — one call instead of asking again and again.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `task_id` | integer | yes | The task_id start_agent_task returned. |
| `wait` | integer | no | Seconds to wait for the task to end before answering, 0-60. |

### `get_optimize_candidate`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** counted

"Which products should I work on first? / Which products without a description get the most visits?" — products worth working on, from the eshop's traffic in the last 30 days. missing = visited product pages without a Pobo description, most visited first: write them. weak = Pobo descriptions people see but rarely add to cart from, worst first: improve them; where can_ab_test is true (300+ page views) prefer an A/B test with the new version as variant B. ai_missed = products an AI assistant did not mention in the last AI visibility measurement: read list_ai_visibility_result and add exactly the facts or FAQ it asks about. catalog = only when the eshop has no measured traffic (a new eshop): visible products without a Pobo description, with a photo and newest first. ranked = all of them ordered by expected add-to-carts per credit, with action (write or ab_test), expected_cart, credit_cost, cart_per_credit and the assumption behind the estimate — work from it when spending a budget; its top items carry "research" (facts and a summary from web research of the last 90 days) — write from it and do not call research_product for that product. Products an agent changed in the last 30 days, products in an open A/B test and products an agent researched in the last 30 days and skipped for lack of facts, and products whose web research found fewer than 4 facts are left out. Small eshops often have no weak list. Use it, not list_product, for any question about traffic or what to improve. Numbers are add to cart, not orders; never claim a product sells badly.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `limit` | integer | no | Candidates per list, 20 at most. Default 10. |

### `list_agent_change`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"What did the agent change?" — changes agents made on this eshop, newest first: tool, entities, the reason given, and whether revert_content can undo it by batch_id. entity_changed_today against daily_entity_cap says how many more products, categories or articles may change today (products changed today do not count again, reverts never count). Read it before a run so you do not redo or contradict an earlier change, and give a reason with every write — the merchant reads this list instead of approving. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `limit` | integer | no | How many, 100 at most. Default 30. |
| `page` | integer | no | Page, from 1. |

### `start_agent_task`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Available:** PROFI plan or higher (AI generation)  
**Rate limit:** counted

"Write great descriptions for these few products" — Pobo's agent writes them in the background with web research, an FAQ, parameters with icons and a fact check. product_id (up to 20), or optimize=true to let the agent pick products from the eshop's traffic within its monthly budget. For many products or products with good data start_generation is faster and cheaper. PAID, two steps: without cost_confirmed it only quotes; call again with cost_confirmed=true. The descriptions end as drafts and each is charged when the merchant approves it. Follow it with get_agent_task. Products only, not categories.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product_id` | array of integer | no | Products to write, 20 at most. Or optimize. |
| `optimize` | boolean | no | Let the agent pick the products from the traffic instead of product_id. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the price. |

## A/B tests

A/B tests of descriptions measured by add-to-cart. A test is created as a draft, the variant is edited, and only `start_ab_test` changes anything for shoppers.

Tools: [`compose_ab_test_variant`](#compose_ab_test_variant), [`create_ab_test`](#create_ab_test), [`finish_ab_test`](#finish_ab_test), [`get_ab_test_result`](#get_ab_test_result), [`list_ab_test`](#list_ab_test), [`start_ab_test`](#start_ab_test)

### `compose_ab_test_variant`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the review of variant B costs 2 credits  
**Rate limit:** counted

Writes variant B of a pobo_vs_variant A/B test (create_ab_test): the whole description in one call, the widget shape of compose_content. It replaces the previous variant B; the live description (variant A) stays. The test must be a draft or paused. Variant B is then reviewed like a description before publishing — rules, facts and spelling (2 credits) — passed=false lists what to fix; start_ab_test starts only a variant that passed.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `ab_test_id` | integer | yes | The A/B test (see list_ab_test). |
| `widget` | array of object | yes | The blocks of variant B in order, 30 at most, each {"widget_id": int, "content": {role: value}, "image": [url]}. |

### `create_ab_test`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"Test whether the new description sells better" — prepares an A/B test as a DRAFT on up to 100 pages. original_vs_pobo compares the platform's original description (variant b) with the live Pobo content (variant a) on many pages; pobo_vs_variant takes exactly one page and copies its live content as variant b to be edited (compose_ab_test_variant or the admin). Nothing changes for shoppers until it starts. Pages not on the eshop come back in not_found, pages already in a running test in busy (the draft is created, but cannot start until that test ends). Measured metric: add to cart. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `name` | string | yes | Test name the merchant sees ("Original vs. Pobo on car mats"). Max length 255. |
| `type` | string | yes | original_vs_pobo or pobo_vs_variant. One of: `original_vs_pobo`, `pobo_vs_variant`. |
| `entity_type` | string | yes | What the pages are. One of: `product`, `category`, `blog`. |
| `entity_id` | array of integer | yes | Pobo ids of the pages, 100 at most; exactly one for pobo_vs_variant. |

### `finish_ab_test`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

"End the test and keep the winner" — closes a running A/B test if its result allows it. Variant B goes live only when it won surely: enough data, no broken visitor split, add-to-cart better with p < 0.01. After 28 days without such a result the test ends and the live description stays; before that an unsure test keeps running — call again later. The answer names the decision and why.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `ab_test_id` | integer | yes | The A/B test (see list_ab_test). |

### `get_ab_test_result`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** counted

"Which variant wins?" — results of one A/B test per variant (a = live Pobo content, b = the other): exposure, page views, cart_session and cart_rate_percent — the metric is ADD TO CART, not orders, report it as such —, time on page, FAQ opens, a daily series, comparison.p_value with significant (p < 0.05), an SRM check of the 50/50 split and meta.sufficient_data (at least 100 exposures per variant and 20 cart sessions). Never call a winner while sufficient_data is false or srm.warning is true. Counts against the tariff limit.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `ab_test_id` | integer | yes | A/B test id (see list_ab_test). |

### `list_ab_test`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Which A/B tests do I have?" — the eshop's tests, newest first: what is compared (original_vs_pobo or pobo_vs_variant), on which entities, status (draft, running, paused, completed, applied), schedule, declared winner. get_ab_test_result reads how one performs. Free.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `status` | string | no | Only tests in this status. One of: `draft`, `running`, `paused`, `completed`, `applied`. |
| `limit` | integer | no | How many, 50 at most. Default 20. |

### `start_ab_test`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Rate limit:** counted

Starts an A/B test: half of the visitors see variant B from now on. Refused unless variant B passed its review (compose_ab_test_variant), the product had at least 300 page views in the last 30 days (less never reaches a result within 28 days — judge such a product from its heatmap in get_product_analytics), and no other test runs on it. finish_ab_test closes it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `ab_test_id` | integer | yes | The A/B test (see list_ab_test). |

## AI visibility

How often AI assistants (ChatGPT and others) recommend the eshop for the questions the merchant measures.

Tools: [`list_ai_visibility_result`](#list_ai_visibility_result)

### `list_ai_visibility_result`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Rate limit:** not counted

"Does ChatGPT recommend my eshop?" — the last AI visibility measurement: for each question asked to AI assistants whether the answer mentioned or cited the eshop, the product the question is about, and missing_signal — what the eshop lacks (description_missing, faq_missing, content_topic, ai_profile_empty, bot_blocked). Answers vary between runs: one measurement proves nothing, compare at least two before claiming a change worked. mentioned=false with a content signal is worth improving: add exactly the facts or FAQ the question asks about.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

## Pobo storefront — orders, stock, prices, branches and add-ons

Tools for eshops on Pobo's own platform (`pobo`) — orders, reservations, price lists, warehouses and stock documents, branches, the storefront banners and add-ons such as the virtual try-on. Every tool marked **Pobo storefront only** refuses an eshop on any other platform before it runs.

Orders are read-only on purpose: changing an order sends the customer an e-mail or SMS, so it stays in the administration. Stock documents are only created and edited as drafts — the merchant confirms them.

Tools: [`add_branch_image`](#add_branch_image), [`create_branch`](#create_branch), [`create_warehouse_document`](#create_warehouse_document), [`fill_try_on_product`](#fill_try_on_product), [`generate_category_banner`](#generate_category_banner), [`generate_product_banner`](#generate_product_banner), [`get_branch`](#get_branch), [`get_erp_status`](#get_erp_status), [`get_order`](#get_order), [`get_sales_summary`](#get_sales_summary), [`get_try_on`](#get_try_on), [`list_branch`](#list_branch), [`list_branch_employee`](#list_branch_employee), [`list_branch_service`](#list_branch_service), [`list_order`](#list_order), [`list_price_list`](#list_price_list), [`list_reservation`](#list_reservation), [`list_stock`](#list_stock), [`list_stock_movement`](#list_stock_movement), [`list_try_on_product`](#list_try_on_product), [`list_warehouse_document`](#list_warehouse_document), [`save_branch_employee`](#save_branch_employee), [`save_branch_notice`](#save_branch_notice), [`save_branch_service`](#save_branch_service), [`set_branch_banner`](#set_branch_banner), [`set_branch_employee_photo`](#set_branch_employee_photo), [`set_storefront_image`](#set_storefront_image), [`update_branch`](#update_branch), [`update_branch_image`](#update_branch_image), [`update_branch_setting`](#update_branch_setting), [`update_try_on_product`](#update_try_on_product), [`update_try_on_setting`](#update_try_on_setting), [`update_warehouse_document`](#update_warehouse_document)

### `add_branch_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Add photos of the branch" — adds photos to the gallery of a branch page of a pobo eshop from public https addresses (JPEG, PNG or WebP up to 10 MB), up to 10 per call and 20 per branch, behind the photos already there. alt: language (default, cs, sk, …) => description. Order, descriptions and removal: update_branch_image. Each photo reported as added or failed. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `branch_id` | integer | yes | see list_branch |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `image` | array of object | yes | Max items 10. |
| `image[].image_url` | string | yes | public https address |
| `image[].alt` | object | no |  |
| `image[].alt.default` | string | no |  |
| `image[].alt.cs` | string | no |  |
| `image[].alt.sk` | string | no |  |
| `image[].alt.en` | string | no |  |
| `image[].alt.de` | string | no |  |
| `image[].alt.pl` | string | no |  |
| `image[].alt.hu` | string | no |  |
| `image[].alt.ro` | string | no |  |
| `image[].alt.vi` | string | no |  |
| `image[].alt.it` | string | no |  |
| `image[].alt.es` | string | no |  |
| `image[].alt.bg` | string | no |  |
| `image[].alt.hr` | string | no |  |
| `image[].alt.fr` | string | no |  |
| `image[].alt.uk` | string | no |  |
| `image[].alt.sl` | string | no |  |
| `image[].alt.pt` | string | no |  |
| `image[].alt.nl` | string | no |  |
| `image[].alt.sr` | string | no |  |

### `create_branch`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Add a new branch / shop" on a pobo eshop: country and translation.default.name are required, the rest optional. The page address comes from the name (slug overrides it). Photos: add_branch_image; banner: set_branch_banner; notices: save_branch_notice. warehouse_id: the warehouse whose stock the branch shows and reserves from (one warehouse = one branch; see list_stock for codes, ids in get_branch of other branches). slot_minute (10–240) and slot_capacity (1–20): length and capacity of a reservation slot. position orders the branches. country: ISO code (CZ, SK, …). opening_hour: ISO weekday "1"–"7" => list of {from, to} in HH:MM, in order, without overlaps; an empty list = closed that day. closed: the full list of extra closed days (YYYY-MM-DD). translation: language (default = main language, then a content language code: cs, sk, en, ro, …) => name, description (HTML: paragraphs, lists, h2/h3, links), seo_title, seo_description, parking, accessibility, transport. service_id: the full list of services (list_branch_service). slug: language => the page address part; the old address keeps redirecting. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `is_active` | boolean | no |  |
| `warehouse_id` | integer | no |  |
| `street` | string | no |  |
| `city` | string | no |  |
| `zip` | string | no |  |
| `country` | string | no | One of: `AT`, `BE`, `BG`, `CY`, `CZ`, `DE`, `DK`, `EE`, `ES`, `FI`, `FR`, `GR`, `HR`, `HU`, `IE`, `IT`, `LT`, `LU`, `LV`, `MT`, `NL`, `PL`, `PT`, `RO`, `SE`, `SI`, `SK`. |
| `phone` | string | no |  |
| `email` | string | no |  |
| `lat` | number | no |  |
| `lng` | number | no |  |
| `opening_hour` | object | no | ISO weekday "1"–"7" => [{"from": "09:00", "to": "17:00"}]; empty list = closed that day |
| `slot_minute` | integer | no |  |
| `slot_capacity` | integer | no |  |
| `position` | integer | no |  |
| `closed` | array of string | no | the full list of extra closed days, YYYY-MM-DD |
| `translation` | object | no |  |
| `translation.default` | object | no |  |
| `translation.default.name` | string | no | Max length 255. |
| `translation.default.description` | string | no | HTML |
| `translation.default.seo_title` | string | no | Max length 255. |
| `translation.default.seo_description` | string | no | Max length 1000. |
| `translation.default.parking` | string | no |  |
| `translation.default.accessibility` | string | no |  |
| `translation.default.transport` | string | no |  |
| `translation.cs` | object | no |  |
| `translation.cs.name` | string | no | Max length 255. |
| `translation.cs.description` | string | no | HTML |
| `translation.cs.seo_title` | string | no | Max length 255. |
| `translation.cs.seo_description` | string | no | Max length 1000. |
| `translation.cs.parking` | string | no |  |
| `translation.cs.accessibility` | string | no |  |
| `translation.cs.transport` | string | no |  |
| `translation.sk` | object | no |  |
| `translation.sk.name` | string | no | Max length 255. |
| `translation.sk.description` | string | no | HTML |
| `translation.sk.seo_title` | string | no | Max length 255. |
| `translation.sk.seo_description` | string | no | Max length 1000. |
| `translation.sk.parking` | string | no |  |
| `translation.sk.accessibility` | string | no |  |
| `translation.sk.transport` | string | no |  |
| `translation.en` | object | no |  |
| `translation.en.name` | string | no | Max length 255. |
| `translation.en.description` | string | no | HTML |
| `translation.en.seo_title` | string | no | Max length 255. |
| `translation.en.seo_description` | string | no | Max length 1000. |
| `translation.en.parking` | string | no |  |
| `translation.en.accessibility` | string | no |  |
| `translation.en.transport` | string | no |  |
| `translation.de` | object | no |  |
| `translation.de.name` | string | no | Max length 255. |
| `translation.de.description` | string | no | HTML |
| `translation.de.seo_title` | string | no | Max length 255. |
| `translation.de.seo_description` | string | no | Max length 1000. |
| `translation.de.parking` | string | no |  |
| `translation.de.accessibility` | string | no |  |
| `translation.de.transport` | string | no |  |
| `translation.pl` | object | no |  |
| `translation.pl.name` | string | no | Max length 255. |
| `translation.pl.description` | string | no | HTML |
| `translation.pl.seo_title` | string | no | Max length 255. |
| `translation.pl.seo_description` | string | no | Max length 1000. |
| `translation.pl.parking` | string | no |  |
| `translation.pl.accessibility` | string | no |  |
| `translation.pl.transport` | string | no |  |
| `translation.hu` | object | no |  |
| `translation.hu.name` | string | no | Max length 255. |
| `translation.hu.description` | string | no | HTML |
| `translation.hu.seo_title` | string | no | Max length 255. |
| `translation.hu.seo_description` | string | no | Max length 1000. |
| `translation.hu.parking` | string | no |  |
| `translation.hu.accessibility` | string | no |  |
| `translation.hu.transport` | string | no |  |
| `translation.ro` | object | no |  |
| `translation.ro.name` | string | no | Max length 255. |
| `translation.ro.description` | string | no | HTML |
| `translation.ro.seo_title` | string | no | Max length 255. |
| `translation.ro.seo_description` | string | no | Max length 1000. |
| `translation.ro.parking` | string | no |  |
| `translation.ro.accessibility` | string | no |  |
| `translation.ro.transport` | string | no |  |
| `translation.vi` | object | no |  |
| `translation.vi.name` | string | no | Max length 255. |
| `translation.vi.description` | string | no | HTML |
| `translation.vi.seo_title` | string | no | Max length 255. |
| `translation.vi.seo_description` | string | no | Max length 1000. |
| `translation.vi.parking` | string | no |  |
| `translation.vi.accessibility` | string | no |  |
| `translation.vi.transport` | string | no |  |
| `translation.it` | object | no |  |
| `translation.it.name` | string | no | Max length 255. |
| `translation.it.description` | string | no | HTML |
| `translation.it.seo_title` | string | no | Max length 255. |
| `translation.it.seo_description` | string | no | Max length 1000. |
| `translation.it.parking` | string | no |  |
| `translation.it.accessibility` | string | no |  |
| `translation.it.transport` | string | no |  |
| `translation.es` | object | no |  |
| `translation.es.name` | string | no | Max length 255. |
| `translation.es.description` | string | no | HTML |
| `translation.es.seo_title` | string | no | Max length 255. |
| `translation.es.seo_description` | string | no | Max length 1000. |
| `translation.es.parking` | string | no |  |
| `translation.es.accessibility` | string | no |  |
| `translation.es.transport` | string | no |  |
| `translation.bg` | object | no |  |
| `translation.bg.name` | string | no | Max length 255. |
| `translation.bg.description` | string | no | HTML |
| `translation.bg.seo_title` | string | no | Max length 255. |
| `translation.bg.seo_description` | string | no | Max length 1000. |
| `translation.bg.parking` | string | no |  |
| `translation.bg.accessibility` | string | no |  |
| `translation.bg.transport` | string | no |  |
| `translation.hr` | object | no |  |
| `translation.hr.name` | string | no | Max length 255. |
| `translation.hr.description` | string | no | HTML |
| `translation.hr.seo_title` | string | no | Max length 255. |
| `translation.hr.seo_description` | string | no | Max length 1000. |
| `translation.hr.parking` | string | no |  |
| `translation.hr.accessibility` | string | no |  |
| `translation.hr.transport` | string | no |  |
| `translation.fr` | object | no |  |
| `translation.fr.name` | string | no | Max length 255. |
| `translation.fr.description` | string | no | HTML |
| `translation.fr.seo_title` | string | no | Max length 255. |
| `translation.fr.seo_description` | string | no | Max length 1000. |
| `translation.fr.parking` | string | no |  |
| `translation.fr.accessibility` | string | no |  |
| `translation.fr.transport` | string | no |  |
| `translation.uk` | object | no |  |
| `translation.uk.name` | string | no | Max length 255. |
| `translation.uk.description` | string | no | HTML |
| `translation.uk.seo_title` | string | no | Max length 255. |
| `translation.uk.seo_description` | string | no | Max length 1000. |
| `translation.uk.parking` | string | no |  |
| `translation.uk.accessibility` | string | no |  |
| `translation.uk.transport` | string | no |  |
| `translation.sl` | object | no |  |
| `translation.sl.name` | string | no | Max length 255. |
| `translation.sl.description` | string | no | HTML |
| `translation.sl.seo_title` | string | no | Max length 255. |
| `translation.sl.seo_description` | string | no | Max length 1000. |
| `translation.sl.parking` | string | no |  |
| `translation.sl.accessibility` | string | no |  |
| `translation.sl.transport` | string | no |  |
| `translation.pt` | object | no |  |
| `translation.pt.name` | string | no | Max length 255. |
| `translation.pt.description` | string | no | HTML |
| `translation.pt.seo_title` | string | no | Max length 255. |
| `translation.pt.seo_description` | string | no | Max length 1000. |
| `translation.pt.parking` | string | no |  |
| `translation.pt.accessibility` | string | no |  |
| `translation.pt.transport` | string | no |  |
| `translation.nl` | object | no |  |
| `translation.nl.name` | string | no | Max length 255. |
| `translation.nl.description` | string | no | HTML |
| `translation.nl.seo_title` | string | no | Max length 255. |
| `translation.nl.seo_description` | string | no | Max length 1000. |
| `translation.nl.parking` | string | no |  |
| `translation.nl.accessibility` | string | no |  |
| `translation.nl.transport` | string | no |  |
| `translation.sr` | object | no |  |
| `translation.sr.name` | string | no | Max length 255. |
| `translation.sr.description` | string | no | HTML |
| `translation.sr.seo_title` | string | no | Max length 255. |
| `translation.sr.seo_description` | string | no | Max length 1000. |
| `translation.sr.parking` | string | no |  |
| `translation.sr.accessibility` | string | no |  |
| `translation.sr.transport` | string | no |  |
| `service_id` | array of integer | no | the full list of services |
| `slug` | object | no | language => slug of the branch page |
| `slug.default` | string | no |  |
| `slug.cs` | string | no |  |
| `slug.sk` | string | no |  |
| `slug.en` | string | no |  |
| `slug.de` | string | no |  |
| `slug.pl` | string | no |  |
| `slug.hu` | string | no |  |
| `slug.ro` | string | no |  |
| `slug.vi` | string | no |  |
| `slug.it` | string | no |  |
| `slug.es` | string | no |  |
| `slug.bg` | string | no |  |
| `slug.hr` | string | no |  |
| `slug.fr` | string | no |  |
| `slug.uk` | string | no |  |
| `slug.sl` | string | no |  |
| `slug.pt` | string | no |  |
| `slug.nl` | string | no |  |
| `slug.sr` | string | no |  |

### `create_warehouse_document`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"A delivery arrived / write down the stocktake" — creates DRAFT warehouse documents of a pobo eshop, up to 20 per call: receipt (pieces that arrived, e.g. a delivery note), issue (pieces leaving without an order; note = why), transfer (warehouse_code → target_warehouse_code) or stocktake (counted pieces; products left out stay). Lines name variants by code or EAN with whole pieces; the same variant twice is added up. A draft changes NO stock — the merchant reviews and confirms it in Storefront → Stock → Documents. Unknown codes come back in not_found; a document with no known code is not created. Check list_warehouse_document for a draft of the same delivery first — update_warehouse_document changes it instead of making a second one.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `document` | array of object | yes | Max items 20. |
| `document[].kind` | string | yes | One of: `receipt`, `issue`, `transfer`, `stocktake`. |
| `document[].warehouse_code` | string | no | the default warehouse when left out |
| `document[].target_warehouse_code` | string | no | transfer only |
| `document[].supplier` | string | no | receipt only |
| `document[].note` | string | no | the reason of an issue; anything else, e.g. the delivery note number |
| `document[].line` | array of object | yes | Max items 500. |
| `document[].line[].code` | string | yes | variant code or EAN |
| `document[].line[].quantity` | integer | yes | whole pieces; at least 1, a stocktake may count 0 |

### `fill_try_on_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Offer the try-on for all my glasses" — adds every variant of the products in the try-on categories (update_try_on_setting category_id), each with the first photo of its product. It only adds: a variant already listed — switched off or with a photo picked by hand — stays, nothing is removed. Products without a downloaded photo are skipped and counted. Run it again after new glasses arrive; check the photos with list_try_on_product. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |

### `generate_category_banner`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Make a banner for this category" — generates the picture behind the category heading on the pobo storefront and sets it (replacing the current one), up to 10 categories per call: a wide photo with products on the right and a calm left half for the heading, no people or text. scene describes it in a sentence or two; without it the photo shows products of the category by its name. PAID, two steps: the first call only quotes; repeat it with cost_confirmed=true. Made in the background within about a minute; only stored banners are billed. Your own picture: set_storefront_image.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `category` | array of object | yes | Max items 10. |
| `category[].category_id` | integer | yes | see list_category |
| `category[].scene` | string | no | what the photo shows, in English or the eshop language Max length 500. |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost. |

### `generate_product_banner`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** yes — the first call only quotes the price; nothing is spent until it is repeated with `cost_confirmed=true`  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Make a hero banner for this product page" on the pobo storefront and set it (replacing the current one), up to 10 products per call. Drawn from one of the product's own photos, so it shows that product — by default in real use (worn or held when it fits), wide, product on the right, calm left half for the heading, no text. scene describes the photo; image_url picks the source (an address from get_product; a clean front photo works best — enhance_product_image makes one), otherwise the main photo. An image model redraws the product: check the banner before customers see it. PAID, two steps: the first call only quotes; repeat it with cost_confirmed=true. Made within about a minute; only stored banners are billed. Your own picture: set_storefront_image.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `product` | array of object | yes | Max items 10. |
| `product[].product_id` | integer | yes |  |
| `product[].scene` | string | no | what the photo shows, in English or the eshop language Max length 500. |
| `product[].image_url` | string | no | source photo, one of the addresses get_product lists |
| `cost_confirmed` | boolean | no | Set true after the user confirmed the quoted cost. |

### `get_branch`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Show me this branch" — one branch of a pobo eshop: address and contact, opening_hour (ISO weekday 1–7 => HH:MM intervals, empty = closed), extra closed days, texts per language (name, description HTML, seo_title, seo_description, parking, accessibility, transport), services, banner, photos and notices with their id. Read it before update_branch.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `branch_id` | integer | yes | see list_branch |

### `get_erp_status`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Is my ERP connected / why was my change refused?" — the ERP connection of a pobo eshop (ABRA Flexi, Pohoda, Money S3 or the merchant's own): whether one is active, what the ERP owns — catalogue, price, action price, stock, invoices (only the ERP sync writes those, other tools are refused) —, the last sync runs with refused items, and orders waiting for or refused by the ERP. Read only.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `get_order`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Show me order 2026001" — one order of a pobo eshop by order_id or number: lines with prices, shipping (with pickup point), payment, billing and shipping address, coupon, customer note and the history of status, payment, e-mail and shipment changes. Read only.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `order_id` | integer | no | Pobo id of the order (list_order); or send number |
| `number` | string | no | The order number the customer sees |

### `get_sales_summary`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"How much did I sell?" — sales of a pobo eshop over a period (Prague days, both included; default the last 30, at most 366): orders and revenue with VAT per currency, the average order, orders per status and best-selling variants by pieces. Cancelled orders count in status, never in revenue. Amounts are decimal strings. Read only.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `date_from` | string | no | YYYY-MM-DD; default 30 days before date_to |
| `date_to` | string | no | YYYY-MM-DD; default today |
| `top` | integer | no | how many best-selling variants, default 10, at most 50 |

### `get_try_on`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"How is the glasses try-on doing?" — the try-on add-on of a pobo eshop (visitors take a selfie and AI puts the glasses on it): on or off, its setting (category_id, daily_limit, ip_hour_limit, cta = reserve|cart), how many variants can be tried on, and the last day days: try-ons seen, ones that failed, how many then reserved or added to cart, and the AI cost.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `day` | integer | no | Days of results, 30 when left out. Min 1. Max 90. |

### `list_branch`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Which branches do I have?" — branches of a pobo eshop in their order: id, name, address, active, warehouse, banner, photo and notice counts. Read one with get_branch before changing it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `list_branch_employee`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Who works at the branch?" — the branch team of a pobo eshop: id for save_branch_employee, name, position and introduction per language, photo, order, branches. The branch page shows only a person with is_public=true — switched on and with the consent to publish name and photo, which the merchant confirms in the admin.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `list_branch_service`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

The services branches of a pobo eshop can offer (id for update_branch service_id, code, icon, name per language, active) and schema_type — what the branch pages tell search engines the branches are (update_branch_setting).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `list_order`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Show me the orders" of a pobo eshop (its own storefront), newest first: number, date, status, payment, customer, pieces, total. Filters: status (new, processing, shipped, delivered, cancelled), payment_status (paid, unpaid), date_from/date_to (day placed), query (number, e-mail or name). Read only — changing an order e-mails the customer, the merchant does it in Storefront → Orders. One order: get_order.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `status` | array of string | no | One of: `new`, `processing`, `shipped`, `delivered`, `cancelled`. |
| `payment_status` | string | no | One of: `unpaid`, `paid`. |
| `date_from` | string | no | YYYY-MM-DD |
| `date_to` | string | no | YYYY-MM-DD |
| `query` | string | no | order number, e-mail or customer name contains |
| `limit` | integer | no | default 20, at most 50 |
| `offset` | integer | no |  |

### `list_price_list`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Which price lists do I have?" on a pobo eshop — code, currency, whether prices include VAT, which is the default, and the storefront languages that show each (a language without one shows the default). create_product and update_product_pricing write into the default unless told otherwise.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

### `list_reservation`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Who is coming to pick up a reservation?" — reservations of goods at the branches of a pobo eshop, soonest first: when, branch, who, which goods, state (pending, converted, cancelled, no_show). Filters: status, branch_id, date_from/date_to (day of the visit), query (code, name, e-mail or phone). Read only.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `status` | string | no | One of: `pending`, `converted`, `cancelled`, `no_show`. |
| `branch_id` | integer | no |  |
| `date_from` | string | no | YYYY-MM-DD |
| `date_to` | string | no | YYYY-MM-DD |
| `query` | string | no | code, name, e-mail or phone contains |
| `limit` | integer | no | default 20, at most 50 |
| `offset` | integer | no |  |

### `list_stock`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"How many pieces do I have / what is out of stock?" — stock of a pobo eshop per warehouse: in stock, held by unshipped orders, available (sellable now) and the minimum. code (variant codes or EANs, up to 100) checks a list (unknown ones in not_found); or filter the catalogue by query, warehouse_code and state (out_of_stock, below_minimum).

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `code` | array of string | no | variant codes or EANs Max items 100. |
| `query` | string | no | product name, variant code or EAN contains |
| `warehouse_code` | string | no | limits state to this warehouse |
| `state` | string | no | One of: `out_of_stock`, `below_minimum`. |
| `limit` | integer | no | default 20, at most 100 |
| `offset` | integer | no |  |

### `list_stock_movement`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

"Why did the stock change?" — history of stock changes of a pobo eshop, newest first: kind (receipt, issue, transfer_out, transfer_in, stocktake, adjustment, sync, order_ship), the change, stock after it, the document or order behind it and who made it. Filters: variant_code (code or EAN), warehouse_code, kind, dates. It starts when stock tracking was switched on.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `variant_code` | string | no | variant code or EAN |
| `warehouse_code` | string | no |  |
| `kind` | string | no | One of: `receipt`, `issue`, `transfer_out`, `transfer_in`, `stocktake`, `adjustment`, `sync`, `order_ship`. |
| `document_id` | integer | no | movements a confirmed document wrote |
| `date_from` | string | no | YYYY-MM-DD |
| `date_to` | string | no | YYYY-MM-DD |
| `limit` | integer | no | default 20, at most 100 |
| `offset` | integer | no |  |

### `list_try_on_product`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

The variants a visitor can try on with the glasses try-on of a pobo eshop: product and variant name, the photo the AI copies the frame from (image_url, product_image_id — it must show the frame from the front), on or off. Search by product name or variant code, page with limit/offset. To swap a photo list the product photos with list_content_image (content_type product) and pass the id to update_try_on_product.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `query` | string | no | Product name or variant code. |
| `is_enabled` | boolean | no | Only switched-on (true) or switched-off (false) variants. |
| `limit` | integer | no | Default 50. Min 1. Max 100. |
| `offset` | integer | no | Min 0. |

### `list_warehouse_document`

**Server:** merchant (`/mcp/client`)  
**Access:** read-only  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** not counted

Warehouse documents of a pobo eshop (receipt, issue, transfer, stocktake), newest first, by kind, status (draft, done, cancelled) and query (number, supplier or note); document_id (up to 20) returns those with their lines.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `document_id` | array of integer | no | Max items 20. |
| `kind` | string | no | One of: `receipt`, `issue`, `transfer`, `stocktake`. |
| `status` | string | no | One of: `draft`, `done`, `cancelled`. |
| `query` | string | no |  |
| `limit` | integer | no | default 20, at most 100 |
| `offset` | integer | no |  |

### `save_branch_employee`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Add / change a person on the branch team" of a pobo eshop, up to 20 at once; with employee_id (list_branch_employee) changed, without created. name: full name with a degree, required on create; title: language => position, up to 120 characters; bio: language => a few sentences of HTML (p, br, strong, em, a), up to 400 visible characters; branch_id: every branch the person works at (replaces the list); is_active=false hides them. The page shows a person only after the merchant confirms in the admin that they agreed to publishing name and photo — remind them. Photo: set_branch_employee_photo. Deleting is done by the merchant in the admin. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `employee` | array of object | yes |  |
| `employee[].employee_id` | integer | no | the person to change; leave out to create one |
| `employee[].name` | string | no |  |
| `employee[].title` | object | no |  |
| `employee[].title.default` | string | no |  |
| `employee[].title.cs` | string | no |  |
| `employee[].title.sk` | string | no |  |
| `employee[].title.en` | string | no |  |
| `employee[].title.de` | string | no |  |
| `employee[].title.pl` | string | no |  |
| `employee[].title.hu` | string | no |  |
| `employee[].title.ro` | string | no |  |
| `employee[].title.vi` | string | no |  |
| `employee[].title.it` | string | no |  |
| `employee[].title.es` | string | no |  |
| `employee[].title.bg` | string | no |  |
| `employee[].title.hr` | string | no |  |
| `employee[].title.fr` | string | no |  |
| `employee[].title.uk` | string | no |  |
| `employee[].title.sl` | string | no |  |
| `employee[].title.pt` | string | no |  |
| `employee[].title.nl` | string | no |  |
| `employee[].title.sr` | string | no |  |
| `employee[].bio` | object | no |  |
| `employee[].bio.default` | string | no |  |
| `employee[].bio.cs` | string | no |  |
| `employee[].bio.sk` | string | no |  |
| `employee[].bio.en` | string | no |  |
| `employee[].bio.de` | string | no |  |
| `employee[].bio.pl` | string | no |  |
| `employee[].bio.hu` | string | no |  |
| `employee[].bio.ro` | string | no |  |
| `employee[].bio.vi` | string | no |  |
| `employee[].bio.it` | string | no |  |
| `employee[].bio.es` | string | no |  |
| `employee[].bio.bg` | string | no |  |
| `employee[].bio.hr` | string | no |  |
| `employee[].bio.fr` | string | no |  |
| `employee[].bio.uk` | string | no |  |
| `employee[].bio.sl` | string | no |  |
| `employee[].bio.pt` | string | no |  |
| `employee[].bio.nl` | string | no |  |
| `employee[].bio.sr` | string | no |  |
| `employee[].position` | integer | no | order in the team, the same at every branch |
| `employee[].is_active` | boolean | no |  |
| `employee[].branch_id` | array of integer | no | see list_branch |

### `save_branch_notice`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"We are closed on Friday / a notice on the branch page" — creates a notice on a branch page of a pobo eshop or changes one (notice_id from get_branch). kind: info, warning or closed (coloured by the storefront); text: language (default = main, cs, sk, …) => up to 500 characters, default required on create; valid_from / valid_to (YYYY-MM-DD, Europe/Prague) limit when it shows, empty = always; is_active=false hides it. Notices are never deleted. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `branch_id` | integer | yes | see list_branch |
| `notice_id` | integer | no | the notice to change; leave out to create one |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `kind` | string | no | One of: `info`, `warning`, `closed`. |
| `text` | object | no |  |
| `text.default` | string | no |  |
| `text.cs` | string | no |  |
| `text.sk` | string | no |  |
| `text.en` | string | no |  |
| `text.de` | string | no |  |
| `text.pl` | string | no |  |
| `text.hu` | string | no |  |
| `text.ro` | string | no |  |
| `text.vi` | string | no |  |
| `text.it` | string | no |  |
| `text.es` | string | no |  |
| `text.bg` | string | no |  |
| `text.hr` | string | no |  |
| `text.fr` | string | no |  |
| `text.uk` | string | no |  |
| `text.sl` | string | no |  |
| `text.pt` | string | no |  |
| `text.nl` | string | no |  |
| `text.sr` | string | no |  |
| `valid_from` | string | no | YYYY-MM-DD |
| `valid_to` | string | no | YYYY-MM-DD |
| `is_active` | boolean | no |  |

### `save_branch_service`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

Creates a service branches of a pobo eshop can offer ("Eye test") or changes one (service_id from list_branch_service). code: a-z, 0-9, - and _, unique, required on create; icon: one of the storefront's icons; name: language => up to 120 characters, default required on create; is_active=false hides it everywhere. Assign it with update_branch service_id. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `service_id` | integer | no | the service to change; leave out to create one |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `code` | string | no | Max length 64. |
| `icon` | string | no | One of: `eye_test`, `contact_lens`, `kids`, `sunglasses`, `repair`, `parking`, `accessible`, `gift`. |
| `name` | object | no |  |
| `name.default` | string | no |  |
| `name.cs` | string | no |  |
| `name.sk` | string | no |  |
| `name.en` | string | no |  |
| `name.de` | string | no |  |
| `name.pl` | string | no |  |
| `name.hu` | string | no |  |
| `name.ro` | string | no |  |
| `name.vi` | string | no |  |
| `name.it` | string | no |  |
| `name.es` | string | no |  |
| `name.bg` | string | no |  |
| `name.hr` | string | no |  |
| `name.fr` | string | no |  |
| `name.uk` | string | no |  |
| `name.sl` | string | no |  |
| `name.pt` | string | no |  |
| `name.nl` | string | no |  |
| `name.sr` | string | no |  |
| `position` | integer | no |  |
| `is_active` | boolean | no |  |

### `set_branch_banner`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Change the picture at the top of the branch page" of a pobo eshop from image_url (public https; JPEG, PNG or WebP up to 10 MB; wide, ideally 1600 × 400 px), replacing the previous banner; remove=true removes it. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `branch_id` | integer | yes | see list_branch |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `image_url` | string | no | public https address of the picture |
| `remove` | boolean | no | true removes the banner |

### `set_branch_employee_photo`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

Sets the photo of a person of the branch team of a pobo eshop from image_url (public https; JPEG, PNG or WebP up to 10 MB; a square portrait), replacing the previous one; remove=true removes it. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `employee_id` | integer | yes | see list_branch_employee |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `image_url` | string | no | public https address of the photo |
| `remove` | boolean | no | true removes the photo |

### `set_storefront_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Put my own picture on the product page / category" of a pobo eshop from a public https address, replacing the previous one: product → banner (the wide picture of the product page; JPEG, PNG or WebP up to 5 MB, ideally 1600 × 400 px); category → preview (the photo of category tiles), banner (behind the heading) or icon (menu icon: SVG, or a square PNG of at least 64 × 64 px, up to 100 KB). remove=true removes it. Product gallery photos are sync_product_image / update_product; a generated banner generate_product_banner / generate_category_banner. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_type` | string | yes | One of: `product`, `category`. |
| `entity_id` | integer | yes | product id (find_product) or category id (list_category) |
| `kind` | string | yes | product: banner only One of: `banner`, `preview`, `icon`. |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `image_url` | string | no | public https address of the picture |
| `remove` | boolean | no | true removes the picture |

### `update_branch`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Change the opening hours / address / texts of a branch" of a pobo eshop; only what you send changes. is_active switches the branch page off and on (branches are never deleted). A day of opening_hour you send replaces only that day; closed and service_id replace the whole list, so read get_branch first. warehouse_id: the warehouse whose stock the branch shows and reserves from (one warehouse = one branch; see list_stock for codes, ids in get_branch of other branches). slot_minute (10–240) and slot_capacity (1–20): length and capacity of a reservation slot. position orders the branches. country: ISO code (CZ, SK, …). opening_hour: ISO weekday "1"–"7" => list of {from, to} in HH:MM, in order, without overlaps; an empty list = closed that day. closed: the full list of extra closed days (YYYY-MM-DD). translation: language (default = main language, then a content language code: cs, sk, en, ro, …) => name, description (HTML: paragraphs, lists, h2/h3, links), seo_title, seo_description, parking, accessibility, transport. service_id: the full list of services (list_branch_service). slug: language => the page address part; the old address keeps redirecting. A temporary change such as "closed on Friday" is a notice — save_branch_notice. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `branch_id` | integer | yes | see list_branch |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `is_active` | boolean | no |  |
| `warehouse_id` | integer | no |  |
| `street` | string | no |  |
| `city` | string | no |  |
| `zip` | string | no |  |
| `country` | string | no | One of: `AT`, `BE`, `BG`, `CY`, `CZ`, `DE`, `DK`, `EE`, `ES`, `FI`, `FR`, `GR`, `HR`, `HU`, `IE`, `IT`, `LT`, `LU`, `LV`, `MT`, `NL`, `PL`, `PT`, `RO`, `SE`, `SI`, `SK`. |
| `phone` | string | no |  |
| `email` | string | no |  |
| `lat` | number | no |  |
| `lng` | number | no |  |
| `opening_hour` | object | no | ISO weekday "1"–"7" => [{"from": "09:00", "to": "17:00"}]; empty list = closed that day |
| `slot_minute` | integer | no |  |
| `slot_capacity` | integer | no |  |
| `position` | integer | no |  |
| `closed` | array of string | no | the full list of extra closed days, YYYY-MM-DD |
| `translation` | object | no |  |
| `translation.default` | object | no |  |
| `translation.default.name` | string | no | Max length 255. |
| `translation.default.description` | string | no | HTML |
| `translation.default.seo_title` | string | no | Max length 255. |
| `translation.default.seo_description` | string | no | Max length 1000. |
| `translation.default.parking` | string | no |  |
| `translation.default.accessibility` | string | no |  |
| `translation.default.transport` | string | no |  |
| `translation.cs` | object | no |  |
| `translation.cs.name` | string | no | Max length 255. |
| `translation.cs.description` | string | no | HTML |
| `translation.cs.seo_title` | string | no | Max length 255. |
| `translation.cs.seo_description` | string | no | Max length 1000. |
| `translation.cs.parking` | string | no |  |
| `translation.cs.accessibility` | string | no |  |
| `translation.cs.transport` | string | no |  |
| `translation.sk` | object | no |  |
| `translation.sk.name` | string | no | Max length 255. |
| `translation.sk.description` | string | no | HTML |
| `translation.sk.seo_title` | string | no | Max length 255. |
| `translation.sk.seo_description` | string | no | Max length 1000. |
| `translation.sk.parking` | string | no |  |
| `translation.sk.accessibility` | string | no |  |
| `translation.sk.transport` | string | no |  |
| `translation.en` | object | no |  |
| `translation.en.name` | string | no | Max length 255. |
| `translation.en.description` | string | no | HTML |
| `translation.en.seo_title` | string | no | Max length 255. |
| `translation.en.seo_description` | string | no | Max length 1000. |
| `translation.en.parking` | string | no |  |
| `translation.en.accessibility` | string | no |  |
| `translation.en.transport` | string | no |  |
| `translation.de` | object | no |  |
| `translation.de.name` | string | no | Max length 255. |
| `translation.de.description` | string | no | HTML |
| `translation.de.seo_title` | string | no | Max length 255. |
| `translation.de.seo_description` | string | no | Max length 1000. |
| `translation.de.parking` | string | no |  |
| `translation.de.accessibility` | string | no |  |
| `translation.de.transport` | string | no |  |
| `translation.pl` | object | no |  |
| `translation.pl.name` | string | no | Max length 255. |
| `translation.pl.description` | string | no | HTML |
| `translation.pl.seo_title` | string | no | Max length 255. |
| `translation.pl.seo_description` | string | no | Max length 1000. |
| `translation.pl.parking` | string | no |  |
| `translation.pl.accessibility` | string | no |  |
| `translation.pl.transport` | string | no |  |
| `translation.hu` | object | no |  |
| `translation.hu.name` | string | no | Max length 255. |
| `translation.hu.description` | string | no | HTML |
| `translation.hu.seo_title` | string | no | Max length 255. |
| `translation.hu.seo_description` | string | no | Max length 1000. |
| `translation.hu.parking` | string | no |  |
| `translation.hu.accessibility` | string | no |  |
| `translation.hu.transport` | string | no |  |
| `translation.ro` | object | no |  |
| `translation.ro.name` | string | no | Max length 255. |
| `translation.ro.description` | string | no | HTML |
| `translation.ro.seo_title` | string | no | Max length 255. |
| `translation.ro.seo_description` | string | no | Max length 1000. |
| `translation.ro.parking` | string | no |  |
| `translation.ro.accessibility` | string | no |  |
| `translation.ro.transport` | string | no |  |
| `translation.vi` | object | no |  |
| `translation.vi.name` | string | no | Max length 255. |
| `translation.vi.description` | string | no | HTML |
| `translation.vi.seo_title` | string | no | Max length 255. |
| `translation.vi.seo_description` | string | no | Max length 1000. |
| `translation.vi.parking` | string | no |  |
| `translation.vi.accessibility` | string | no |  |
| `translation.vi.transport` | string | no |  |
| `translation.it` | object | no |  |
| `translation.it.name` | string | no | Max length 255. |
| `translation.it.description` | string | no | HTML |
| `translation.it.seo_title` | string | no | Max length 255. |
| `translation.it.seo_description` | string | no | Max length 1000. |
| `translation.it.parking` | string | no |  |
| `translation.it.accessibility` | string | no |  |
| `translation.it.transport` | string | no |  |
| `translation.es` | object | no |  |
| `translation.es.name` | string | no | Max length 255. |
| `translation.es.description` | string | no | HTML |
| `translation.es.seo_title` | string | no | Max length 255. |
| `translation.es.seo_description` | string | no | Max length 1000. |
| `translation.es.parking` | string | no |  |
| `translation.es.accessibility` | string | no |  |
| `translation.es.transport` | string | no |  |
| `translation.bg` | object | no |  |
| `translation.bg.name` | string | no | Max length 255. |
| `translation.bg.description` | string | no | HTML |
| `translation.bg.seo_title` | string | no | Max length 255. |
| `translation.bg.seo_description` | string | no | Max length 1000. |
| `translation.bg.parking` | string | no |  |
| `translation.bg.accessibility` | string | no |  |
| `translation.bg.transport` | string | no |  |
| `translation.hr` | object | no |  |
| `translation.hr.name` | string | no | Max length 255. |
| `translation.hr.description` | string | no | HTML |
| `translation.hr.seo_title` | string | no | Max length 255. |
| `translation.hr.seo_description` | string | no | Max length 1000. |
| `translation.hr.parking` | string | no |  |
| `translation.hr.accessibility` | string | no |  |
| `translation.hr.transport` | string | no |  |
| `translation.fr` | object | no |  |
| `translation.fr.name` | string | no | Max length 255. |
| `translation.fr.description` | string | no | HTML |
| `translation.fr.seo_title` | string | no | Max length 255. |
| `translation.fr.seo_description` | string | no | Max length 1000. |
| `translation.fr.parking` | string | no |  |
| `translation.fr.accessibility` | string | no |  |
| `translation.fr.transport` | string | no |  |
| `translation.uk` | object | no |  |
| `translation.uk.name` | string | no | Max length 255. |
| `translation.uk.description` | string | no | HTML |
| `translation.uk.seo_title` | string | no | Max length 255. |
| `translation.uk.seo_description` | string | no | Max length 1000. |
| `translation.uk.parking` | string | no |  |
| `translation.uk.accessibility` | string | no |  |
| `translation.uk.transport` | string | no |  |
| `translation.sl` | object | no |  |
| `translation.sl.name` | string | no | Max length 255. |
| `translation.sl.description` | string | no | HTML |
| `translation.sl.seo_title` | string | no | Max length 255. |
| `translation.sl.seo_description` | string | no | Max length 1000. |
| `translation.sl.parking` | string | no |  |
| `translation.sl.accessibility` | string | no |  |
| `translation.sl.transport` | string | no |  |
| `translation.pt` | object | no |  |
| `translation.pt.name` | string | no | Max length 255. |
| `translation.pt.description` | string | no | HTML |
| `translation.pt.seo_title` | string | no | Max length 255. |
| `translation.pt.seo_description` | string | no | Max length 1000. |
| `translation.pt.parking` | string | no |  |
| `translation.pt.accessibility` | string | no |  |
| `translation.pt.transport` | string | no |  |
| `translation.nl` | object | no |  |
| `translation.nl.name` | string | no | Max length 255. |
| `translation.nl.description` | string | no | HTML |
| `translation.nl.seo_title` | string | no | Max length 255. |
| `translation.nl.seo_description` | string | no | Max length 1000. |
| `translation.nl.parking` | string | no |  |
| `translation.nl.accessibility` | string | no |  |
| `translation.nl.transport` | string | no |  |
| `translation.sr` | object | no |  |
| `translation.sr.name` | string | no | Max length 255. |
| `translation.sr.description` | string | no | HTML |
| `translation.sr.seo_title` | string | no | Max length 255. |
| `translation.sr.seo_description` | string | no | Max length 1000. |
| `translation.sr.parking` | string | no |  |
| `translation.sr.accessibility` | string | no |  |
| `translation.sr.transport` | string | no |  |
| `service_id` | array of integer | no | the full list of services |
| `slug` | object | no | language => slug of the branch page |
| `slug.default` | string | no |  |
| `slug.cs` | string | no |  |
| `slug.sk` | string | no |  |
| `slug.en` | string | no |  |
| `slug.de` | string | no |  |
| `slug.pl` | string | no |  |
| `slug.hu` | string | no |  |
| `slug.ro` | string | no |  |
| `slug.vi` | string | no |  |
| `slug.it` | string | no |  |
| `slug.es` | string | no |  |
| `slug.bg` | string | no |  |
| `slug.hr` | string | no |  |
| `slug.fr` | string | no |  |
| `slug.uk` | string | no |  |
| `slug.sl` | string | no |  |
| `slug.pt` | string | no |  |
| `slug.nl` | string | no |  |
| `slug.sr` | string | no |  |

### `update_branch_image`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

Changes gallery photos of a branch of a pobo eshop (image_id from get_branch): image = list of {image_id, alt (language => description, replaces them), remove (true deletes the photo from the CDN — cannot be undone)}; order = every image_id of the branch once, the main photo first (after removals). Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `branch_id` | integer | yes | see list_branch |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `image` | array of object | no | Max items 20. |
| `image[].image_id` | integer | yes |  |
| `image[].alt` | object | no |  |
| `image[].alt.default` | string | no |  |
| `image[].alt.cs` | string | no |  |
| `image[].alt.sk` | string | no |  |
| `image[].alt.en` | string | no |  |
| `image[].alt.de` | string | no |  |
| `image[].alt.pl` | string | no |  |
| `image[].alt.hu` | string | no |  |
| `image[].alt.ro` | string | no |  |
| `image[].alt.vi` | string | no |  |
| `image[].alt.it` | string | no |  |
| `image[].alt.es` | string | no |  |
| `image[].alt.bg` | string | no |  |
| `image[].alt.hr` | string | no |  |
| `image[].alt.fr` | string | no |  |
| `image[].alt.uk` | string | no |  |
| `image[].alt.sl` | string | no |  |
| `image[].alt.pt` | string | no |  |
| `image[].alt.nl` | string | no |  |
| `image[].alt.sr` | string | no |  |
| `image[].remove` | boolean | no |  |
| `order` | array of integer | no | every image_id of the branch once, the main photo first |

### `update_branch_setting`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

Sets schema_type of all branch pages of a pobo eshop — the schema.org type search engines read: LocalBusiness (any shop), Optician or Store. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `schema_type` | string | yes | One of: `LocalBusiness`, `Optician`, `Store`. |

### `update_try_on_product`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

Changes try-on variants of a pobo eshop, up to 100 per call (id from list_try_on_product): is_enabled switches a variant; product_image_id swaps the photo the AI copies the frame from — a photo of the same product (list_content_image, content_type product) showing the frame from the front; a side view or a person wearing it gives a poor result. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `item` | array of object | yes |  |
| `item[].id` | integer | yes |  |
| `item[].is_enabled` | boolean | no |  |
| `item[].product_image_id` | integer | no |  |

### `update_try_on_setting`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

"Switch the glasses try-on on / off" for a pobo eshop and set it; only the fields sent change. category_id = categories with glasses (replaces the list; from list_category), then fill_try_on_product offers their products; daily_limit = try-ons per day for the eshop (each costs about 0.02 USD); ip_hour_limit = per visitor per hour; cta = the button under the result: reserve (at a branch) or cart. Give a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `is_enabled` | boolean | no | Show the try-on on the storefront. |
| `category_id` | array of integer | no | Categories with glasses; replaces the list. |
| `daily_limit` | integer | no | Min 1. Max 10000. |
| `ip_hour_limit` | integer | no | Min 1. Max 100. |
| `cta` | string | no | One of: `reserve`, `cart`. |

### `update_warehouse_document`

**Server:** merchant (`/mcp/client`)  
**Access:** writes  
**Costs credits:** no  
**Available:** Pobo storefront only (platform `pobo`)  
**Rate limit:** counted

Replaces a DRAFT warehouse document of a pobo eshop: warehouses, supplier, note and ALL its lines (send the whole list, lines left out are removed). Only a draft changes — a confirmed document never does; a mistake there is fixed by a new document. Changes no stock.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `reason` | string | no | Why you are making this change, in one sentence the merchant will read in the overview of agent changes ("the description did not say which car models the mat fits"). Max length 1000. |
| `document_id` | integer | yes |  |
| `warehouse_code` | string | no | the default warehouse when left out |
| `target_warehouse_code` | string | no | transfer only |
| `supplier` | string | no | receipt only |
| `note` | string | no |  |
| `line` | array of object | yes | Max items 500. |
| `line[].code` | string | yes | variant code or EAN |
| `line[].quantity` | integer | yes |  |

## White label content

All tools in this section live on the white label server (`/mcp/whitelabel`) and are gated to eshops with `platform = b2b` — calling any of them against an eshop on another platform returns "Eshop not found."

Tools: [`add_entity_product_card`](#add_entity_product_card), [`add_entity_widget`](#add_entity_widget), [`compose_entity_content`](#compose_entity_content), [`copy_entity_widget`](#copy_entity_widget), [`create_entity`](#create_entity), [`edit_entity_product_card`](#edit_entity_product_card), [`edit_entity_widget`](#edit_entity_widget), [`fill_design_content`](#fill_design_content), [`get_design_structure`](#get_design_structure), [`get_entity_content`](#get_entity_content), [`get_entity_history`](#get_entity_history), [`get_entity_html`](#get_entity_html), [`get_entity_widget_catalog`](#get_entity_widget_catalog), [`list_entity`](#list_entity), [`list_entity_batch`](#list_entity_batch), [`move_entity_widget`](#move_entity_widget), [`remove_entity_widget`](#remove_entity_widget), [`revert_entity`](#revert_entity), [`set_entity_design`](#set_entity_design)

### `add_entity_product_card`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Add a block of product cards with YOUR own products to one entity or to up to 50 at once. Pass your product ids in product_id (at most 10, in the order to show) — Pobo does not hold your products and cannot check the ids: the cards are filled in the visitor's browser from the product endpoint set in Pobo (Integration → White label), and an id your endpoint does not return is simply left out, so a typo shows only as a missing card. layout: carousel (default), single (one large card per product), grid, card_left / card_right (the card next to a text column — send text: <h2> heading and <p> paragraphs presenting the product). The whole run shares a batch_id that revert_entity can undo. product_endpoint_set=false in the answer means the eshop has no endpoint yet and the cards will not show until one is set.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | array of string | yes | Host ids of the entities to add the block to, 50 at most (see list_entity). |
| `entity_type` | string | yes | What the entities are. One of: `product`, `category`, `blog`, `page`. |
| `product_id` | array of string | yes | Your own product ids, in the order to show, 10 at most; card_left and card_right take exactly one. |
| `layout` | string | no | carousel (default), single, grid, card_left or card_right. One of: `carousel`, `single`, `grid`, `card_left`, `card_right`. |
| `text` | string | no | HTML of the text column (<h2>, <h3>, <p>). Required by card_left and card_right, not allowed with the other layouts. |
| `position` | integer | no | Where to place the block, 1 = first. Omit to append at the end. |

### `add_entity_widget`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Add a widget to one entity or to up to 50 at once. Pick the template with get_entity_widget_catalog and pass its widget_id — never guess one, and keep the same widget_id for the whole batch so the entities end up looking alike. Same content everywhere goes in content; different content per entity goes in content_per_entity keyed by entity id. A non-repeatable widget holds one item, so five FAQ questions are five calls, not one. The text is written into every language of the eshop. The whole run shares a batch_id that revert_entity can undo. Hotspot widget (ai_widget_type hotspot): each item also takes x, y (0–100, percent of the photo from the left/top) and placement (above|below); 1–8 items, one point each. Look at the photo before you place points.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | array of string | yes | Host ids of the entities to add the widget to, 50 at most (see list_entity). |
| `entity_type` | string | yes | What the entities are. One of: `product`, `category`, `blog`, `page`. |
| `widget_id` | integer | yes | Widget template to place (see get_entity_widget_catalog). |
| `content` | object | no | Texts keyed by role, e.g. {"question": "…", "answer": "<p>…</p>"}. Used for every entity that content_per_entity does not name. A repeatable widget carries its values in content.item. |
| `content_per_entity` | object | no | Per-entity content: {"<entity_id>": {"<role>": "text"}}. Overrides content for the entities it names. |
| `image` | array of string | no | Your own https image URLs, filled into the image slots in the order given. |
| `position` | integer | no | Where to place the widget, 1 = first. Omit to append at the end. |

At least one of `content` or `content_per_entity` is required; if both are empty the call errors.

**Returns:** `batch_id`, a `summary` sentence, and a per-entity `entity` array with `status` (`added`, `error`, or `not_found`), `widget_instance_id` and `position` on success. If `content_per_entity` names an id not present in `entity_id`, the response adds `unmatched_content_key` and a `warning` — those entities silently received the shared `content` instead.

**Careful:** an entity with no applicable content (neither in `content` nor `content_per_entity`) is reported as `error`, not silently skipped, but the batch as a whole still runs for the other entities. A widget instance that gets text longer than its `max_length` hint is written anyway (length is advisory) and flagged with a `warning` per entity.

### `compose_entity_content`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Build the whole description of one entity in a single call: pick the widgets yourself with get_entity_widget_catalog, order them, and write their texts. No template is involved. mode=replace (the default) makes the entity hold exactly what you send; mode=append puts your widgets after what is already there. Images belong to the widget that shows them and each may carry an alt. The entity is created when it does not exist yet. The whole call is one version that revert_entity can undo. Hotspot widget (ai_widget_type hotspot): each item also takes x, y (0–100, percent of the photo from the left/top) and placement (above|below); 1–8 items, one point each. Look at the photo before you place points.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Your own id of the entity. It is created when unknown. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `widget` | array of any | yes | The widgets in the order they should appear, 30 at most. Each is {widget_id, content, image}: content holds the texts by role, image holds your own https photo URLs, each either a URL or {url, alt}. |
| `mode` | string | no | replace (default) makes the entity hold exactly these widgets; append adds them after the current content. One of: `replace`, `append`. |
| `title` | string | no | Name of the entity, used by list_entity when searching. |

**Returns:** `entity_id`, `entity_created` (bool), `widget_count`, `mode`, `batch_id`, and a `message`. Includes `warning` when any widget's content exceeded its length hint.

**Careful:** `mode=replace` is the default and force-deletes every existing widget of the entity before writing the new ones. The whole composition is validated up front — if any widget fails validation, nothing is written at all (no partial replace that could leave the entity with a half-empty description). `mode=append` instead adds the new widgets after the current content and does not delete anything.

### `copy_entity_widget`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Copy widgets from one entity to up to 50 others. mode="append" adds them at the end and keeps what is there; mode="replace" throws the target content away first — say which one you are about to do before you call it. Pass widget_instance_id to copy only some widgets of the source. The run shares a batch_id that revert_entity can undo.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `source_entity_id` | string | yes | Host id of the entity to copy FROM. Max length 64. |
| `target_entity_id` | array of string | yes | Host ids of the entities to copy TO, 50 at most. |
| `entity_type` | string | yes | What the entities are — source and targets share it. One of: `product`, `category`, `blog`, `page`. |
| `mode` | string | no | append = add to what the target already has (default); replace = throw the target content away first. One of: `append`, `replace`. |
| `widget_instance_id` | array of integer | no | Copy only these widgets of the source (see get_entity_content). Omit to copy all of them. |

**Returns:** `batch_id`, `mode`, a `summary` sentence, and a per-target `entity` array with `status` (`copied` or `error`) and `widget_copied` count.

**Careful:** `mode=replace` force-deletes the target entity's existing widgets before copying in the source's widgets — the tool description explicitly tells the caller to state which mode it is about to use before calling. A target that is the same entity as the source is rejected per-entity with an error, not treated as a no-op.

### `create_entity`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Register a product, category, blog article or page of the host in Pobo so content tools can write to it, without giving it a description template. Use it when add_entity_widget or compose_entity_content answers "Entity not found" — that means Pobo has not seen this entity_id yet. The entity is created with no template, which is what the host's own entities normally have; fill_design_content is the call to use instead when the description SHOULD follow a template. Registering an entity that already exists changes nothing and returns it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Host id of the entity — the same id the host uses in its own system and in the SDK. Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `title` | string | no | Entity name, for the listing in the Pobo administration. Optional. Max length 255. |

### `edit_entity_product_card`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Change which of YOUR products a product card block shows. get_entity_content lists the block's widget_instance_id and its current product_id; send the complete new list in the order to show (add = current list + new id, remove = list without it, reorder = same ids in a new order), at most 10. The layout and the text next to the card stay — the text changes with edit_entity_widget. Versioned, undoable with revert_entity.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Host id of the entity holding the block. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `widget_instance_id` | integer | yes | The product card block (get_entity_content). |
| `product_id` | array of string | yes | Your own product ids the block shows after the edit, in order, 10 at most; a card next to text (card_left, card_right) holds exactly one. |

### `edit_entity_widget`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Rewrite the texts of widgets that already exist. Either one widget by widget_instance_id (from get_entity_content), or every instance of one template across up to 50 entities by widget_id. Only the roles you send change — other roles and the structure stay as they are, and the text is written into every language. Images change with image and icon, keyed by the slot index that get_entity_content reports; url and alt are independent, so an alt can be set without replacing the photo, and icons take alt only. The run shares a batch_id that revert_entity can undo. Hotspot widget (ai_widget_type hotspot): each item also takes x, y (0–100, percent of the photo from the left/top) and placement (above|below); 1–8 items, one point each. Look at the photo before you place points.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | array of string | yes | Host ids of the entities to edit, 50 at most. |
| `entity_type` | string | yes | What the entities are. One of: `product`, `category`, `blog`, `page`. |
| `content` | object | no | New texts keyed by role, e.g. {"paragraph": "<p>…</p>"}. A repeatable widget carries its values in content.item. |
| `image` | object | no | Photo slots to change, keyed by the slot index from get_entity_content, e.g. {"1": {"url": "https://…", "alt": "Blue pool 3 × 4 m"}}. url and alt are independent — send alt alone to describe the photo without replacing it. |
| `icon` | object | no | Icon slots, keyed the same way. Icons take alt only; their image cannot be replaced here. |
| `widget_instance_id` | integer | no | Edit exactly this one widget (see get_entity_content). Only with a single entity_id. |
| `widget_id` | integer | no | Edit every instance of this template on the given entities. |

Exactly one of `widget_instance_id` or `widget_id` must be given, and at least one of `content`/`image`/`icon` must be non-empty.

**Returns:** `batch_id`, a `summary`, and a per-entity `entity` array with `status` (`edited`, `error`, or `not_found`), `widget_changed` and `field_written` counts. Includes `warning` when content exceeded its length hint.

**Careful:** a slot index outside the widget's actual image/icon count is rejected as an error rather than silently ignored. If nothing sent actually matches anything on the widget, the call reports `error` ("Nothing was written") rather than a false `edited`. Only the roles/slots explicitly sent are touched — everything else on the widget is left as is.

### `fill_design_content`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Fill a design template with your own texts, images and icons and get the rendered HTML back. No AI runs on the server and it costs no credits — the words are yours. Call get_design_structure first: address slots by design_widget_id, respect max_length (measured on the text, markup excluded) and item counts. A widget you do not send is not rendered at all, and a second call REPLACES the whole content of the entity. The entity is registered automatically if it does not exist yet.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Your own id of the item, max 64 characters. Registered automatically on first use. Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `design_id` | integer | yes | Design template to fill (see list_design, get_design_structure). |
| `widget` | array of object | yes | Widgets to fill: [{"design_widget_id": 71, "content": {"<role>": "text"}, "image": ["https://…"], "icon": ["https://…"]}]. A repeatable widget carries its values in content.item as an array of objects, with exactly item_count entries. Images and icons are your own https URLs, filled in the order given. |
| `lang` | string | no | Which language of the rendered HTML to return; omit for the default one. The content itself is written into every language of the eshop either way. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |
| `title` | string | no | Name of the item, stored on the entity when it is registered by this call. Ignored for an entity that already exists. Max length 255. |

**Returns:** `entity_id`, `entity_type`, `design_id`, `widget_created` (count), `html` (keyed by the requested language, or all languages), and `warning` when any text exceeded its `max_length` hint.

**Careful:** every call REPLACES the entity's whole content, not just the widgets sent — confirmed in code (`EntityDesignFillService::fill()`, and the class docblock states the tool is deliberately not marked idempotent because "volání nahrazuje celý obsah entity"). A widget template not included in `widget` is not rendered at all, so a second call with a shorter widget list drops whatever was left out the first time; there is no append mode. The entity is registered automatically (`WhiteLabel::registerEntity()`) if `entity_id`/`entity_type` do not resolve to an existing one — this tool never errors with "entity not found."

### `get_design_structure`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

Describe what a design template accepts: for every widget its design_widget_id, roles with format and max_length, how many items a repeatable widget holds, and how many image/icon slots it has. Read this before fill_design_content — the limits are a property of the template, not a convention. Widgets that cannot be filled stay in the list with a reason.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `design_id` | integer | yes | Design template id (see list_design). |

**Returns:** the template's widget/slot description (via `EntityDesignFillService::describe()`) — widgets that cannot be filled are still listed, with a reason.

### `get_entity_content`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

What is written on one white label entity: the ordered widget list with position, widget_instance_id, widget_id, template name, image slot count and the texts keyed by role. Repeatable widgets list their roles once per item, in document order. Read it before editing anything — widget_instance_id comes from here.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Host id of the entity (see list_entity). Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `lang` | string | no | Which language of the texts to read; omit for the default one. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

**Returns:** `entity_id`, `entity_type`, `title`, `design_id`, `lang`, and `widget` (the ordered snapshot). Repeatable widgets list their roles once per item, in document order.

### `get_entity_history`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

Versions of one entity, newest first: version_id, when it was taken, which channel and operation caused it, how many widgets it holds and the batch_id of the run. A version is the state BEFORE that write, so reverting to it undoes the write. Pass a version_id to revert_entity.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Host id of the entity (see list_entity). Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `limit` | integer | no | How many versions to return, 50 at most (default 20). |

**Returns:** `entity_id`, `entity_type`, and `version`, an array of `{version_id, created_at, source, tool, user_id, batch_id, widget_count}`. Each version is the state BEFORE the write that produced it, so reverting to it undoes that write.

### `get_entity_html`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

Read the current HTML of a white label entity (product, category, blog or page) as it is stored in Pobo right now. Use it after fill_design_content, or whenever the merchant may have edited the content in the Pobo editor since you last stored it.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | The host's own id of the item, as used in fill_design_content. Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `lang` | string | no | Which language to return; omit for the default one. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

**Returns:** `entity_id`, `entity_type`, `html` (keyed by language, or just the requested one). Errors with "Entity has no content yet" if the entity has never been filled.

### `get_entity_widget_catalog`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

Widget templates that can be placed on a white label entity, with their slots: text roles (role, hint, max_length), image slot count and item container count. Pick one and pass its widget_id when adding a widget — never guess a template.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |

**Returns:** `widget`, an array of widget slot descriptions (via `WidgetSchemaService::describeSlots()`).

### `list_entity`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

Find white label entities of an eshop. Without a query it lists the newest ones; with one it searches the host id, the entity name and the texts inside the widgets at once, and says which of the three matched. Case and diacritics are ignored. Use it to resolve the entities you then read with get_entity_content.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `query` | string | no | What to look for — searched in the host entity id, the entity name and the widget texts. Max length 255. |
| `type` | string | no | Restrict to one kind of entity. One of: `product`, `category`, `blog`, `page`. |
| `limit` | integer | no | How many entities to return, 50 at most (default 20). |
| `offset` | integer | no | How many entities to skip — paging through a larger result. |
| `lang` | string | no | Language of the returned names and snippets; omit for the default one. One of: `default`, `cs`, `sk`, `en`, `de`, `pl`, `hu`, `ro`, `vi`, `it`, `es`, `bg`, `hr`, `fr`, `uk`, `sl`, `pt`, `nl`, `sr`. |

**Returns:** `total`, `limit`, `offset`, and `entity` (array of `{entity_id, type, title, design_id, widget_count, updated_at}`, plus `match` info when a query was given). If the eshop holds too many widgets to search, widget text is not searched, and the response adds `content_searched: false` with a `note` explaining only ids/names were searched.

### `list_entity_batch`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** read-only  
**Costs credits:** no

Recent runs of content changes on this eshop, newest first: batch_id, which tool and channel made it, when, how many entities it touched and a few of their ids. Use it to find a run somebody wants undone — revert_entity takes the batch_id and puts every entity of that run back.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `tool` | string | no | Only runs made by this tool, e.g. "add_entity_widget". Max length 64. |
| `limit` | integer | no | How many runs to return, 50 at most (default 20). |

**Returns:** `batch`, an array of `{batch_id, tool, source, user_id, entity_count, created_at, entity_id_sample}` — `source` is the channel that made the run, `entity_count` is how many distinct entities it touched, and `entity_id_sample` is up to 5 of their ids.

### `move_entity_widget`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Move a widget to another place inside its entity. Positions are 1-based and the rest closes the gap, so position 1 puts the widget first.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Host id of the entity. Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `widget_instance_id` | integer | yes | Widget to move (see get_entity_content). |
| `position` | integer | yes | Where to put it, 1 = first. |

**Returns:** `entity_id`, `widget_instance_id`, and `widget` — the full updated widget snapshot of the entity (default language) after the move.

### `remove_entity_widget`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Remove widgets from up to 50 entities. Pick them by widget_instance_id (one widget), by widget_id (every instance of a template) or by query (every widget whose text contains it). The first call REMOVES NOTHING — it answers with what would go, and you repeat it with confirmed=true to do it. Removal is reversible: the run shares a batch_id that revert_entity can undo.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | array of string | yes | Host ids of the entities to remove from, 50 at most. |
| `entity_type` | string | yes | What the entities are. One of: `product`, `category`, `blog`, `page`. |
| `widget_instance_id` | integer | no | Remove exactly this one widget (see get_entity_content). |
| `widget_id` | integer | no | Remove every instance of this template on the given entities. |
| `query` | string | no | Remove every widget whose text contains this. Case and diacritics are ignored. Max length 255. |
| `confirmed` | boolean | no | Pass true only after you have shown the preview to the merchant and they agreed. |

**Returns (preview, `confirmed` not true):** `confirm_required: true`, `would_remove` (total widget count), `entity_affected`, a `sample` (up to 5 matched widgets with entity_id, widget_instance_id, widget_id, and up to 120 chars of text), and a `note` that nothing was removed yet.

**Returns (confirmed run):** `batch_id`, `widget_removed` (total), a `summary`, and per-entity `entity` array with `status: removed` and `widget_removed` count.

**Careful:** the first call with `confirmed` omitted or false performs no deletion regardless of how broad the filter is — this is a deliberate two-phase gate specifically because a typo in `query` could otherwise strip content from hundreds of entities in one call. The actual deletion (second call) is a soft delete of the matched widget rows (the row is marked deleted, not physically removed), but it is reversible via `revert_entity` using the returned `batch_id`, since a version snapshot is captured per entity before deleting.

### `revert_entity`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes  
**Costs credits:** no

Put content back the way it was. Either one entity to a version_id from get_entity_history, or a whole run at once by its batch_id — every entity that run touched goes back to the state before it. The current state is saved as a new version first, so a revert can itself be reverted.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | no | Host id of the entity to revert. Give it together with entity_type and version_id. Max length 64. |
| `entity_type` | string | no | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `version_id` | integer | no | Version to go back to (see get_entity_history). |
| `batch_id` | string | no | Revert a whole run instead of one entity — every entity it touched goes back to the state before it. Max length 26. |

Either `batch_id` alone, or `entity_id` + `entity_type` + `version_id` together, must be given.

**Returns (batch revert):** `batch_id`, `entity_reverted` (count), `widget_restored` (count). Errors if no entity of the eshop was touched by that `batch_id`.

**Returns (single entity revert):** `entity_id`, `entity_type`, `version_id`, `widget_restored`.

**Careful:** reverting overwrites the entity's current widget content with the older version's content — the current state is saved as a new version first (so a revert is itself revertible), but the effect is still a full replace of what's there now.

### `set_entity_design`

**Server:** white label (`/mcp/whitelabel`)  
**Access:** writes — destructive  
**Costs credits:** no

Say which description template an entity uses, or take the template off it. The exported HTML carries this as data-pobo-design-id, so an entity that once went through fill_design_content keeps announcing that template until you change it here. Send design_id to point it at a template, or design_id=null to put it back to the default the host's own entities have. This only changes the label: the widgets stay exactly as they are, so use fill_design_content when the content itself should follow a template. The previous value is versioned and revert_entity puts it back.

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `eshop_id` | integer | yes | Eshop id (see list_eshop). |
| `entity_id` | string | yes | Host id of the entity (see list_entity). Max length 64. |
| `entity_type` | string | yes | What the entity is. One of: `product`, `category`, `blog`, `page`. |
| `design_id` | integer | no | Template to point the entity at (see list_design). Send null to remove the template and go back to the default. |
