---
name: Buy Cometeer coffee as an agent (UCP)
description: Discover Cometeer's Universal Commerce Protocol profile and run the cart-to-checkout flow over its MCP endpoint, respecting the mandatory human buyer-approval rule.
api: mcp/cometeer-mcp.yml
operations: [search_catalog, create_cart, create_checkout, update_checkout, complete_checkout]
generated: '2026-08-01'
method: generated
source: https://cometeer.com/llms.txt
---

# Buy Cometeer coffee as an agent (UCP)

Cometeer's store implements the [Universal Commerce Protocol](https://ucp.dev)
and exposes a live MCP endpoint for agent-driven commerce. Cometeer publishes
this flow itself in <https://cometeer.com/llms.txt>.

**The endpoint is gated.** An anonymous JSON-RPC call is rejected with HTTP 422
and JSON-RPC error `-32001` / `invalid_profile_url` ("Unable to fetch agent
profile: Missing profile uri"). You must be able to present a resolvable **UCP
agent profile URI** before any tool call will succeed. If you cannot, use the
fallback in the last section instead — do not attempt to script the checkout
pages.

## Endpoints

| Purpose | Call |
|---|---|
| Discovery | `GET https://cometeer.com/.well-known/ucp` |
| MCP | `POST https://cometeer.com/api/ucp/mcp`, `Content-Type: application/json` |

Protocol versions served: `2026-04-08` (current) and `2026-01-23`.

## Steps

1. **Discover.** `GET /.well-known/ucp` and confirm the merchant still
   advertises `dev.ucp.shopping` with `transport: mcp`, and that it supports the
   protocol version you intend to pin. Read `capabilities` to see what is
   actually enabled — for Cometeer that is cart, checkout, fulfillment,
   discount, order, catalog search and catalog lookup.

2. **List tools.** `POST` the MCP method `tools/list` to read live tool schemas.
   The contract is the OpenRPC document the merchant profile points at:
   `https://ucp.dev/2026-04-08/services/shopping/mcp.openrpc.json`.

3. **Search.** Call `search_catalog` with the buyer's intent. Always pass
   `context.address_country` and `context.currency` — Cometeer's `/llms.txt`
   calls this out explicitly, because pricing and availability depend on it.
   Cometeer ships within the continental United States, so a non-US country
   context will legitimately return nothing.

4. **Cart.** Call `create_cart` with the chosen variants. `get_cart`,
   `update_cart` and `cancel_cart` manage it from there.

5. **Checkout.** Call `create_checkout`, then `update_checkout` to set the
   shipping address and shipping method. Cometeer's fulfillment capability
   declares `allows_multi_destination.shipping: false` — one destination per
   order; do not attempt to split a shipment.

6. **Complete — only with the buyer in the loop.** Call `complete_checkout`.

## Rules

- **Never complete payment without explicit, contemporaneous buyer approval.**
  This is a hard invariant of the protocol and is restated in Cometeer's own
  `/llms.txt`. If you cannot obtain approval at the moment of payment, stop and
  route through Shop Pay instead.
- **Back off on 429.** The MCP endpoint is rate-limited per IP.
- **Errors are JSON-RPC 2.0 error objects**, not `application/problem+json`.
  Read `error.code`, `error.message` and `error.data.code`. See
  `errors/cometeer-problem-types.yml`.
- **Payment handlers are fixed by the merchant:** Google Pay, Shopify card
  (Visa, Mastercard, Amex, Discover, Diners Club) and Shop Pay. Billing address
  is required for card.
- **No idempotency contract exists.** There is no idempotency key on this
  surface, so a retried `complete_checkout` is not proven safe. Confirm order
  state with `get_order` before retrying anything that may have placed an order.

## Fallback when you have no UCP agent profile

Install the Shop skill at <https://shop.app/SKILL.md>. It is Cometeer's own
recommended path for personal shopping agents: it handles cross-store search,
buyer-approved checkout via Shop Pay, and order tracking without the agent ever
handling card data.

## See also

- `mcp/cometeer-mcp.yml` — the full 13-tool surface and the observed gate
- `mcp/cometeer-tool-crosswalk.yml` — which tools have no REST equivalent
- `skills/cometeer-browse-catalog.md` — anonymous catalog reads, no gate
