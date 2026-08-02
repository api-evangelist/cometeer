---
name: Browse the Cometeer coffee catalog
description: Enumerate and inspect Cometeer's flash-frozen coffee capsules and merchandise using the public, unauthenticated storefront JSON endpoints.
api: openapi/cometeer-storefront-openapi.yml
operations: [listProducts, listCollectionProducts, getProduct, getSitemap]
generated: '2026-08-01'
method: generated
source: https://cometeer.com/llms.txt
---

# Browse the Cometeer coffee catalog

Cometeer sells single-serve flash-frozen coffee capsules, roasted by third-wave
partners, plus brewing merchandise. Its catalog is readable with **no
credential at all** — Cometeer documents these endpoints for agents itself,
under "Read-Only Browsing (No Authentication Required)" in
<https://cometeer.com/llms.txt>.

Use this skill for research, price checking, and availability. Use
`cometeer-agentic-purchase.md` when the user actually wants to buy.

## Auth

None. Do not send an Authorization header — these are public endpoints.

## Steps

1. **Enumerate the catalog.** Call `listProducts`:
   `GET https://cometeer.com/products.json?limit=250&page=1`.
   Page forward with `page=2`, `page=3`, … until the `products` array comes
   back empty. There is no total count, no cursor and no `next` link, so the
   empty page *is* the terminator.

2. **Narrow by collection instead, when you know one.** Call
   `listCollectionProducts`:
   `GET https://cometeer.com/collections/{handle}/products.json?limit=250`.
   The handle `all` returns everything. Collection handles come from the
   storefront navigation, or from `getSitemap`.

3. **Inspect one product.** Call `getProduct`:
   `GET https://cometeer.com/products/{handle}.json`.
   The `handle` is the URL slug, e.g. `frozen-gold-ember-espresso`.

4. **Read the shape correctly.** Price lives on the **variant**, never on the
   product. Each product has `variants[]`; each variant carries `price`,
   `price_currency`, `sku`, `compare_at_price` and `quantity_price_breaks`.
   Segment with `product_type` — observed values include `Coffee Sleeve` and
   `Merchandise` — and note that `vendor` is the *roasting partner*, not always
   Cometeer (East Fork, Fellow, Subminimal and notNeutral appear for
   merchandise).

5. **Discover URLs** with `getSitemap`: `GET https://cometeer.com/sitemap.xml`.

## Rules

- **Do not screen-scrape.** Cometeer's `/llms.txt` explicitly asks agents to
  prefer these JSON endpoints and the Shop skill over scripting the storefront.
- **Back off on 429.** Rate limiting is per IP and is documented, but no
  `Retry-After` or rate-limit headers are published — use exponential backoff.
- **Do not expect JSON search.** `/search?q=…&type=product` returns an HTML
  page. For structured search you need the UCP MCP `search_catalog` tool.
- **Errors are HTML, not JSON.** A bad handle returns a 404 HTML page, so
  branch on the status code and never assume a parseable error body. See
  `errors/cometeer-problem-types.yml`.
- Ignore products whose titles are prefixed `[Test KH]` or `[KH Test]` — these
  are merchant test fixtures visible in the public feed, not sellable goods.

## Filtering out non-purchasable entries

```
skip if product.title starts with "[Test" or "[KH Test"
skip if product.product_type == "TBD"
```

## See also

- `conventions/cometeer-conventions.yml` — pagination, rate limits, error envelopes
- `data-model/cometeer-data-model.yml` — Product / Variant / Image / Option graph
- `mcp/cometeer-tool-crosswalk.yml` — what the REST surface cannot do
