---
name: transact-with-shop-icon
description: >-
  Use the Shop ICON UCP/MCP endpoint to search the catalog, build a cart and prepare a checkout for
  a human to approve — and know exactly where the point of no return is.
api: icon-aircraft:icon-aircraft-ucp-mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-22'
method: generated
source: >-
  Grounded in the live tool manifest saved verbatim at mcp/icon-aircraft-ucp-mcp-tools.json
  (anonymous tools/list, HTTP 200, 13 tools, 2026-08-22) and in the provider's published agent
  instructions at https://store.iconaircraft.com/agents.md.
---

# Transact with Shop ICON

Endpoint: `POST https://store.iconaircraft.com/api/ucp/mcp`
Protocol: Universal Commerce Protocol 2026-04-08 over MCP. `tools/list` answers anonymously.

## The rule that outranks everything else

**Do not complete a payment without contemporaneous human approval.** This is not a caution this
skill invented — the store publishes it in `/robots.txt` and `/agents.md`: agents must not complete
checkout, payment or order placement automatically, and must not use scripted form fills or
end-to-end flows that finalize payment. Your job ends at a checkout a person can look at and say yes
to.

## Every call needs an agent profile

All 13 tools require `meta.ucp-agent.profile` — a fetchable URI identifying you. The server resolves
it *before* it dispatches the method, so if you omit it you get:

```
{"jsonrpc":"2.0","error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri"}}}
```

…with HTTP 422 — and you get that *same* error for an unknown method name. Until your profile is
valid you cannot tell a bad tool name from a bad identity. Fix the profile first, always.

Also pass `context.address_country` and `context.currency`; the store's own instructions say pricing
and availability are inaccurate without them.

## Money

Every price in a tool response is an **integer in ISO 4217 minor units, paired with a currency
code**. `{"amount": 3000, "currency": "USD"}` is $30.00. Divide by 100 for two-decimal currencies
before you quote anything to a person. Zero-decimal currencies such as JPY are already whole units.
The storefront JSON endpoints on the same store use decimal strings instead — do not mix them.

## The flow

1. `search_catalog` — find candidates from the buyer's intent. Use `get_product` for full detail on
   one, or `lookup_catalog` to resolve several identifiers at once.
2. `create_cart` — build the cart. `update_cart` to change it, `get_cart` to read it back.
3. `create_checkout` — returns line items, totals, discounts and taxes. This is your price preview;
   there is no dry-run flag anywhere on this surface.
4. `update_checkout` — set the shipping address and fulfillment method. The store ships within the
   countries listed in `/meta.json` and does not ship to P.O. boxes.
5. **Stop. Show the buyer the totals and get an explicit yes.**
6. `complete_checkout` — only after that yes.

## Reversibility — read this before step 6

| Action | Reversal | Window |
|---|---|---|
| `create_cart` | `cancel_cart` | none published |
| `update_cart` | another `update_cart` | while the cart is open |
| `create_checkout` | `cancel_checkout` | none published |
| `update_checkout` | another `update_checkout` | while the checkout is open |
| `complete_checkout` | **none** | — |

There is no refund, void or reverse tool in the manifest, and `get_order` is read-only. Once
`complete_checkout` succeeds, the only way back is out-of-band: the published refund policy allows
unused, resalable merchandise to be returned for a full refund excluding shipping **within 30 days
of the original purchase date**, by physically shipping it to ICON Aircraft Store Returns, 2141 ICON
Way, Vacaville, CA 95688. You cannot execute that. Treat step 6 as final.

Source: https://store.iconaircraft.com/policies/refund-policy

## Idempotency

`complete_checkout` **requires** `meta.idempotency-key`. Generate one per intended purchase and
reuse it on retry — that is what stops a timeout from charging the buyer twice.

No other tool accepts an idempotency key. If `create_cart` or `create_checkout` times out, retrying
can create a duplicate; read back with `get_cart` / `get_checkout` before you retry.

## Rate limits

Per-IP, no published quota, no `RateLimit-*` headers and no `Retry-After`. Back off exponentially on
429. You cannot see your remaining budget before you exhaust it.

## What this surface does not do

No collection browsing, no full catalog enumeration and no store metadata — those are REST-only. See
`mcp/icon-aircraft-tool-crosswalk.yml` and the `browse-shop-icon-catalog` skill.
