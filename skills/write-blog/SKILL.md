---
name: write-blog
description: Write and edit Pobo Page Builder descriptions — compose a blog article, a product description or a category text from widget templates, rewrite existing sections, insert photos and real product carousels, or hand a whole article to Pobo's server-side generator. Use when the user wants a blog post, magazine article, buying guide, landing text, or the description of a product or category: "write an article about…", "edit that blog post", "write descriptions for these products", "add photos to the article". Uses the `pobo` MCP server tools.
---

# Write Pobo Page Builder blog articles

You author and edit blog articles that render through Pobo Page Builder widgets
on the merchant's e-shop. All logic runs on the Pobo Page Builder backend.

Communicate with the user in their language (typically Czech) and **write the
article in the language the e-shop sells in**, not in English.

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

## Two ways to produce an article — pick deliberately

**You write it (preferred, free).** You compose the copy yourself and place it
with the deterministic tools. No credits are spent, you control every sentence,
and you can iterate as long as you like. This is the default; reach for it
unless the user explicitly wants the other one.

**Pobo's generator writes it (`generate_blog_article`, paid).** One server-side
job builds the whole article from a brief. Costs **1 credit per article, plus 2
per AI photo**. Use it when the user asks for it by name, wants a long article
produced in one shot, or wants Pobo's own house style rather than yours.

Whichever you pick, say which one you are using and why before you start — the
user should never be surprised by a credit charge.

## Workflow — you as the author

### 1. Set up

`list_eshop` → `list_blog` (existing articles, with `widget_count` and
`is_visible`) or `create_blog` for a new one.

**The same tools write products and categories.** Every widget tool below takes
`entity_type` (`product` | `category` | `blog`) and `entity_id`; an article is
simply `entity_type: "blog"`. Products and categories are not created here —
they arrive from the shop platform, so resolve them with `list_product` /
`find_product` / `list_category`. A blog article is the one entity you may
create yourself.

`get_content_widget_catalog` is the step people skip and then guess. It returns
the widget templates this e-shop actually has, and for each one its content
roles — the named slots you fill (`role`, `hint`, `max_length`, `image_slot`,
`icon_slot`, `item_count`). **You can only use widgets from this catalog**;
anything else is an error.

### 2. Plan the article before writing into it

Sketch the structure first — intro, sections, comparison, FAQ, closing — and map
each part onto a widget role from the catalog. `max_length` is a hint for how
much text the block was designed to hold — nothing on this path enforces it. A
value you overflow is written in full and can break the block's layout, with no
error or warning back to you. Keep copy inside `max_length` yourself.

### 3. Write and place

`add_content_widget` inserts one widget with `content` keyed by role — either
`{role: value}` for a simple widget or `{"items": [...]}` for a repeatable one.
Optional `position` places it (1-based); optional `image_url[]` fills image slots.

`compose_content` writes the **whole** description in one call when it should not
grow block by block. Its default `mode` is `replace`, which empties the entity
first — that call writes nothing until you repeat it with `confirmed: true`, and
you should say what would be deleted before you do.

**Repeatable widgets need genuinely DISTINCT items.** Three benefit tiles that
say the same thing three ways is the most common failure of AI-written text;
if you cannot think of a third distinct point, use a widget with two slots.

`edit_content_widget` rewrites an existing widget by role. It is
structure-preserving, and with `lang` it touches only that language, so
translations and photos survive an edit. Note the difference: `add_content_widget`
and `compose_content` store the text in **every** language of the entity, and
`edit_content_widget` with a `lang` is how one language gets its own wording.

**Check `content_sanitized` in the response.** Active HTML — `script`, `iframe`,
`img`, inline event attributes — is stripped before storage. When the flag is
true, what got stored differs from what you sent; do not tell the user an image
or an embed made it in without looking.

Call `get_content_widget` before any edit — it returns a numbered snapshot with
`position` and `widget_instance_id`, which is what the edit and move tools
address. Editing from memory of an earlier snapshot is how you overwrite the
wrong widget.

### 4. Arrange and enrich

- `move_content_widget` reorders (1-based positions).
- `remove_content_widget` deletes — destructive, and its first call only returns
  a preview; it removes nothing until you repeat it with `confirmed: true` **and
  the `confirm_token` the preview handed back**. That token is a fingerprint of
  the run: if the entity changed in between, it no longer matches and the call
  is refused. That is not an error to work around — it means what you were shown
  is no longer what would happen, so preview again. It also takes
  `entity_id_list` for up to ten entities behind one preview.
- `search_content` finds the entities whose description contains a phrase, which
  `list_product` does not search. Use it before a rewrite: "which products still
  mention the old price".
- `get_platform_content` is what the shop itself says about the entity today.
  **Read it before rewriting anything** — it usually holds facts that exist
  nowhere else (dimensions, materials, what is in the box), and a rewrite that
  ignores it quietly drops them. Carry its facts over; do not copy its wording.
- `copy_content_widget` puts one entity's widgets onto others of the same type —
  up to 25 products or 50 categories / articles in one call.
- `update_content_meta` writes the name and the SEO fields. **It will refuse
  fields the shop platform is master of**, and say which export mode would allow
  them; that is not a bug, it is the tool declining to write a value the next
  import would overwrite. In practice most e-shops cannot write SEO fields here.
- `add_content_product_carousel` inserts **real products** from the shop,
  addressed by `product_id[]` and/or `product_url[]` (at most 10 each). Prefer
  this over writing product names into prose — the carousel stays correct when
  prices and stock change.
