---
name: write-blog
description: Write and edit Pobo Page Builder blog articles — compose an article yourself from widget templates, rewrite existing sections, insert photos and real product carousels, or hand the whole thing to Pobo's server-side generator. Use when the user wants a blog post, magazine article, buying guide or landing text on their e-shop: "write an article about…", "edit that blog post", "add photos to the article", "insert products into the article". Uses the `pobo` MCP server tools.
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

`get_blog_widget_catalog` is the step people skip and then guess. It returns the
widget templates this e-shop actually has, and for each one its content roles —
the named slots you fill (`role`, `hint`, `max_length`, `image_slot`,
`item_count`). **You can only use widgets from this catalog**; anything else is
an error.

### 2. Plan the article before writing into it

Sketch the structure first — intro, sections, comparison, FAQ, closing — and map
each part onto a widget role from the catalog. `max_length` is a hint for how
much text the block was designed to hold — nothing on this path enforces it. A
value you overflow is written in full and can break the block's layout, with no
error or warning back to you. Keep copy inside `max_length` yourself.

### 3. Write and place

`add_blog_widget` inserts a widget with `content` keyed by role — either
`{role: value}` for a simple widget or `{"items": [...]}` for a repeatable one.
Optional `position` places it; optional `image_url[]` fills image slots.

**Repeatable widgets need genuinely DISTINCT items.** Three benefit tiles that
say the same thing three ways is the most common failure of AI-written articles;
if you cannot think of a third distinct point, use a widget with two slots.

`edit_blog_widget` rewrites an existing widget by role. It is
structure-preserving and only touches the target language, so translations and
photos survive an edit.

Call `get_blog_content` before any edit — it returns a numbered snapshot with
`position` and `widget_instance_id`, which is what the edit and move tools
address. Editing from memory of an earlier snapshot is how you overwrite the
wrong widget.

### 4. Arrange and enrich

- `move_blog_widget` reorders; it returns a fresh snapshot.
- `remove_blog_widget` deletes one — destructive, so confirm with the user first.
- `update_blog_title` overwrites the title and SEO title together.
- `add_blog_product_carousel` inserts **real products** from the shop, addressed
  by `product_id[]` and/or `product_url[]` (at most 10 each). Prefer this over
  writing product names into prose — the carousel stays correct when prices and
  stock change.
- `set_blog_widget_image` fills an image slot; `convert_blog_widget` turns a
  text widget into an image+text layout, keeping the texts verbatim.

### 5. Photos and what they cost

`image_source` of `stock`, `uploaded` or `image_bank` is **free**. Use these
first.

`image_source: "ai"` is **paid**. The tool will not spend anything on the first
call — it returns `confirm_required: true` with `photo_count` and `cost`. Relay
that price to the user in their currency terms, wait for an explicit yes, and
only then repeat the same call with `cost_confirmed: true`. There is no separate
confirm tool, and `confirm_required` is not an error.

Check `get_credit` before offering a paid option, so you do not quote a price
the balance cannot cover.

### 6. Review

`review_blog` renders the article and returns a quality report: `byte_size`,
empty slots and warnings. Fix empty slots before telling the user you are done —
an unfilled role renders as a gap on the live page.

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
- When it finishes, review it like your own work: `get_blog_content` and
  `review_blog`, and offer to fix weak sections with `edit_blog_widget` — which
  is free.

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
  re-run `get_blog_widget_catalog`.
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
