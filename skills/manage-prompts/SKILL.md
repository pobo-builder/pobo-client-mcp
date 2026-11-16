---
name: manage-prompts
description: Manage and test AI generation prompt profiles for Pobo Page Builder product descriptions. Use when the user wants to create, edit, review, try out, or organize the prompts (the brief) Pobo uses to generate product content — including per-widget instructions that pin specific requirements to specific widgets of a design template, paid try-outs on 1–3 real products that write nothing to them, and starting the real generation when asked. Typical asks: "set up the generation prompt", "edit the description brief", "try out that prompt", "break down the client's requirements into the prompt", "what's in our prompt", "generate descriptions for these products". Uses the `pobo` MCP server tools.
---

# Manage Pobo Page Builder AI generation prompts

You manage the merchant's custom prompt profiles — the instructions Pobo Page
Builder uses when generating product descriptions. You can create and edit
profiles, link them to a design (widget template), and decompose the client's
requirements into deterministic per-widget instructions. All logic runs on the
Pobo Page Builder backend.

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

## What a prompt profile is

A profile has:

- **name** — shown in the Pobo admin when the merchant picks a profile for
  generation,
- **prompt** — free-text general instructions (tone, brand voice, language,
  what to emphasize) applied to the whole product description,
- optional **design** — the widget template the content is generated into,
- optional **per-widget instructions** — one instruction pinned to one widget
  of the design; it overrides the general prompt for that widget.

Only *custom* profiles are visible and manageable here — Pobo's own gallery
templates are read-only and never appear in these tools.

## Workflow

### 1. Pick the e-shop and review what exists

Call `list_eshop`, then `list_prompt`. Before changing anything, read the
current state with `get_prompt` — and if you are about to overwrite the prompt
text, check `get_prompt_history` first so you can tell the user what was there
before (the history keeps every previous version of the text; per-widget
instructions are NOT versioned).

### 2. Choose a design (when the profile should generate into a template)

Call `list_design` — it returns the public Pobo templates plus the e-shop's own
custom templates. Call `list_design_widget` on the chosen design to see its
widgets in display order (`id`, `position`, `name`). The `id` values are what
per-widget instructions target.

A profile without a design is valid — it just cannot have per-widget
instructions.

### 3. Create or update the profile

- `create_prompt` with `eshop_id`, `name` (5–255 chars), `prompt`
  (20–1,000,000 chars), optional `design_id` and `icon_category_id`.
