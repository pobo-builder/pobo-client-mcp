---
name: diagnose-content
description: Find out why a generated Pobo Page Builder product description is missing something, came out wrong, or is not visible on the e-shop. Use when the user complains about generated content — "why is the dosage missing from the description", "it generated badly", "why isn't it on the e-shop", "the change never went through" — or asks what the AI actually found out about a product. Reads the generation runs, the web research behind them and the export that follows. Uses the `pobo` MCP server tools.
---

# Diagnose a Pobo Page Builder product description

You explain why a generated product description turned out the way it did.
Everything here is read-only — you inspect generation runs, the web research
behind them and the export that follows. Nothing you do changes content.

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

## The three questions, in order

A missing or wrong fact fails in exactly one of three places. Work through them
in order and stop at the first that explains it — answering the wrong one sends
the user off to fix something that was never broken.

1. **Did the instruction ask for it?** If nothing told the AI to write the
   thing, no amount of research would have put it there.
2. **Did the research find it?** A fact no provider returned was never
   available to the writing model, however plainly it sits on the
   manufacturer's page.
3. **Did it get exported?** Content can be perfect in Pobo and still not be on
   the e-shop.

## Workflow

### 0. Locate the product

`list_eshop` → `find_product` (EAN, code, URL or name) or `list_product`. You
need `eshop_id` and `product_id` for everything below.

### 1. Did the instruction ask for it?

`get_prompt` on the profile used for the product returns the general prompt text
and the per-widget instructions. `list_design_widget` names the widgets those
instructions target.

- Nothing mentions the missing thing → **this is the answer.** The fix is the
  prompt. Add a per-widget instruction with `set_widget_prompt` (see the
  `manage-prompts` skill), test it with `preview_generation`, then save.
- The instruction is there but positional ("in the first section write…") → likely it
  landed in a different widget than intended. Per-widget instructions target one
  widget deterministically; recommend converting it.
- The instruction is there and pinned to the right widget → go to step 2.

To see which profile a run actually used, `list_generation` (filter by
`product_id`) shows `prompt_text_name` per run — profiles change over time, and
an old description was generated with whatever was current then.

### 2. Did the research find it?

`get_product_research` with `product_id` returns, per recent generation run,
what every provider returned — plus a `coverage` block that is the fast answer:

- `coverage.key_spec_label` — every parameter that turned up across providers.
- `coverage.field_provider` — which providers filled each free-text field
  (`description_snippet`, `usage_and_dosage`).
- `coverage.source` — the pages that were actually cited.

Read it like this:

- **The fact is in some provider's payload, but not in the description** → the
  research had it and the writing model dropped it. Usually the instruction was
  vague or the widget had a tight character limit. Back to step 1.
- **No provider returned it, and `coverage.source` lists the right page** →
  the provider opened the page and did not extract that fact. **This is a Pobo
  limitation, not the user's prompt.** Say so plainly, and tell them to report
  it — the research contract may need a field for that class of information.
- **`coverage.source` is empty, or `research_status.has_data` is false** →
  research produced nothing. Check `research_status.reason` (below).

`get_generation` carries the same block for one specific run alongside its
settings — there it is the top-level `research_coverage`, while
`get_product_research` nests it per run as `research[].coverage`. Use
`get_generation` when the user asks about one dated description rather than
about the product in general.

### 3. Did it get exported?

`get_export_status` lists recent exports to the e-shop platform.

- A run with `failed_at` set is **dead** — it is never retried, so its products
  stayed unexported. The content is fine; the export needs re-running from the
  Pobo admin.
- `accepts_new_export: false` means an export is still in flight; wait for it.
- No recent export at all, and the product is `ready` → nobody started one.

## Reading the states honestly

These four are misread constantly. Never guess at them and never soften them
into "something went wrong".

| Signal | What it actually means |
|---|---|
| `is_complete: false`, `skip_reason: null` | The run is **stuck or still queued** — NOT failed. |
| `skip_reason` set | The run was deliberately skipped; the reason says why. |
| `research_status.reason: "no_ean"` | Research **never ran** — the product has no EAN to search by. It did not run and find nothing. |
| `research_status.reason: "no_data"` | Research ran and came back empty. |

`before`/`after` style comparisons do not exist here — this skill explains one
run, it does not measure performance. That is the `product-analytics` skill.

## Error handling

- `Eshop not found.` — wrong `eshop_id`, or the e-shop does not belong to this
  account; re-run `list_eshop`.
- `Generation run not found for this eshop.` — the run id belongs to a different
  e-shop or does not exist; re-run `list_generation`.
- Empty `research` array from `get_product_research` — no run of this product
  recorded any research. Either research was never enabled for those runs
  (`use_web_research` / `search_model` in the run settings), or the product has
  no EAN. Check `list_generation` settings.
- 401 — see Prerequisites & auth above.

## Reporting to the user

Say which of the three questions failed, in one sentence, before any detail.
"The research never found it" and "it was never in the brief" lead to
completely different fixes, and the user needs the difference more than they
need the payloads.

Quote concrete evidence — the cited source URL, the provider names, the widget
the instruction was pinned to. When the answer is a Pobo limitation, say that
too; do not send the user off to rewrite a prompt that was already correct.

## Out of scope

- The assembled prompt Pobo sends to the model — not available on this channel.
  Everything you need is in the profile, the per-widget instructions and the
  research coverage.
- Fixing the content — this skill diagnoses. Prompt edits are `manage-prompts`,
  approving content is `confirm-products`, styling is `style-widgets`.
- Performance of a description (views, add-to-cart) — that is
  `product-analytics`.
- Re-running an export or a generation — both are started in the Pobo admin.
