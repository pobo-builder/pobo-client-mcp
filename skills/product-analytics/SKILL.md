---
name: product-analytics
description: Read and interpret Pobo Page Builder product performance analytics — how a product performs after its Pobo description went live. Use when the user asks how a product is doing, whether the new description helped, which parts of the description visitors read, or wants to find products whose content needs improvement. Uses the `pobo` MCP server tools.
---

# Pobo Page Builder product analytics

You read the performance analytics of a product's Pobo description and turn the
numbers into content decisions. All data comes from the Pobo Page Builder
backend — nothing runs on your machine.

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

There are no tokens to handle — never ask the user for credentials.

## Workflow

1. `list_eshop` — find the eshop to work with.
2. `find_product` — resolve the user's identifier (EAN, product code, URL or
   name) to a Pobo product; `list_product` when they describe a set instead
   ("products with label X", "approved products in category Y").
3. `get_product_analytics` — fetch the analytics (period 30d/60d/90d,
   default 90d).

## What the data means

- **totals** — page views, unique visitors, add-to-cart count and order count
  for the whole period.
- **pobo_content.first_detected_at** — the day the Pobo description first
  appeared on the live page. Everything "before/after" pivots on this date.
- **before_after_summary** — average daily views/add-to-cart and the
  add-to-cart rate before vs. after the description, with percent changes.
  Both sides cover a symmetric window (N days right before × first N days
  after detection, N = shorter side) so seasonality of a longer "before"
  stretch cannot skew the averages — `before.days` always equals `after.days`.
  `before: null` means the product has no history before the description
  (new product) — say so, do not invent a comparison.
- **engagement** — scroll depth funnel (share of views reaching 25/50/75/100 %
  of the page), average active reading seconds, most opened FAQ questions.
- **section_map** — the content heatmap: every section of the description
  (with its real widget name), how many visitors saw it (`seen_percent`),
  average dwell seconds and `cart_rate_percent` — the share of sessions that
  saw the section and later added the product to cart.

## Interpretation rules (MANDATORY)

- **Correlation, not causality.** Before/after changes and `cart_rate_percent`
  are correlations — engaged visitors scroll further AND buy more, and
  seasonality or campaigns move numbers too. Phrase findings as
  "after the description went live, X increased by Y%", never "the
  description caused it".
- **Orders are attributed, not complete.** Order counts cover only sessions
  where the customer viewed the product page first, and only on Shoptet.
  Call them "orders after viewing the product page".
- **Null ≠ zero.** `null` values and empty blocks mean not enough data was
  collected yet (tracking starts when the Pobo plugin ships to the eshop) —
  never present missing data as bad performance.
- Small numbers make wild percentages — if the before period has little
  traffic, say the comparison is not reliable yet instead of quoting it.

## Turning findings into action

- Sections with **high `cart_rate_percent` but low `seen_percent`** are the
  gold: visitors who see them convert, but few scroll that far. Recommend
  moving them up in the description.
- A scroll funnel that collapses early (e.g. only 20 % reach half the page)
  suggests the opening sections do not hold attention.
- Top FAQ questions show what customers actually worry about — useful input
  for description copy.
- For a batch of underperforming products, label them via `list_label` /
  `create_label` + `assign_product_label` so the content team can filter them
  in the Pobo admin grid (see the label-products skill).
