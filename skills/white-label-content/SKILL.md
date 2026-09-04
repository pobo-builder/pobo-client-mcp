---
name: white-label-content
description: Work with Pobo Page Builder content that lives on your own entities — find entities by id, name or the text inside them, add widgets to one product or fifty at once, rewrite or remove them in bulk, copy content between entities, and undo any of it. For white label (B2B) e-shops whose products, categories and articles are identified by your own ids. Use when the user says "přidej ke všem těmhle produktům…", "uprav ve všech widgetech…", "najdi produkty, kde se píše o…", "zkopíruj obsah z produktu X na Y" or "vrať to zpátky". Uses the `pobo` MCP server tools.
---

# Work with white label entity content

You manage Pobo Page Builder content on a white label e-shop: the merchant's own
products, categories, articles and pages, addressed by **their** ids, not Pobo's.

Communicate with the user in their language (typically Czech) and write the
content in the language the e-shop sells in, not in English.

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

**This skill only applies to white label e-shops.** On any other platform the
tools answer `Eshop not found.` — that is the scope gate, not a bug. If that is
what you get, the merchant is not a white label host and the work belongs to the
regular product/blog skills instead.

## Find before you change

`list_eshop` → `list_entity`. The search covers three things at once — the
host's own entity id, the entity name and **the text inside the widgets** — and
the answer says which of them matched, with a snippet. Case and diacritics do
not matter, so "bazen" finds "Bazén".

That is how you answer "the products where we still mention the old delivery
date" without anyone pasting a list. Without a query it lists the newest
entities.

`get_entity_content` is the numbered snapshot of one entity: widgets in order
with `position`, `widget_instance_id`, `widget_id`, the template name and the
texts keyed by role. **Read it before any edit** — `widget_instance_id` comes
from here, and editing from memory of an earlier snapshot is how you overwrite
the wrong widget.

## Adding content

`get_entity_widget_catalog` first, always. It returns the widget templates this
e-shop can use and, for each, its roles with `hint` and `max_length`, its image
slots and its item count. **You can only use templates from this catalog**, and
`add_entity_widget` requires the `widget_id` — never guess one and never carry
one over from another e-shop.

`add_entity_widget` takes up to **50 entities per call**:

- same text everywhere → put it in `content` (a delivery notice, a seasonal
  banner),
- different text per entity → put it in `content_per_entity`, keyed by entity
  id (an FAQ written for each product).

Keep the **same `widget_id` for the whole batch**. Mixing templates is how five
products end up looking like five different shops.

**A non-repeatable widget holds one item.** Five FAQ questions are five widgets,
not one `add_entity_widget` call with five items — check `item_count` in the
catalog before you assume otherwise.

The text you send is written into **every language** the e-shop has. Per-language
wording is not possible here; if the merchant needs it, that is admin work.

## Changing and removing

`edit_entity_widget` rewrites texts by role. Two ways to aim it:

- `widget_instance_id` — exactly one widget on one entity,
- `widget_id` — **every instance of that template** across the whole batch. This
  is the one for "do každého widgetu napiš…".

Only the roles you send change; images, other roles and the structure survive.

`remove_entity_widget` picks widgets by instance, by template, or by `query` —
the text inside them. **The first call removes nothing.** It answers with how
many widgets would go, how many entities are affected and a sample of them.
Show that to the user, get an explicit yes, and only then repeat the call with
`confirmed: true`. `confirm_required` is not an error.

`move_entity_widget` reorders inside one entity; positions are 1-based.

`copy_entity_widget` copies from one entity to up to fifty others:
`mode: "append"` keeps what the target already has, `mode: "replace"` throws it
away first. **Say which one you are about to do before you call it** — replace
is the destructive one. `widget_instance_id[]` copies only some of the source's
widgets.

## Every batch is undoable — use it

Every write stores a version of the entity first, and a whole run shares one
`batch_id`, which comes back in the response. `revert_entity` takes either:

- `batch_id` — the whole run goes back to how it was, in one call,
- `entity_id` + `version_id` from `get_entity_history` — one entity, one step.

`list_entity_batch` lists the recent runs on the whole e-shop — when, by which tool,
how many entities and a few of their ids. That is where you start when the user says
"vrať, cos včera udělal" and nobody kept the id.

A revert saves the current state as a version too, so a revert can be reverted.
Versions cover writes from anywhere — this server, the host's own integration
and the merchant's editor in Pobo — so a revert never silently discards a change
someone made by hand.

**Keep the `batch_id` in your reply** after any bulk write. It is what the user
needs if they change their mind an hour later, and it is far cheaper than
undoing fifty entities by hand — and if it is lost, `list_entity_batch` finds it.

## Working with the user

- Say what a call is about to touch **before** it runs: how many entities, which
  template, and whether anything gets overwritten.
- Report per entity, not just a total. The tools answer per entity — an id that
  was not found is reported, not guessed at, and the rest of the batch still
  runs.
- If `unmatched_content_key` comes back, a key in `content_per_entity` matched
  no entity in the call — the text you wrote for it was **not** used. Fix the id
  and repeat rather than letting the generic text stand.
- Over 50 entities, work in batches and say so. Each batch has its own
  `batch_id`; keep the list.

## Error handling

- `Eshop not found.` — not a white label e-shop, or not one this account owns;
  see the scope note in Prerequisites.
- `Entity not found on this eshop.` (per entity) — the host's id does not exist
  in Pobo yet. Entities are registered by the host's integration or by
  `fill_design_content`, not by the widget tools.
- Widget template not in the catalog — re-run `get_entity_widget_catalog`.
- `At most N characters…` — the limit comes from the template, not from Pobo's
  taste; shorten the text, do not switch template to dodge it.
- `Nothing was written — no role you sent exists in that widget.` — you used a
  role from a different template; read `get_entity_content` for what that widget
  actually has.
- 401 — see Prerequisites & auth above.

## Out of scope

- AI-written content — nothing here calls a model; the words are yours and no
  credits are spent.
- Filling a whole template at once — that is `fill_design_content`, which
  replaces the entity's content from a design template.
- Per-language wording, video widgets, and publishing to the host's own site.
- Machine-to-machine automation: this server authenticates a person. A nightly
  job over thousands of products stays on the white label REST API.
