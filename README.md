# Pobo Page Builder — Claude Code plugin

[Pobo Page Builder](https://www.pobo.space) tools for Claude — nothing runs on
your machine, all logic lives on the backend:

- **AI styling of widgets** — Claude reads your e-shop's look (colors, fonts,
  buttons, border radius), generates matching styles, and deploys them through
  Pobo Page Builder.
- **Product labeling** — Claude takes a product list in any shape (EANs, product
  codes, URLs, or names — e.g. an xlsx from a client), resolves it to Pobo
  products, and labels them so you can filter the batch in the Pobo admin grid.
- **Prompt management** — Claude creates and edits the AI generation prompt
  profiles Pobo uses for product descriptions, and decomposes a client's brief
  into precise per-widget instructions on your design template.
- **Product confirmation** — Claude switches a batch of products between
  `draft` (waiting for approval) and `ready` (approved for export) once the
  client signs off the new content.

Supported platforms: Shoptet, Shopify, WooCommerce, PrestaShop, Upgates.

## Installation

In [Claude Code](https://claude.com/claude-code), run:

```
/plugin marketplace add pobo-builder/pobo-client-mcp
/plugin install pobo@pobo-builder
```

## Connecting your Pobo Page Builder account

Pobo exposes two separate MCP servers. Most people only need the first one —
the second is for white label (B2B) hosts managing content on their own
entities. Connect either or both; they are independent OAuth connections and
coexist without conflict.

### `pobo` — merchant tools

Everything except white label bulk content editing: writing the description of a
product, a category or a blog article, styling, product labeling and status,
prompt management, diagnostics and analytics.
Run this once in your terminal:

```
claude mcp add -s user --transport http pobo https://api.pobo.space/mcp/client
```

Claude Code opens your browser — log in with your Pobo Page Builder account and
approve access. That's it: no tokens, no environment variables, no config files.
The connection persists across all your projects; if it ever expires, run `/mcp`
in Claude Code and re-authenticate.

### `pobo-whitelabel` — white label bulk content editing

Only for white label (B2B) e-shops, where products, categories and articles
are identified by your own ids. Connect it under a **different** server name —
reusing `pobo` would overwrite the connection above instead of adding a second
one:

```
claude mcp add -s user --transport http pobo-whitelabel https://api.pobo.space/mcp/whitelabel
```

Same login flow as above. If your e-shop is not a white label host, its tools
will report `Eshop not found.` for every call — that's the scope gate, not a
bug, and you likely don't need this server at all.

### Claude.ai / ChatGPT (web)

Both servers work without Claude Code. Add a custom connector with the
relevant URL — `https://api.pobo.space/mcp/client` for merchant tools,
`https://api.pobo.space/mcp/whitelabel` for white label bulk content editing:

- **Claude.ai**: Settings → Connectors → Add custom connector
- **ChatGPT**: Settings → Connectors → Create

Leave the OAuth Client ID/Secret fields empty — you will be asked to log in and
approve access in your browser.

## Usage

Just tell Claude, for example:

> Make my Pobo Page Builder widgets match my e-shop's design.

Claude picks the e-shop, reads its design, generates the styles, and deploys them.
You can then review the changes on your e-shop (hard refresh) and iterate with
feedback. Deployed AI styles are also visible in the Pobo Page Builder admin, where
you can remove them at any time.

Or hand Claude a product list from a client:

> Here is the client's xlsx with the products to rewrite — label them
> "Sportrec update 07/2026" in Pobo.

Claude resolves the products (URLs, EANs, codes, or names), creates the label, and
attaches it — ambiguous or unmatched items are reported back instead of guessed.
The labeled batch is then filterable in the Pobo Page Builder admin product grid.

Or refine your generation prompts:

> Here is the client's brief — set up a generation prompt on the "Elektro"
> template: mention the stainless steel construction in the first section and
> add a sizing FAQ.

Claude creates (or updates) the prompt profile, links the design template, and
pins each requirement to the right widget as an explicit per-widget instruction.
Before saving, it can dry-run the prompt against a couple of real products and
show you the rendered result — that costs nothing and writes nothing, so you can
iterate until the copy is right. You then run the real generation from the Pobo
Page Builder admin as usual.

Or confirm a finished batch:

> The client approved the new descriptions — mark the "Sportrec update 07/2026"
> batch as ready for export.

Claude switches the products from `draft` to `ready` (or back to `draft` when
content returns for rework). Only these two statuses are ever touched — products
currently in review or AI generation are reported and skipped, never overridden.

Or ask why a description came out wrong:

> The dosage is missing from that pet food description, even though it's
> right there on the manufacturer's page. Why?

Claude works through the three places it can fail — was it in the instruction,
did the web research find it, and did the content get exported — and tells you
which one it was. That matters, because "nobody asked for it" and "the research
never found it" need completely different fixes, and one of them is not yours.

Or have Claude write a blog article:

> Write an article about choosing winter tires, about three pages long, and
> place our four best-selling models into it.

Claude composes the article itself from your widget templates — free, no credits
— places real products as a carousel rather than writing their names into prose,
and runs a quality check for empty slots at the end. Pobo's own server-side
generator is available too when you want it; Claude always says up front which
one it is using and what it costs.

The same tools write **product and category descriptions**, not just articles —
each one takes `entity_type`. And every write is versioned, so "undo what you did
to those twenty products" is one call, not twenty.

Or, on a white label e-shop, change content across a batch of your own entities:

> Add a widget with Christmas Eve delivery information to all five products
> from that holiday promotion.

Claude finds the entities, picks one widget template from your catalog and puts
the same text on all of them — or a different text on each, when that is what you
need. It can also find entities by what is written **inside** their widgets
("where we still mention the old delivery date"), rewrite that text everywhere at once, or
copy content from one product to others. Removing asks twice: the first call only
shows what would go. Every batch comes back with a `batch_id`, so one call takes
the whole run back if you change your mind.

### Filtering products

Claude can filter products with the same filters as the Pobo Page Builder admin
product grid: **label**, **grid tabs** (all / without description / edited in
Pobo / favourites / waiting for approval), **full-text search**, **category**,
**brand** and **visibility**, plus ordering by recent edits, page views, or
add-to-cart stats. Categories are searchable by name the same way, so "drafts
in the Massage category" needs no ids from you. So requests like

> Switch everything with the "Sportrec update 07/2026" label that waits for
> approval to ready.

work without pasting any product list — Claude filters the batch itself and
confirms the count with you before switching. The same filters are available in
the admin grid, so you can verify the result visually.

## What's in the plugin

- `skills/style-widgets/` — the workflow for AI styling of widgets
- `skills/write-blog/` — the workflow for authoring and editing descriptions: blog articles, products and categories
- `skills/label-products/` — the workflow for labeling products from a client-supplied list
- `skills/confirm-products/` — the workflow for switching product status (draft ⇄ ready)
- `skills/manage-prompts/` — the workflow for managing and dry-run testing AI generation prompt profiles
- `skills/manage-automation/` — the workflow for scheduled nightly description automation: create, preview, pause/resume, run history
- `skills/diagnose-content/` — the workflow for finding out why a generated description is missing something
- `skills/product-analytics/` — the workflow for reading how a description performs after it went live
- `skills/white-label-content/` — the workflow for white label e-shops: finding entities and editing their widget content in bulk (uses the separate `pobo-whitelabel` server, see above)

See [`TOOLS.md`](TOOLS.md) for the complete reference of every tool both servers expose.

The connection to the Pobo Page Builder MCP servers is set up by the `claude mcp add`
commands above (OAuth login in the browser), not bundled in the plugin.

## About

Made by [Pobo Page Builder](https://www.pobo.space).
