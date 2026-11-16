---
name: fix-translations
description: Find and fix missing, stale or wrong translations of Pobo Page Builder content — products, categories, blog articles and shared descriptions — and translate a whole e-shop into a new language. Use when the user asks what is not translated, why a language version is missing or wrong, wants to check the quality of translations, or wants everything translated into a language. Uses the `pobo` MCP server tools.
---

# Translations — find what is wrong, translate exactly that

Pobo tracks the translation state of every product, category, blog article and
shared description per language. On top of that, the AI translation check reads
translations against their main-language text and reports what is wrong in them.
You find the problem, then translate again exactly what needs it.

Communicate with the user in their language (typically Czech).

## Prerequisites & auth

The `pobo` MCP server is connected once via OAuth — the user runs
`claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
and logs in in the browser. If the tools are unavailable or calls fail with
**401**, tell the user to connect it (or run `/mcp` and sign in again). Every tool
refuses an e-shop where the merchant has not enabled Pobo LLM — `list_eshop`
shows where to switch it on.

## Two layers

| | What it knows | Cost |
|---|---|---|
| **Translation state** | missing, partial, or stale (the main text changed since it was translated) — per field and per widget | free, always there |
| **AI translation check** | text left in another language (`wrong_language`), `partial`, `missing_sentence`, `extra_sentence`, `translated_term` (a name that should stay untranslated) — with the quoted words | paid, PREMIUM plan and up |

`fix_translation` translates again whatever either layer flags. It is paid.

## Paid calls quote first

`start_translation_check` and `fix_translation` answer the first call with the
price and do nothing. Show it, wait for an explicit yes, then repeat the call
with the **same arguments** plus `expected_credit` = the quoted price and
`cost_confirmed: true`. `get_credit` shows the balance.

## Workflow

### 1. Where is something wrong?

`list_product`, `list_category` and `list_blog` take the admin grid's
translation filter:

- `translation: "attention"` — not translated, partly, out of date, or with an
  open finding of the AI check; `"ok"` — everything in order;
- `translation_lang` — one language, or `"any"` (default).

Every item in the answer carries its translation state per language. Report the
count from `total`.

### 2. What exactly is wrong with one item?

`get_translation_finding` (`entity_type`: `product`, `category`, `blog` or
`share`) — per language: state, what is missing (field, widget, kind) and the
open findings of the AI check with the translated words (`quote`) and the
main-language words they belong to (`source_quote`). Free.

### 3. Check the quality of translations (optional, PREMIUM and up)

When the user asks whether the translations are any good — not just whether they
exist — run the AI translation check:

1. `start_translation_check` — scope by `entity_type`, `lang` and `field`
   (all by default). `only_changed: true` skips units whose texts did not change
   since the last check (the quote stays the full price; the unsent part comes
   back). First call = quote; confirm as above.
2. `get_translation_check` with the `run_id` — progress, findings per kind, and
   `entity_with_finding` = **how many items still have an open finding**. That is
   the number to report; never take it from the totals of `list_product` or
   `list_category`, which count the whole catalogue.
3. Then back to step 1 with `translation: "attention"`.

Only one check per e-shop runs at a time.

### 4. Translate again — `fix_translation`

Per item and language it translates again the areas (name, short description,
SEO, description) that miss a translation or have an open finding, and closes
those findings.

- **A few items** — `entity_type` + `entity_id` (up to 500).
- **Everything missing** ("translate the whole e-shop into Slovak", "fix what the
  check found") — `all: true` and leave `entity_type` out: products, categories,
  articles and shared descriptions in one call, one price, one approval
  (`by_type` splits the quote). Each call takes the next 500 items per kind —
  **repeat it until `fix_count` is 0**. Items still being translated come back as
  `running_count`; wait, then call again.
- `lang` — only some languages; without it every translation language of the
  e-shop.
- **Stale translations** may hold the merchant's own edits. They are left alone
  unless the user explicitly asks for them — then `include_stale: true`. The quote
  reports `stale_count`, so you can ask.

The merchant's own rules for translations (terminology, how to address the
customer, what never to translate) are `translate_prompt` in
`get_eshop_ai_profile`. If the user corrects the same thing twice, offer to put it
there with `update_eshop_ai_profile`.

### 5. Report and hand over

What was translated, into which languages, how many items, the credits spent,
and what was left out (stale ones without `include_stale`, items still running).
Translated content is in Pobo; to get it onto the e-shop, upload the products
(`publish-content`) — `get_push_status` lists languages that stayed out because
they were not translated.

## Out of scope

- Checking texts against the merchant's rules (tone, claims, law) — that is
  `check-content`. A content fix only swaps quoted words; it does not translate.
- Writing new content in one language — `write-blog`, `design-description`.
