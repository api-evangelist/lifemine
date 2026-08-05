---
name: Track LifeMine company news
description: Pull LifeMine press releases, publications and in-the-news coverage from the company's public WordPress REST API, filter by category and date, and keep a watch for new items.
api: openapi/lifemine-content-openapi.yml
operations:
  - getWpV2Posts
  - getWpV2PostsById
  - getWpV2Categories
  - getWpV2Search
generated: '2026-08-04'
method: generated
source: openapi/lifemine-content-openapi.yml
---

# Track LifeMine company news

LifeMine is a clinical-stage biopharmaceutical company. It publishes no developer documentation,
but its WordPress site serves a fully anonymous read API at `https://lifeminetx.com/wp-json`.
That is where its news lives as structured JSON.

**Base URL:** `https://lifeminetx.com/wp-json`
**Auth:** none. Do not send credentials — reads are public.
**Pacing:** the site's robots.txt asks for `Crawl-delay: 10`. There is no published rate limit and
no `Retry-After` contract, so leave ~10 seconds between requests and back off hard on any non-2xx.

## 1. Learn the news taxonomy first

Call `getWpV2Categories` — `GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count`.

Four categories existed on 2026-08-04, and knowing which is which is the difference between
"LifeMine announced" and "someone wrote about LifeMine":

| slug | count | what it actually is |
|---|---|---|
| `press-releases` | 9 | LifeMine's own announcements — the authoritative voice |
| `in-the-news` | 13 | earned media written by third parties, republished here |
| `publications` | 1 | peer-reviewed science |
| `expedition-academy` | 1 | internal thought leadership |

Never treat `in-the-news` as a company statement. Attribute it to the original outlet.

## 2. List the news

Call `getWpV2Posts` — `GET /wp/v2/posts`.

Always send `_fields`. A full post object is 28 properties of mostly rendered HTML, and you will
burn context for nothing:

```
GET /wp/v2/posts?per_page=20&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt,categories
```

Useful parameters, all read from the host's own route schema:

- `categories=<id>` / `categories_exclude=<id>` — scope to press releases only.
- `after=2025-01-01T00:00:00` and `before=...` — date windows (ISO 8601).
- `modified_after=...` — catch edits to existing items, not just new ones.
- `search=<term>` — full-text within posts.
- `page` / `per_page` (max 100) / `offset`.

## 3. Page correctly

Read `X-WP-Total` and `X-WP-TotalPages` from the response headers rather than paging until you get
an empty array — both are exposed cross-origin via `Access-Control-Expose-Headers`. A `Link`
header carries `rel="next"`. There were 24 posts total on 2026-08-04.

Requesting a page beyond `X-WP-TotalPages` returns `400 rest_invalid_page_number`.

## 4. Fetch a single item

Call `getWpV2PostsById` — `GET /wp/v2/posts/{id}`. The full body is in `content.rendered` as HTML.
`excerpt.rendered` is usually enough for summarisation.

## 5. Cross-resource search

Call `getWpV2Search` — `GET /wp/v2/search?search=<term>&per_page=20`. This spans posts, pages and
the `team` custom post type, returning typed stubs (`id`, `title`, `url`, `type`, `subtype`). It
does not return bodies — follow up with `getWpV2PostsById` or `getWpV2PagesById`.

## 6. Watch for new items

Poll `getWpV2Posts` with `orderby=date&order=desc&per_page=5&_fields=id,date,modified,title,link`
and keep the highest `date` you have seen. To catch corrections as well as new posts, keep the
highest `modified` instead and query `modified_after`.

An RSS alternative exists at `https://lifeminetx.com/feed/` if you would rather not poll JSON.

## Errors

Responses are **not** RFC 9457. The envelope is
`{"code": "...", "message": "...", "data": {"status": ...}}`.

- `401 rest_forbidden` — you touched a credentialed route. Stay on published reads.
- `404 rest_no_route` — wrong path *or* wrong method.
- `400 rest_invalid_param` — check `data.params`; usually `per_page` over 100 or a bad `orderby`.

Full catalog: `errors/lifemine-problem-types.yml`.

## Cautions

- Responses carry `x-robots-tag: noindex`. This JSON is public but deliberately not indexed —
  do not republish it as if it were a syndication feed.
- Collections are cached ~10 minutes (`Cache-Control: max-age=600`). Polling faster gains nothing.
- Content is corporate and press material. It is **not** clinical or regulatory data, and nothing
  here should be treated as medical information.
