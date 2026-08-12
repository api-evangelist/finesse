---
name: finesse-agentic-checkout
description: Buy a FINESSE item on behalf of a consenting human buyer using the store's UCP/MCP commerce endpoint — search the catalog, build a cart, convert it to a checkout, set fulfillment, and complete payment only after explicit buyer approval.
api: FINESSE UCP Commerce MCP
endpoint: https://finesse.us/api/ucp/mcp
generated: '2026-08-12'
method: generated
source: mcp/finesse-mcp-tools-list.json (live tools/list, 2026-08-12) + https://finesse.us/agents.md
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - update_cart
  - get_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Agentic checkout at FINESSE

FINESSE is a direct-to-consumer apparel brand whose storefront implements the
[Universal Commerce Protocol](https://ucp.dev). There is no REST developer API and no OpenAPI —
the machine-readable contract is the MCP tool manifest at `POST https://finesse.us/api/ucp/mcp`.
Every tool name and argument below is transcribed from a live `tools/list` response; the full
`inputSchema` for each tool is stored in `mcp/finesse-mcp-tools-list.json`.

## Before you start

1. **Confirm the protocol version.** `GET https://finesse.us/.well-known/ucp`. The store declares
   `2026-04-08` as current and also supports `2026-01-23`. Responses carry
   `x-shopify-ucp-mcp-api-version`.
2. **Bring an agent profile.** Every tool call requires
   `meta.ucp-agent.profile` — a URI describing your agent. Omitting it returns JSON-RPC error
   `-32001` with `data.code = invalid_profile_url`. This is not optional and there is no anonymous
   fallback for tool calls (`tools/list` itself is anonymous).
3. **Set buyer context early.** Pass `checkout.context.address_country` and
   `checkout.context.currency` so pricing, availability and tax are computed for the right market.
   These are hints — a real shipping address supersedes them.

## Steps

1. **Find the product** — call `search_catalog` with the buyer's intent. For a known handle or
   variant id, use `get_product` (single) or `lookup_catalog` (batch). Product variant ids are
   Shopify GIDs of the form `gid://shopify/ProductVariant/…`.
2. **Build a cart** — call `create_cart` with `line_items[]`, each `{item: {id: <variant gid>},
   quantity: <int>}`. Keep the returned cart id (`gid://shopify/Cart/abc123?key=secret` — the
   `key` query parameter is part of the identifier, do not strip it). Amend with `update_cart`,
   read back with `get_cart`, abandon with `cancel_cart`.
3. **Convert to a checkout** — call `create_checkout`, passing either `checkout.cart_id` (cart
   contents win over any overlapping fields you also send) or `checkout.line_items[]` directly.
   Supply `checkout.buyer.email` and, where known, `phone_number`.
4. **Set fulfillment** — call `update_checkout` with
   `checkout.fulfillment.methods[]`, each carrying `type`, the `line_item_ids` it covers, the
   `destinations[]` address, and once options are returned, `groups[].selected_option_id`. The
   store declares `allows_multi_destination.shipping = false` and
   `allows_method_combinations = [["shipping"]]`, so a single shipping destination per checkout.
5. **Apply a discount only if the buyer mentions one** — `checkout.discounts.codes[]` on
   `update_checkout`. Codes are case-insensitive and each submission *replaces* the previous set;
   send an empty array to clear. Do not fish for codes.
6. **Read totals back and quote them correctly** — `get_checkout`. Money is an integer in ISO 4217
   **minor units** paired with a currency code: `{"amount": 2500, "currency": "USD"}` is $25.00.
   Divide by 100 for two-decimal currencies; zero-decimal currencies such as JPY are already whole
   units. Never quote the raw integer to a human.
7. **Get explicit approval, then complete** — attach a payment instrument
   (`checkout.payment.instruments[]`, `handler_id` one of `gpay`, `shopify.card`, `shop_pay`) and
   call `complete_checkout`. **The buyer must approve the payment contemporaneously.** If you
   cannot obtain that approval at the moment of payment, FINESSE's own `agents.md` instructs you to
   stop and route the purchase through the Shop skill (`https://shop.app/SKILL.md`) instead.
8. **Confirm** — `complete_checkout` returns the order id and Thank You Page URL; `get_order`
   retrieves order detail afterwards. Use `cancel_checkout` to abandon.

## Error and rate-limit handling

- Errors are JSON-RPC 2.0 objects: `error.code` (e.g. `-32001`) plus a `data` block carrying a
  string `data.code` (e.g. `invalid_profile_url`), human `data.content`, and sometimes a
  `data.continue_url` to hand back to the buyer. See `errors/finesse-problem-types.yml`.
- The endpoint is rate-limited per IP. Back off on `429`. Responses carry
  `shopify-complexity-score` / `shopify-complexity-score-v2` as a cost signal, and every response
  carries `x-request-id` — log it before escalating.
- There is no documented idempotency key on this surface. Treat `complete_checkout` as
  non-idempotent: do not blind-retry it. Re-read state with `get_checkout` first.

## Do not

- Do not complete a payment without contemporaneous buyer approval.
- Do not screen-scrape or script the HTML storefront when these tools cover the flow.
- Do not assume a public REST API exists — it does not.
