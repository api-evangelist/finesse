---
name: finesse-catalog-browse
description: Read the FINESSE product catalog without authenticating and without transacting, using the unauthenticated storefront JSON endpoints the store documents for agents.
api: FINESSE Storefront JSON (read-only)
endpoint: https://finesse.us/
generated: '2026-08-12'
method: generated
source: https://finesse.us/agents.md (read-only browsing section) + live probe of https://finesse.us/products.json (HTTP 200, 2026-08-12)
operations:
  - GET /products.json
  - GET /products/{handle}.json
  - GET /collections/{handle}/products.json
  - GET /collections/all
  - GET /search?q={query}&type=product
  - GET /sitemap.xml
---

# Reading the FINESSE catalog

FINESSE documents a read-only browsing surface in its own `agents.md` for agents that need store
data but are not transacting. No credentials are required. These are Shopify storefront JSON
endpoints served from FINESSE's apex domain; they are **not** a versioned developer API and carry
no published stability guarantee.

## Endpoints

| Purpose | Request |
|---|---|
| Paged product feed | `GET https://finesse.us/products.json` |
| One product | `GET https://finesse.us/products/{handle}.json` |
| Products in a collection | `GET https://finesse.us/collections/{handle}/products.json` |
| Browse everything (HTML) | `GET https://finesse.us/collections/all` |
| Search | `GET https://finesse.us/search?q={query}&type=product` |
| URL discovery | `GET https://finesse.us/sitemap.xml` |

## Response shape (observed 2026-08-12)

`/products.json` returns `{"products": [...]}` — 30 products per page. Each product carries
`id`, `title`, `handle`, `body_html`, `vendor`, `product_type`, `tags[]`, `published_at`,
`created_at`, `updated_at`, plus `variants[]`, `images[]` and `options[]`.

- `variants[]` — `id`, `title`, `option1..option3`, `sku`, `price`, `compare_at_price`,
  `available`, `grams`, `requires_shipping`, `taxable`, `position`, `product_id`, `featured_image`.
  **Note the money convention differs from MCP:** these storefront prices are decimal strings
  (`"78.00"`), not minor-unit integers.
- `images[]` — `id`, `src`, `width`, `height`, `position`, `variant_ids[]`, `product_id`.
- `options[]` — `name`, `position`, `values[]` (this is where size runs XS–3X appear).

See `data-model/finesse-data-model.yml` for the entity graph.

## Manners

- Paginate with `?page=` / `?limit=` rather than hammering the feed.
- `robots.txt` disallows `/search`, `/cart`, `/checkout*`, `/account`, `/orders` and
  faceted/sorted collection URLs — honor it. The search endpoint above is documented for agents by
  the store itself, but crawling it is disallowed; use it for a buyer's direct request, not for
  bulk indexing.
- Responses carry `x-request-id` and `shopify-complexity-score`. Back off on `429`.
- To buy, switch to `skills/finesse-agentic-checkout.md`. This skill never transacts.