- `update_prompt` is **partial** — send only the fields to change. Changing
  `design_id` **clears existing per-widget instructions** (they referenced the
  old design's widgets); the response signals this with
  `widget_prompt_cleared: true` — recreate them in the next step.

Write the prompt text in the language the merchant generates in (usually
Czech). Put durable, general guidance in the profile text; put widget-specific
requirements into per-widget instructions instead of positional prose.

### 4. Set per-widget instructions

Prefer per-widget instructions over positional wording ("in the first section
write…") — positions shift when the design changes, per-widget instructions
target one widget deterministically and override the general prompt for it.

Call `set_widget_prompt` with the **complete** new set (up to 30 items, each
prompt up to 50,000 chars):

- the call **replaces all** existing per-widget instructions — always build the
  full list from the current `get_prompt` state plus your changes, never send
  just the delta;
- an empty array `[]` deliberately clears them all;
- every `design_widget_id` must belong to the profile's design (see
  `list_design_widget`), and the profile must have a `design_id`.

Typical use: the client's brief says "mention the stainless steel construction,
add a sizing FAQ, and keep benefits under 5 words" — map each requirement onto
the matching widget of the design and write one focused instruction per widget.

### 5. Test the prompt before you save it

`preview_generation` tries the prompt on 1–3 real products **exactly like a real
generation** — text, SEO, AI photos, icons — but writes **nothing** to the
products. This is how you find out whether an edit actually produced better copy;
guessing from the prompt text alone does not.

**It is paid, like a generation.** The first call only quotes the most it can
cost and starts nothing. Tell the user the price, wait for an explicit yes, then
repeat the call with the same settings plus `expected_credit` = the quote and
`cost_confirmed: true`. Check `get_credit` first so you do not offer what the
balance cannot cover. Only one try-out per e-shop runs at a time.

- Pass `eshop_id`, `product_id` (1–3, from `find_product` or `list_product`) and
  either `prompt_id` (the saved profile) or the `prompt` text you are testing —
  `prompt` wins over the saved text, so you can test an edit that is not saved
  anywhere yet. `design_id` overrides the template; without it the template of
  `prompt_id` is used, without both a basic Pobo layout.
- With `prompt_id`, the profile's **saved per-widget instructions are applied
  too**, as in the real run. Instructions you have not saved yet are not — save
  them with `set_widget_prompt` first, or judge their wording by re-reading it.
- Optional settings mirror the admin: `paragraph_length`, `generate_seo_meta`,
  `generate_entity_name`, `generate_image_variation`, `use_ai_profile`; research
  (`use_serp_context`, `use_web_research`, `search_web`, `search_model`) only with
  a template, and each research source is paid on its own.
- Poll the token with `get_preview_status`: per product `pending`, `running`,
  `complete` (rendered HTML, SEO fields, name, the credits it cost) or `failed`.
  A preview runs the full pipeline, so it takes tens of seconds, not one. Try-outs
  are kept 30 days.
- Pick products that actually exercise the prompt — if the brief is about
  material and sizing, preview a product that has both. Start with one product;
  add more only when the first one looks right.

Loop prompt → preview → prompt, but count the cost: each round is paid. Call
`update_prompt` / `set_widget_prompt` once the output is right.

### 6. Run the real generation (only when asked)

`start_generation` generates descriptions for up to 500 products with a prompt
and **writes them into the products** (as drafts, waiting for approval). It needs
the PROFI plan or higher; a `custom` prompt or one with a design needs SAAS.

- First call with `product_id` and `prompt_id` (omit it for the one used last):
  it prepares the batch and returns `generation_id`, the price and the products
  that lack data for a good description. Report all three.
- Second call with `generation_id` and `cost_confirmed: true` — only after the
  user agreed to the price.
- Follow it with `get_generation_status` (`wait` up to 60 s): generated, running,
  skipped. `list_generation` / `get_generation` show one product's run.
- Category descriptions: `entity_type: "category"` with `category_id` and a prompt
  whose template is for categories (`list_prompt`: `for_category: true`). Products
  never use such a prompt.

Do not start a generation just to test a prompt — that is what the preview is for.
The generated products are drafts; to put them live see `publish-content`.

### 7. Report

Summarize for the user:

- what was created/changed (profile name, linked design, per-widget coverage:
  "5 of 8 widgets have an explicit instruction"),
- if the prompt text was overwritten, note that previous versions remain
  available in the history,
- what the preview looked like and what it cost, if you ran one,
- if you started a generation: how many products, the price, and that the results
  wait for approval as drafts.

### Rollback

- Prompt text: fetch the previous version via `get_prompt_history` and write it
  back with `update_prompt`.
- Per-widget instructions: re-send the previous set with `set_widget_prompt`
  (keep a copy of the state you read in step 1). They have no server-side
  history.
- Whole profile: `delete_prompt` removes it from the e-shop (confirm with the
  user first). If no other e-shop uses the same profile it is deleted
  outright, together with its per-widget instructions; the response's `action`
  field says `deleted` vs `detached`.

## Error handling

All tools return human-readable MCP errors — read them and react. Common ones:

- `Eshop not found.` — wrong `eshop_id`; re-run `list_eshop`.
- `Prompt not found.` — the profile does not belong to this e-shop, or it is a
  read-only gallery template; re-run `list_prompt`.
- `Design not found.` / validation error on `design_id` — the design is not
  public and not bound to this e-shop; re-run `list_design`.
- `The prompt has no design — set design_id first…` — link a design via
  `update_prompt` before `set_widget_prompt`.
- Validation error on `widget_prompt.*.design_widget_id` — the widget belongs
  to a different design; re-check with `list_design_widget`.
- `Nothing to update…` — `update_prompt` needs at least one field to change.
- `AI content generation is temporarily unavailable. Try again later.` —
  Pobo has AI generation switched off (outage or maintenance). Previews and
  generations are blocked for everyone; prompt edits still work. Tell the user to
  try later rather than retrying in a loop.
- A refusal that names the plan — AI generation needs PROFI or higher, an
  extended template (custom prompt or a design) SAAS. Tell the user; do not try to
  get around it with another prompt.
- `Design does not belong to this eshop.` — preview needs a design the e-shop
  can use; re-run `list_design`.
- `Product does not belong to this eshop.` — comes back per product inside an
  otherwise successful preview call, not as a whole-call error; the other
  products still queued.
- `not_found` from `get_preview_status` — the token never existed, is older
  than the 30 days try-outs are kept, or belongs to a different e-shop.
- 401 — see Prerequisites & auth above.

## Out of scope

- Uploading generated content to the e-shop — `publish-content`.
- Generation settings other than name/prompt/design/icon category (section
  counts, image flow, translations, …) — managed in the Pobo admin UI.
- Pobo gallery prompt templates — read-only, not accessible here.
- Styling (that is `style-widgets`) and labeling (that is `label-products`).