- `list_content_image` is the entity's picture library and `import_content_image`
  copies one in from a public https address; `set_content_widget_image` fills the
  slots, and `convert_content_widget` turns a widget into another template of the
  catalog, keeping the texts verbatim (pass the target `widget_id` — it never
  picks a layout for you).

### 5. Photos, icons, and what they cost

**Look before you buy.** In this order:

1. `list_content_image` — what this entity already holds.
2. `list_content_media` — the eshop's whole media library, the folders and files
   the merchant uploaded themselves. A picture they already chose and paid for
   beats a stock photo, and beats a generated one by a mile.
3. `import_content_image` — pull one in from a public https URL if the user
   gives you an address.
4. `image_source: "stock"` — Pexels, free.
5. `image_source: "ai"` — **paid**, and the last resort.

`image_source` of `stock`, `uploaded` or `image_bank` is **free**.

`image_source: "ai"` is **paid**. The tool will not spend anything on the first
call — it returns `confirm_required: true` with `photo_count` and `cost`. Relay
that price to the user in their currency terms, wait for an explicit yes, and
only then repeat the same call with `cost_confirmed: true`. There is no separate
confirm tool, and `confirm_required` is not an error.

**Icons are their own thing.** Widgets that show a row of benefits or parameters
have icon slots (`icon_slot` in the catalog), filled with `icon_url`.

- `list_icon` is the ready-made set — hundreds of icons in named categories,
  free. **Always look here first.**
- `generate_content_icon` draws new ones for subjects the set does not have.
  It is **paid**, quotes first like AI photos, and runs in the background:
  it hands back a token and `get_content_icon_status` says when the files are
  there and what they cost.
- **There is no cache.** Every subject is drawn and billed again, so asking for
  a subject the ready-made set already covers is paying twice for the same
  picture. A duplicate subject inside one call is refused for the same reason.
- Derive the subjects from the text the icons will stand next to — write the
  benefits first, then name their icons. Subjects invented before the copy
  exists produce icons that do not match it.

Check `get_credit` before offering any paid option, so you do not quote a price
the balance cannot cover.

### 6. Review

`render_content_html` renders the description and returns a quality report:
`byte_size`, empty slots and warnings. Fix empty slots before telling the user
you are done — an unfilled role renders as a gap on the live page. A Shoptet
**product** over 65 000 bytes takes no further content at all, and the report is
where you see that coming.

### 7. Undo

Every write is versioned, including the ones made in the Pobo editor.
`get_content_history` lists the versions of one entity, `list_content_batch` the
recent runs across the e-shop, and `revert_content` puts either back — one entity
to a `version_id`, or a whole run by its `batch_id`. A revert saves the current
state first and hands back its own `batch_id`, so it can itself be reverted.
This is the honest answer to "undo that" — never re-type the old text from
memory when a version holds it.

## Workflow — server-side generation

`generate_blog_article` takes the brief and the shape of the article: `brief`,
`normpages` (length, 1-10), `structure_mode` + `design_id`, `photo_source`,
`image_count` (0-10), `product_id`/`product_url` (at most 10 each),
`inspiration_url` (at most 3), `tone`, `remove_exist_widget`, `lang`.

- **`photo_source: "ai"` requires `image_count`** — the quote has to be
  deterministic — and `cost_confirmed: true`. The quote is `1 + image_count × 2`.
- It returns a `job_id`. Poll `get_blog_generate_status`; the `step` values are
  `preparing` → `writing` → `selecting_photos` → `composing` → `done`. Report
  progress in those terms rather than repeating "still running".
- When it finishes, review it like your own work: `get_content_widget` and
  `render_content_html`, and offer to fix weak sections with
  `edit_content_widget` — which is free.

## Writing quality

The widgets give the article its shape; you still have to make it worth reading.

- Open with the reader's problem, not with the brand.
- One idea per widget. A section that needs "and also" wants to be two.
- Concrete beats clever: sizes, materials, numbers, real use cases.
- Do not invent facts about products. If you need a specification you do not
  have, ask the user or leave it out — a wrong dimension in a buying guide is
  worse than a missing one.
- Match the e-shop's voice. If the user has a tone in mind, ask for one existing
  article as a reference before writing a long piece.

## Error handling

- `Eshop not found.` / `Blog not found.` — wrong id, or it belongs to a
  different e-shop; re-run `list_eshop` / `list_blog`.
- Widget not in the catalog — you used a template this e-shop does not have;
  re-run `get_content_widget_catalog`.
- `Tool not found` on `add_blog_widget`, `get_blog_content`, `review_blog`,
  `update_blog_title` or any other `*_blog_*` widget tool — those were retired on
  2026-09-15. The replacement is the same operation with `entity_type: "blog"`.
- Validation error on `content` roles — the role names must match the catalog
  exactly, and repeatable widgets take `items`, not a flat map.
- `confirm_required: true` — not an error; see step 5.
- Insufficient credits on a paid call — report the balance from `get_credit` and
  offer the free path (you write it, free photo sources) instead.
- 401 — see Prerequisites & auth above.

## Out of scope

- Publishing / visibility scheduling and platform sync — done in the Pobo admin.
- Product descriptions — that is the generation prompt workflow
  (`manage-prompts`), not blog authoring.
- Styling of the rendered widgets — that is `style-widgets`.
- The in-app Pobo chat assistant — you are the agent here; there is no reason to
  call another one.
