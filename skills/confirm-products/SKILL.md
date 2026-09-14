---
name: confirm-products
description: Switch the status of Pobo Page Builder products between draft (waiting for approval) and ready (approved for export). Use when the user wants to approve, confirm, publish-approve, or return products for rework — typically a copywriter who finished new content and needs the batch marked ready for export, or wants approved products sent back to draft. Uses the `pobo` MCP server tools.
---

# Confirm Pobo Page Builder products (draft ⇄ ready)

You switch the status of a batch of Pobo Page Builder products between **draft**
(content waiting for approval) and **ready** (approved, prepared for export to
the e-shop platform). All logic runs on the Pobo Page Builder backend.

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

## Product statuses

Every product in Pobo Page Builder has exactly one status:

| Status | Meaning | Switchable by this skill |
|--------|---------|--------------------------|
| `draft` | Content waiting for approval | ✓ |
| `ready` | Approved, prepared for export to the e-shop | ✓ |
| `review` | In an editorial review flow | ✗ — owned by other flows |
| `generate` | AI generation in progress | ✗ — owned by other flows |

`set_product_status` switches **only between `draft` and `ready`, in both
directions**. Any other target status is rejected by validation, and a product
currently in `review` or `generate` is not switchable — never try to work
around that, those states belong to other pipelines.

## Workflow

### 1. Pick the e-shop

Call `list_eshop`. If exactly one e-shop is returned, use it. If multiple, match
against the client's domain when the input contains product URLs, otherwise ask
the user.

### 2. Collect the product ids

Three common sources:

- **A grid filter** — `list_product` filters products the same way the Pobo
  admin grid does (see the filtering section below). The typical ask "mark
  everything with label X that waits for approval as ready" is one call:
  `list_product` with `label_id` and `filter: "waiting_for_approval"`.
  Paginate with `page` until you have all `total` items.
- **Continuation of the labeling workflow** (`label-products` skill) — you
  already hold the matched `product.id` list; reuse it.
- **A fresh client list** (EANs, codes, URLs, names) — resolve it with one
  batch `find_product` call exactly as described in the `label-products` skill
  (`matched` → collect ids, `ambiguous` → let the user choose, `not_found` →
  report).

### 3. Confirm the direction with the user

Make sure you know which way to switch and that the user really wants it:

- `draft` → `ready`: the content was approved, the batch should be prepared
  for export.
- `ready` → `draft`: the content goes back for rework and must not be
  exported meanwhile.

State the product count and the direction, and get an explicit confirmation
before writing.

### 4. Switch with `set_product_status`

Call `set_product_status` with `eshop_id`, the `product_id` array (up to 500
per call; chunk larger sets) and `status: "ready"` or `status: "draft"`. The
call is idempotent — setting the current status again is a no-op. It is also
strict, with no partial writes:

- a single foreign/unknown id fails the whole call with
  `Product not found: 1, 2.` — remove the listed ids and report them,
- a product currently in `review`/`generate` fails the whole call with
  `Product not switchable (current status must be ready or draft): 1, 2.` —
  take the listed ids out of the batch, retry with the rest, and report the
  skipped ones to the user (they can be switched later, once their pipeline
  state finishes).

Response: `{"status": "ready", "product_count": 12}`.

### 5. Report

Summarize for the user:

- how many products were switched and to which status,
- which ids were skipped as not switchable (`review`/`generate`) or not found,
- that the batch is now filterable by status in the Pobo Page Builder admin
  product grid (see below), and — for `ready` — prepared for the next export.

## Filtering products (list_product = the admin grid filters)

`list_product` gives you the same filters the user sees in the Pobo admin
product grid — what you list is exactly what they see there. Combinable
parameters. The last column is the literal label as it appears in the
**Czech** Pobo admin UI — keep it in Czech so you can match what the user
sees on screen there; the English meaning follows in parentheses:

| Parameter | Values | Label in the Czech admin UI |
|-----------|--------|------------------------------|
| `filter` | `all` (default) | Vše (all) |
| | `without_description` | Bez popisku (no Pobo content yet) |
| | `edited` | Upravené v Pobo (edited in Pobo) |
| | `favourite` | Oblíbené (favourites) |
| | `waiting_for_approval` | Čeká na schválení (waiting for approval; status `draft`) |
| | `recently_edited` | ordering: Naposledy upraveno (most recently edited) |
| | `most_visited` | ordering: Nejnavštěvovanější (most visited) |
| | `most_added_to_cart` | ordering: Nejčastěji v košíku (most added to cart) |
| `query` | string | full-text search (name, code, EAN) |
| `label_id` | array of label ids | label filter |
| `category_id` | array of category ids | Kategorie (category) |
| `brand_id` | array of brand ids | Značka (brand) |
| `is_visible` | `all` / `visible` / `hidden` | Viditelnost (visibility) |
| `limit`, `page` | max 100 per page, 1-based | pagination |

When the user names a category ("drafts in Massage Tools"), resolve the name
to an id first with `list_category` (`query` full-text search; returns id,
name, url, status, has_content, product_count and takes the same `filter`
values), then pass the id to `list_product` via `category_id`. If the name
matches several categories, present them and let the user choose.

Response: `{"product": [{id, name, code, url, status, is_visible,
is_favourite, has_content}], "total", "page", "limit"}`. Always page through
the whole result (`page` until `total` is covered) before switching, and tell
the user the `total` you are about to switch.

The typical copywriting loop: label the client's batch (`label-products`) →
generate/edit content → the client approves → `list_product` (label +
`waiting_for_approval`) → this skill switches the batch to `ready` → export
picks it up. The same filters are available to the user in the admin grid,
so they can verify the result visually.

## Error handling

All tools return human-readable MCP errors — read them and react. Common ones:

- `Eshop not found.` — wrong `eshop_id`; re-run `list_eshop`.
- `Product not found: {ids}.` — those ids do not belong to this e-shop (or were
  deleted); drop them from the batch and report.
- `Product not switchable (current status must be ready or draft): {ids}.` —
  those products are in `review`/`generate`; drop them, retry the rest, report.
- Validation error on `status` — only `ready` and `draft` are accepted.
- Validation errors on `product_id` — respect the cap (500 per call) and chunk.
- 401 — see Prerequisites & auth above.

## Out of scope

- Setting `review` or `generate` — those states are owned by other flows.
- Running the export itself — exports are triggered from the Pobo admin.
- Creating or editing products/content — see the other skills.
- Labeling — that is the `label-products` skill (typically the step before
  this one).
