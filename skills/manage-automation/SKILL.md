---
name: manage-automation
description: Manage scheduled nightly AI description automation for Pobo Page Builder — create, preview, pause/resume and review the run history of rules that generate descriptions for products that still have none. Use when the user wants an e-shop to "fill itself in", asks to automate or schedule description generation, or asks whether an automation is healthy, what it costs, or why it picked up few (or no) products. Uses the `pobo` MCP server tools.
---

# Manage Pobo Page Builder description automation

You manage the merchant's scheduled nightly automations — rules that generate AI
product descriptions for products that still have none, without anyone picking
products in the grid by hand. All logic runs on the Pobo Page Builder backend.

Communicate with the user in their language (typically Czech).

## Prerequisites & auth

The `pobo` MCP server is connected once via OAuth — the user runs
`claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
and logs in with their Pobo Page Builder account in the browser. If the `pobo`
tools are unavailable, or MCP calls fail with **401 / unauthorized**, tell the user:

> Připojte Pobo server příkazem
> `claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client`
> a přihlaste se v prohlížeči svým Pobo účtem. Pokud připojení vypršelo, spusťte
> `/mcp` a přihlaste se znovu.

There are no tokens to handle — authentication is a browser login, never ask the
user for credentials in the conversation.

## What an automation is

A rule runs on a schedule (at most once a day), picks up to `batch_size`
products that still have no description, and generates them the way its linked
prompt profile says (design, languages, SEO — the automation only decides WHAT
and WHEN, the prompt decides HOW). **An automation spends the merchant's
credits every night on its own — treat setting one up, or widening one, as a
standing commitment, not a one-off action.**

## Workflow

### 1. Pick the e-shop and see what already runs

`list_eshop` → `list_automation_rule` (optionally filtered by `is_paused`) to
see existing rules: schedule, prompt, scope, batch size, paused state, last
and next run.

### 2. The prompt decides HOW, the rule decides WHAT and WHEN

`list_prompt` — the profile a rule references must be able to generate text.
A prompt of type `image` or `scss` has no design and no section settings, so
`create_automation_rule` / `update_automation_rule` refuse it with:

> `Prompt "{name}" is a {type} profile and cannot generate product
> descriptions. Pick a prompt from list_prompt.`

### 3. Create the rule

`create_automation_rule` with `eshop_id`, `name`, `prompt_id`, `schedule_cron`
(a cron expression in the eshop timezone, at most once a day), `batch_size`
(the nightly ceiling on products, and therefore on credits — max 500),
optional `description`, `category_id[]` / `brand_id[]` to limit scope (omit
for the whole catalogue).

**It is created PAUSED unless you pass `is_paused: false` explicitly.** The
schedule was just invented by an agent, not chosen by a human — the merchant
should see `preview_automation_rule` before it is allowed to run.

### 4. ALWAYS preview before telling the merchant to start one

`preview_automation_rule` with `eshop_id` and `rule_id` is a free dry run of
the selector: `candidate_count` (how many products the rule would find),
`would_queue_next_run` (capped by `batch_size`), `credit_available`, a
`sample_product` of up to 5, and the `selector` rule itself: **only products
without a description are ever picked, and a product generated in the last 30
days is skipped.** So a low or zero `candidate_count` usually means the
catalogue is already covered, not that the rule is broken — say so instead of
troubleshooting a rule that works exactly as designed.

### 5. Start, pause, resume or change it

`update_automation_rule` is **partial** — only the fields you send change.
`is_paused: false` resumes it (and clears its failure streak, see below);
`is_paused: true` pauses it. `category_id` / `brand_id` **REPLACE** the whole
scope — an empty array widens the rule back to the whole catalogue, it does
not leave the existing scope untouched. Never resume or widen an automation
without telling the merchant what it will now cost per night
(`preview_automation_rule` again after a scope or batch size change).

### 6. Check on it later

`get_automation_rule` (one rule, with its last 5 runs and an
`auto_pause_warning` flag) or `list_automation_run` (history across the
e-shop, optionally filtered by `rule_id`) — read `status` per run:

| `status` | Meaning |
|---|---|
| `success` | every candidate was queued |
| `partial` | some candidates were skipped (credit or the 30-day anti-dupe window), but at least one job was created |
| `failed` | an error stopped the run before any job was created |
| `skipped` | the selector found no candidates, or the e-shop has too many jobs already pending |
| `running` | still in progress (this is normally seconds, not minutes) |

`skipped_no_credit` and `skipped_recently_generated` on a run say exactly why
candidates did not become jobs.

## Rules you MUST follow

- An automation spends credits every night on its own — never present setting
  one up, unpausing one, or widening its scope as free or reversible without
  cost; always check `preview_automation_rule` first.
- `create_automation_rule` makes a PAUSED rule unless `is_paused: false` is
  passed explicitly.
- Only products WITHOUT a description are ever picked, and one generated in
  the last 30 days is skipped — a low candidate count is normal, not a bug.
- **There is no manual "run tonight now" trigger, and this is deliberate** —
  it is not a gap to work around. Automation credits and the text-generation
  pipeline do not have room for an agent to also press a button that spends
  them; a rule only ever runs on its own schedule.
- Three failed nights in a row pause a rule automatically
  (`consecutive_failure` ≥ 3); resuming it (`is_paused: false`) resets the
  counter, so tell the merchant what actually failed before resuming rather
  than just clearing the warning.

## Error handling

- `Eshop not found.` — wrong `eshop_id`; re-run `list_eshop`.
- `Automation rule not found.` — wrong `rule_id`, or it belongs to a
  different e-shop; re-run `list_automation_rule`.
- `Prompt not found.` — the prompt does not belong to this e-shop; re-run
  `list_prompt`.
- `Prompt "{name}" is a {type} profile and cannot generate product
  descriptions...` — pick a prompt that can generate text (not `image` or
  `scss`); see step 2.
- Validation error on `schedule_cron` — only daily or weekly patterns are
  accepted, at most once a day; no sub-hourly expressions.
- Validation error on `batch_size` — 1 to 500.
- A run with `status: "skipped"` and a nonzero `candidate_count` — the e-shop
  hit its pending-job backpressure limit; check `error_message` on the run for
  the count, and try again once existing jobs finish rather than raising
  `batch_size` to compensate.
- 401 — see Prerequisites & auth above.

## Out of scope

- Manually triggering a run right now — no such tool exists on this channel,
  by design (see "Rules you MUST follow").
- Catalogue priority (newest-first / traffic / add-to-cart) and auto-approve
  + auto-export after generation — both are rule settings, but set only in
  the Pobo Page Builder admin, not through these tools.
- Deleting an automation — no such tool exists here; pause it instead
  (`update_automation_rule` with `is_paused: true`).
- Generation settings other than WHAT/WHEN (design, languages, SEO, section
  counts, image flow) — those live on the prompt profile, see the
  `manage-prompts` skill.
- Diagnosing why one particular generated description came out a certain way
  — that is the `diagnose-content` skill.
