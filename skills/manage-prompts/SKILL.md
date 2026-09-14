---
name: manage-prompts
description: Manage and test AI generation prompt profiles for Pobo Page Builder product descriptions. Use when the user wants to create, edit, review, try out, or organize the prompts (the brief) Pobo uses to generate product content — including per-widget instructions that pin specific requirements to specific widgets of a design template, and free dry-run previews against real products before a prompt is saved. Typical asks: "set up the generation prompt", "edit the description brief", "try out that prompt", "break down the client's requirements into the prompt", "what's in our prompt". Uses the `pobo` MCP server tools.
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

`preview_generation` dry-runs the prompt against 1–3 real products **without
writing anything and without spending credits** — no widgets, no images, no
generation history row. This is how you find out whether an edit actually
produced better copy; guessing from the prompt text alone does not.

- Pass `eshop_id`, `design_id`, `product_id` (1–3, from `find_product` or
  `list_product`) and the `prompt` text you are testing. Optional generation
  settings mirror the admin: `paragraph_length`, `generate_seo_meta`,
  `generate_entity_name`, `use_ai_profile`, `use_serp_context`,
  `use_web_research`, `search_web`, `search_model`.
- It returns one token per product. Poll them with `get_preview_status` —
  status `pending` until the queue picks the job up, then `complete` with the
  rendered `html` and any generated SEO fields. Tokens expire after 30 minutes.
- Poll at a sensible pace: a preview runs the full research + generation
  pipeline, so it takes tens of seconds, not one.
- Pick products that actually exercise the prompt — if the brief is about
  material and sizing, preview a product that has both.

Loop prompt → preview → prompt as many times as it takes, and call
`update_prompt` / `set_widget_prompt` only once the output is right. Note that
`preview_generation` takes the prompt as text, so you can test an edit that is
not saved anywhere yet.

### 6. Report

Summarize for the user:

- what was created/changed (profile name, linked design, per-widget coverage:
  "5 of 8 widgets have an explicit instruction"),
- if the prompt text was overwritten, note that previous versions remain
  available in the history,
- what the preview looked like, if you ran one (previews cost nothing, so
  there is no reason to report them as an expense),
- that the real generation is started in the Pobo Page Builder admin (product
  grid → select products → pick this profile) — previewing is available here,
  triggering the real run deliberately is not.

### Rollback

- Prompt text: fetch the previous version via `get_prompt_history` and write it
  back with `update_prompt`.
- Per-widget instructions: re-send the previous set with `set_widget_prompt`
  (keep a copy of the state you read in step 1). They have no server-side
  history.
- Whole profile: `delete_prompt` removes it from the e-shop (confirm with the
  user first — this also deletes its per-widget instructions).

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
  Pobo has AI generation switched off (outage or maintenance). Previews are
  blocked for everyone; prompt edits still work. Tell the user to try the
  preview later rather than retrying in a loop.
- `Design does not belong to this eshop.` — preview needs a design the e-shop
  can use; re-run `list_design`.
- `Product does not belong to this eshop.` — comes back per product inside an
  otherwise successful preview call, not as a whole-call error; the other
  products still queued.
- `not_found` from `get_preview_status` — the token expired (30 min), never
  existed, or belongs to a different e-shop. Re-run `preview_generation`.
- 401 — see Prerequisites & auth above.

## Out of scope

- Triggering the real generation — previews are here, the real run is started
  in the Pobo admin. That is deliberate: unlike a preview it spends credits and
  overwrites live content.
- Checking credits before a preview — previews are free. (`get_credit` exists
  for the paid tools, e.g. blog generation.)
- Generation settings other than name/prompt/design/icon category (section
  counts, image flow, translations, …) — managed in the Pobo admin UI.
- Pobo gallery prompt templates — read-only, not accessible here.
- Styling (that is `style-widgets`) and labeling (that is `label-products`).
