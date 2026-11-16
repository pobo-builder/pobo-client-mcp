---
name: publish-content
description: Check finished Pobo Page Builder product descriptions and upload them to the e-shop, so customers see them. Use when the user wants to publish, upload, push, send live or "put on the eshop" product descriptions written in Pobo — after writing them, after a generation, or after the client approved a batch — or asks why an upload was refused or what happened to it. Uses the `pobo` MCP server tools.
---

# Check and upload product descriptions to the e-shop

Content written in Pobo — by you, by a generation or by the merchant — stays in
Pobo until it is uploaded. This skill takes a batch of products through the
quality check and uploads the ones that passed. All logic runs on the Pobo
backend.

Communicate with the user in their language (typically Czech).

## Prerequisites & auth

The `pobo` MCP server is connected once via OAuth — the user runs
`claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
and logs in with their Pobo Page Builder account in the browser. If the `pobo`
tools are unavailable, or MCP calls fail with **401 / unauthorized**, tell the user
to connect it (or run `/mcp` and sign in again when the connection expired). Never
ask for credentials in the conversation.

Every tool refuses an e-shop where the merchant has not enabled Pobo LLM;
`list_eshop` shows `pobo_llm_enabled` and the `pobo_llm_setting_url` where it is
switched on.

## What an upload does — say it before you do it

`push_product` **overwrites what the e-shop shows today** for those products.
That is the point of it, but it is also the one step in Pobo that customers see
immediately. Before the first upload of a batch, tell the user how many products
go live and get an explicit yes. Never upload "just to try it".

Only products are uploaded this way. Categories and blog articles are exported by
the merchant from the Pobo administration.

## Workflow

### 1. Pick the e-shop and collect the products

`list_eshop`, then the product ids:

- you just wrote them (`write-blog`, `design-description`) — reuse the ids;
- a generation finished (`manage-prompts`) — the ids you generated;
- the client approved a batch — `list_product` with the label and
  `filter: "waiting_for_approval"` (see `confirm-products`), paging through
  `total`.

`push_product` takes 50 products per call; chunk larger batches.

### 2. Check every product with `check_content`

`check_content` is the quality gate, up to 10 products per call
(`entity_type: "product"`, `entity_id_list`). **It costs 2 credits per product**
(the same text over the same facts again is free), so for a large batch tell the
user the price first and check `get_credit`.

It reports two kinds of issues:

- **Rules** — no widget, a widget without text, an empty picture or icon slot
  (a broken image on the shop), the platform size limit, phrases the AI profile
  forbids, placeholder text.
- **Review** (for products that passed the rules) — `fact` = a concrete claim
  that neither the platform data, the AI profile nor the web research supports;
  `spelling`; `filler` = a sentence that tells the shopper nothing. Each quotes
  the words and suggests a fix.

`review_unavailable` means the model could not be reached — try again later,
do not upload around it.

### 3. Fix what failed, then check again

- Rules, spelling, filler — fix the text with `edit_content_widget` (or the
  other content tools, see `write-blog`), then run `check_content` again. The
  upload accepts only text that passed **in its current form**: any edit after
  the check means checking again.
- `fact` — do not just delete it and do not wave it through. Ask the user
  whether the claim is true:
  - true for this product only → the user confirms; you may upload with
    `acknowledge_fact: true` and a `reason` saying who confirmed what (step 5);
  - true for the whole e-shop (free delivery, a certification, origin) → it
    belongs in the AI profile (`update_eshop_ai_profile`), then check again;
  - not true → rewrite or remove the sentence, then check again.

Never set `acknowledge_fact` on your own judgement.

### 4. Approve the products

`push_product` skips products waiting for approval (`not_approved`). If the
products are drafts, switch them to ready with `set_product_status`
(`status: "ready"`) — but only once the user (or their client) approved the
content. See `confirm-products`.

### 5. Upload with `push_product`

Call `push_product` with `eshop_id`, `product_id` (50 at most) and a `reason`
— one sentence the merchant will read in the overview of agent changes ("the
client approved the winter tyre descriptions"). With `acknowledge_fact: true` the
`reason` is required.

The answer:

- `token` — for `get_push_status`;
- `refused` — products that failed `check_content`, were never checked, or
  changed since (`not_reviewed`), each with its issues. They are **not** uploaded;
  the rest of the batch is;
- `acknowledged` — fact issues let through with `acknowledge_fact`.

### 6. Follow it with `get_push_status`

The upload runs in the background. Poll `get_push_status` with the token until
`status` is `complete` or `failed`. Per product:

- `ok` — on the e-shop. `untranslated_lang` lists languages whose description
  stayed out because it is not translated — the e-shop keeps its previous text
  there; translate them (`fix-translations`) and upload again;
- `failed` — the platform refused it; the platform message says why;
- `skipped` — `no_content` (nothing in Pobo) or `not_approved` (still a draft);
  nothing broke;
- `not_found` — not a product of this e-shop.

A token older than 30 minutes comes back as `not_found`.

### 7. Report

How many products went live, which were refused by the check (with the issue in
one line each), which the platform refused, which were skipped and why, and the
languages that stayed untranslated. Offer to verify one live page with
`fetch_eshop_page` or `screenshot_eshop_page`.

## When the content is still not on the e-shop

- `get_push_status` said `ok` but the page shows the old text — the e-shop may
  cache pages; check again with `fetch_eshop_page` after a while.
- An upload started from the administration — `get_export_status` shows the
  merchant's exports; a failed export is never retried.
- New products from the platform are missing in Pobo — that is the other
  direction: `get_import_status` / `start_import` (see `diagnose-content`).

## Out of scope

- Writing or changing the content itself — `write-blog`, `design-description`.
- Uploading categories and blog articles — the merchant does that in the
  administration.
- Checking the whole catalogue against the merchant's rules — that is
  `check-content`; `check_content` here is for a few products before upload.
