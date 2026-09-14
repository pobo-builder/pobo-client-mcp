---
name: label-products
description: Label Pobo Page Builder products from a client-supplied list of identifiers (EANs, product codes, product URLs, or product names). Use when the user wants to tag, label, mark, or organize products in Pobo Page Builder — typically a copywriter who received a messy product list (xlsx, PDF, plain text) from a client and needs those products labeled so they can be filtered in the Pobo admin grid. Uses the `pobo` MCP server tools.
---

# Label Pobo Page Builder products from a client-supplied list

You resolve a client-supplied list of product identifiers to Pobo Page Builder
products and attach a label to them, so the user can filter the batch in the Pobo
admin grid (e.g. to run content generation on it). All logic runs on the Pobo
Page Builder backend.

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

## Workflow

### 1. Pick the e-shop

Call `list_eshop`. If exactly one e-shop is returned, use it. If multiple, match
against the client's domain when the input contains product URLs (e.g. URLs on
`www.sportrec.cz` → the e-shop with that URL), otherwise ask the user.

### 2. Collect the identifiers

The input arrives in any shape — xlsx, PDF, e-mail text, a pasted table. Extract
one identifier per product, prefer them in this order of reliability:

1. **product URL** (most reliable — exact match regardless of http/https, www,
   trailing slash)
2. **EAN** (8 or 13 digits)
3. **product code**
4. **product name** (least reliable — names are not unique)

Deduplicate. Do not reformat or "clean" URLs beyond trimming whitespace — the
server normalizes them itself.

### 3. Resolve with `find_product`

Send the identifiers in **one batch call** (up to 100 per call; chunk larger
lists): `find_product` with `eshop_id` and `identifier: [...]`. Leave `type` on
auto-detection unless the whole list is verifiably one kind. Each identifier
comes back as:

- `matched` — collect `product.id`
- `ambiguous` — up to 5 candidates; **present them to the user and let them
  choose, never silently take the first one**
- `not_found` — collect for the final report

Verify the matches make sense: skim the returned product names against the
client's list and flag anything suspicious to the user.

### 4. Pick or create the label

Call `list_label` first — the user may already have one. If the user named a
label, use it; otherwise propose a descriptive name (client/brand + purpose +
month, e.g. "Sportrec update 07/2026") and confirm with the user. Create it with
`create_label` — it is idempotent by name (`action: created | existing`), so a
retry or an admin-created label of the same name never duplicates.

### 5. Assign

Call `assign_product_label` with `eshop_id`, `label_id` and the collected
`product_id` array (up to 500 per call; chunk larger sets). The call is
idempotent — re-attaching already labeled products is a no-op. It is also
strict: a single unknown product id fails the whole call with
`Product not found: 1, 2.` — remove the listed ids and report them, do not
guess replacements.

### 6. Report

Summarize for the user:

- how many products got the label (and the label name),
- the `ambiguous` items and what was chosen,
- the `not_found` identifiers — ask the user to fix or complete those inputs
  (typo in URL, product not imported into Pobo yet, …),
- that the batch is now filterable by this label in the Pobo Page Builder admin
  product grid — and by you via `list_product` (`label_id` + grid filters like
  `waiting_for_approval`; see the `confirm-products` skill for the full list).

### Rollback

To undo, call `assign_product_label` with the same `label_id` and `product_id`
list and `action: "detach"`. The label itself stays (harmless); the user can also
manage labels in the Pobo admin. Confirm with the user before detaching.

## Error handling

All tools return human-readable MCP errors — read them and react. Common ones:

- `Eshop not found.` — wrong `eshop_id`; re-run `list_eshop`.
- `Label not found.` — the label does not belong to this e-shop; re-run `list_label`.
- `Product not found: {ids}.` — those ids do not belong to this e-shop (or were
  deleted); drop them from the batch and report.
- Validation errors on `identifier` / `product_id` — respect the caps
  (100 identifiers, 500 products per call) and chunk.
- 401 — see Prerequisites & auth above.

## Out of scope

- Creating or editing products/content — labeling only.
- Deleting labels (the admin UI covers that).
- Styling — that is the `style-widgets` skill.
- Setting up the generation prompt for the labeled batch — that is the
  `manage-prompts` skill (a natural next step once the batch is labeled).
- Switching product status (draft ⇄ ready) after the content is approved —
  that is the `confirm-products` skill.
