---
name: read-icon-aircraft-news
description: >-
  Pull ICON Aircraft news posts, marketing pages and media from the corporate site's WordPress REST
  content API, with the pagination and PII rules that surface requires.
api: icon-aircraft:icon-aircraft-content-api
operations:
  - listPosts
  - getPost
  - listPages
  - getPage
  - listMedia
  - listCategories
  - listTags
  - searchContent
  - listTypes
generated: '2026-08-22'
method: generated
source: >-
  Grounded in openapi/icon-aircraft-content-api-openapi.yml, derived from the live route index at
  https://www.iconaircraft.com/wp-json/ and verified by anonymous probe on 2026-08-22.
---

# Read ICON Aircraft company content

Base: `https://www.iconaircraft.com/wp-json`. Anonymous, read-only, WordPress 6.7.1 core REST API.
The site advertises it itself with a `rel="https://api.w.org/"` link on every page.

## Do not touch the users collection

`GET /wp/v2/users` answers anonymously with 8 identifiable staff records. It is a WordPress core
default, not a misconfiguration unique to ICON, but it is still real people. **Do not call it, do
not resolve `Post.author`, do not quote or store a name.** If you need attribution, say "ICON
Aircraft".

## Finding things

- `searchContent` (`GET /wp/v2/search?search=...`) is the fastest way in — one call across posts and
  pages, returning `{id, title, url, type, subtype}` stubs. 242 items were searchable on 2026-08-22.
  Then fetch the full record with `getPost` or `getPage` using the returned `id`.
- `listPosts` for the news archive, newest first — 159 published posts. Filter with `categories` or
  `tags` (5 categories: engineering, events, podcast, training, uncategorized; 38 tags).
- `listPages` for marketing and policy pages — 94 of them, hierarchical. Walk `parent` to reconstruct
  the tree; `parent: 0` is a root page.

## Pagination

`page` and `per_page`, and **`per_page` is capped at 100** — asking for more returns HTTP 400
`rest_invalid_param` with the exact constraint in `data.details.per_page.message`. Read the totals
from the response headers, not the body:

- `X-WP-Total` — items matching the query
- `X-WP-TotalPages` — pages available
- `Link` — RFC 8288 `rel="next"` / `rel="prev"`

Both `X-WP-*` headers are listed in `access-control-expose-headers`, so they are readable from a
browser too.

## Fewer round trips

- `_embed=true` inlines featured media and terms into an `_embedded` member — one request instead of
  three. (It also inlines the author, which is the PII you are avoiding; ignore that member.)
- `_fields=id,title,link,date` trims the response. Post bodies carry full rendered HTML and are
  large; if you only need a headline list, ask for one.

## Media

`listMedia` covers 1,709 items — A5 photography, video posters and documents. Use
`media_type=image` to filter. `source_url` is the original; `media_details.sizes` holds the
generated renditions, which is usually what you want rather than a full-resolution original.

## Errors

`{"code": "...", "message": "...", "data": {"status": ...}}` — not RFC 9457. The codes you will
actually hit are `rest_invalid_param` (400), `rest_post_invalid_id` (404), `rest_no_route` (404) and
`rest_forbidden_context` (401, from asking for `context=edit`). Full catalog in
`errors/icon-aircraft-problem-types.yml`.

## What is not here

No aircraft data. The A5's configuration, options and pricing exist only as prose inside page HTML —
there is no product entity on this surface. The site registers one custom type, `work`, which is
REST-visible but held 0 published entries when probed.
