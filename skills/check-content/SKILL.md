---
name: check-content
description: Check the texts of a whole Pobo Page Builder catalogue (products, categories or blog articles) against the merchant's own rules — tone, forbidden claims, legal risk, SEO, missing photos — and fix what the check found. Use when the user wants to check, audit or review their product descriptions or texts in bulk, find health claims, superlatives, misleading or illegal wording, or fix the errors a check reported. Not for translations (fix-translations) and not for a few texts before upload (publish-content). Uses the `pobo` MCP server tools.
---

# Bulk content check and fixes

Three paid runs over the catalogue:

1. **Check** — AI reads every text in scope against the merchant's rules and
   reports each place that breaks one, with the exact quote, what is wrong, why it
   matters and a suggested fix.
2. **Fix** — rewrites only the quoted words of each finding and saves every fixed
   product as a draft. Nothing reaches the e-shop.
3. **Recheck** — checks the fixed texts again and says, finding by finding,
   whether it is resolved.

Communicate with the user in their language (typically Czech).

## Prerequisites & auth

The `pobo` MCP server is connected once via OAuth — the user runs
`claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
and logs in in the browser. If the tools are unavailable or calls fail with
**401**, tell the user to connect it (or run `/mcp` and sign in again). Every tool
refuses an e-shop where the merchant has not enabled Pobo LLM — `list_eshop`
shows where to switch it on.

## Money first

All three runs cost credits and all three work the same way: **the first call only
quotes the price and does nothing**. Show the user the price, wait for an explicit
yes, then repeat the call with the same arguments plus
`expected_credit` = the quoted price and `cost_confirmed: true`. Never send
`cost_confirmed` without that yes, and never change the scope between the quote
and the confirmed call — the price would no longer match.

`get_credit` shows the balance; if it does not cover the quote, say so instead of
starting.

Only one check and one fix run per e-shop at a time.

## Workflow

### 1. Agree on the rules

The check is only as good as its rules. Rules are plain sentences, one per line,
in the merchant's words.

- `list_content_check_preset` — tested ready-made rules grouped by what they
  protect: `geo` (AI assistant recommendations), `seo`, `law` (legal risk),
  `brand` (tone and trust), `data` (catalogue data such as photos). Each says
  why it matters and when it fits, and `industry` says which shops it suits.
  Offer the ones that fit the e-shop.
- A rule with `legal_pack` is checked against the official text of the law and
  its findings carry `legal_ref`; it is priced higher. A rule with `data_rule`
  (e.g. missing photo) is answered from catalogue data, free and exact. **Pass both
  kinds unchanged, each on its own line**, or they are not recognised.
- A rule with `needs_fact` compares texts with product parameters — start the
  check with `with_fact: true`.
- `with_ai_profile: true` adds the rules the AI profile already states
  (forbidden phrases, tone, emoji, delivery and return terms).
- `rule_severity` gives each line `high`, `medium` or `low`, in the same order.

### 2. Agree on the scope

- `entity_type` — `product`, `category` or `blog`; the whole catalogue of that
  kind is checked.
- `field` — which texts (name, short description, description, SEO). Omit for
  all of them. `platform_description` (the original platform text even when Pobo
  replaced it) only when the user asks for it.
- `lang` — languages; omit for the main language only. Every language of every
  entity is one item.
- `with_pobo_content: true` — only entities with a Pobo description.
- `category_id` — products only, just these categories (`list_category`).

### 3. Quote — `estimate_content_bulk_check`

Free, changes nothing: entities and items in scope and the most it costs
(`credit`). Pass `instruction` too — a legal rule changes the price, and
`credit_without_legal` shows the price without it. Tell the user: what will be
checked, how many items, the price, and that unread items come back.

### 4. Start — `start_content_bulk_check`

Same scope and `instruction`, plus `expected_credit` and `cost_confirmed: true`.
Credits are reserved at the start; what was not read comes back. It takes
minutes. Keep the returned `check_id`.

### 5. Follow and read — `get_content_bulk_check`

`wait` (up to 60 s) holds the answer until the check is done. Status:
`pending`, `running`, `finished`, `finished_partial` (some batches unread and not
charged), `failed`, `cancelled`.

When done: `score` (share of items without a finding), `grade` 1–5, the most
common problems and a page of findings. **Only the first 10 findings are ever
served** — `finding_total` and `severity_count` are the full numbers, so report
those, not the length of the page. The rest is visible to the merchant in the Pobo
administration, where the product grid can filter by the check.

Report like this: the grade and score, the counts per severity, the three most
common problems with one quoted example each, and the legal references if there
are any. Do not paste every finding.

A finding on one product can be fixed by hand with `get_content_widget` /
`edit_content_widget` (`widget_instance_id` points to the block); for many, use
the fix run.

### 6. Fix — `start_content_fix`

Only for a finished **product** check. The first call quotes what would be fixed,
what skipped and the price; confirm as above.

- Only the quoted words change; the rest of the text stays.
- Every fixed product is saved as a **draft** for the merchant to approve.
- Skipped with a reason: the platform's own description (never changed), missing
  data (never made up), a field the e-shop manages itself, a quote that changed
  since the check, a suggested fix equal to the quote.
- `translate: true` (default) translates again the products whose translation is
  a copy of the main language, priced separately; `false` skips them.
- `research: true` also looks up missing data on the web into the parameter
  block, priced separately.

Follow it with `get_content_fix` (`wait` up to 60 s): fixed and skipped per
reason, credits spent and the changes — product, field, language, old text, new
text. Show the user a few before/after pairs.

### 7. Verify — `recheck_content_fix`

Checks again only what the fix changed, with the same rules, and marks every
earlier finding as resolved or still there. Use this, **not a new
`start_content_bulk_check`**, to measure a fix — a fresh check of the same text
reports different findings from run to run. Paid, quote first. The result is in
`get_content_fix` → `comparison`: `resolved`, `persisted`, `new`. New findings are
the check reading the text differently, not damage done by the fix — say so.

### 8. Hand over

The fixed products are drafts. The merchant approves them in the administration,
or — with the user's yes — you approve them and upload (`publish-content`).

`list_content_bulk_check` lists past checks with their latest fix, so you can
pick up where an earlier conversation left off.

## Out of scope

- Translations — text left in another language, missing sentences in a
  translation: `fix-translations`. A content fix only swaps quoted words.
- A few texts right before upload — `check_content` in `publish-content`.
- Rewriting whole descriptions — `write-blog`, `design-description` or a
  generation (`manage-prompts`).
