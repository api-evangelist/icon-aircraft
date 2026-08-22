---
name: browse-shop-icon-catalog
description: >-
  Read the Shop ICON merchandise catalog — products, variants, prices, availability and
  collections — from the anonymous Shopify storefront JSON endpoints, without creating a cart or a
  checkout.
api: icon-aircraft:icon-aircraft-store-api
operations:
  - getStoreMeta
  - listProducts
  - getProduct
  - listCollections
  - listCollectionProducts
  - suggestSearch
generated: '2026-08-22'
method: generated
source: >-
  Grounded in openapi/icon-aircraft-store-api-openapi.yml, which was derived from
  https://store.iconaircraft.com/agents.md and verified by anonymous probe on 2026-08-22.
---

# Browse the Shop ICON catalog

Read-only. No credential, no signup, no cart, no checkout. Every operation below answered HTTP 200
anonymously when this skill was written.

## Before you start

Call `getStoreMeta` (`GET /meta.json`) once. It tells you the currency, the money format, and which
countries the store ships to — all three of which change what you should say to a buyer. It also
confirms you are talking to the right merchant: `name` is "Shop ICON", `myshopify_domain` is
`iconaircraft.myshopify.com`.

## Finding a product

Two paths, and they are not interchangeable:

1. **Known intent, unknown handle** — `suggestSearch` (`GET /search/suggest.json?q=...`) with
   `resources[type]=product`. Fast, ranked, and it is what the store's own search box uses.
2. **Enumerate everything** — `listProducts` (`GET /products.json?limit=250`). The catalog held 43
   published products on 2026-08-22, so one page covers it. Paginate with `page` and stop when
   `products` comes back empty; there is no total-count field.

To browse the way a shopper does, use `listCollections` (13 collections: hats, mens, womens,
junior-aviators, cold-weather-gear, sport-flying-manuals, publishing-books, new-arrivals and more)
and then `listCollectionProducts` for the one you want.

## Reading a product correctly

`getProduct` (`GET /products/{handle}.json`) returns the full record. Three things to get right:

- **Price lives on the variant, not the product.** Read `variants[].price`. It is a decimal string
  in the store currency — `"30.00"` is $30.00. This is NOT the same representation the MCP surface
  uses; see the caution below.
- **Availability lives on the variant too.** `variants[].available` is the boolean to check before
  telling a buyer something is in stock. `compare_at_price` being non-null means the variant is on
  sale.
- **Options are positional.** `options[0].values` indexes `variants[].option1`, `options[1]` indexes
  `option2`, and so on. There is no id joining them — match by position or you will report the
  wrong size.

## Caution: two money representations on one store

The storefront JSON surface returns prices as decimal strings (`"30.00"`). The UCP/MCP surface on
the same store returns integers in ISO 4217 minor units paired with a currency code
(`{"amount": 3000, "currency": "USD"}`). If you read a price from one surface and pass it to the
other, you will be off by a factor of 100. Convert deliberately.

## Errors

There is no error contract here. An unknown handle returns HTTP 404 with `content-type:
application/json` and an **HTML body** — parsing it will throw. Check the status code before you
parse. See `errors/icon-aircraft-problem-types.yml`.

## Where this skill stops

It does not create a cart, start a checkout or take payment. The storefront JSON surface has no
write path at all; those live on the UCP/MCP endpoint and are covered by
`icon-aircraft-transact-with-approval`.
