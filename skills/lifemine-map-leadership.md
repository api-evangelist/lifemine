---
name: Map LifeMine leadership and board
description: Read LifeMine's leadership, founders and board of directors as structured records from the `team` custom post type, grouped by their taxonomy, with biographies and headshots.
api: openapi/lifemine-content-openapi.yml
operations:
  - getWpV2Team
  - getWpV2TeamById
  - getWpV2TeamCategories
  - getWpV2Media
generated: '2026-08-04'
method: generated
source: openapi/lifemine-content-openapi.yml
---

# Map LifeMine leadership and board

Most company sites bury their people in HTML. LifeMine registers a **custom post type** called
`team`, which means its leadership, founders and board are available as structured JSON records
with a real taxonomy — the single most useful thing on this provider's API surface.

**Base URL:** `https://lifeminetx.com/wp-json`
**Auth:** none.
**Observed 2026-08-04:** 15 people, 3 groups.

## 1. Get the groups

Call `getWpV2TeamCategories` — `GET /wp/v2/team_categories?per_page=100&_fields=id,name,slug,count`.

| slug | count |
|---|---|
| `leadership` | 10 |
| `founders` | 3 |
| `board-of-directors` | 7 |

Those counts sum to 20 against 15 people because **a person can hold more than one term** — the
founders also sit in leadership and on the board. Deduplicate by person `id`, never by summing
group counts, or you will overstate the size of the company's leadership.

## 2. List the people

Call `getWpV2Team` — `GET /wp/v2/team`:

```
GET /wp/v2/team?per_page=100&_fields=id,slug,link,title,team_categories,featured_media
```

`title.rendered` is the person's name and post-nominals (e.g. `Nitender Goyal, MD, MBA`) —
HTML-escaped, so decode entities before display. `team_categories` is an array of term ids that
you resolve against step 1.

Filter to one group with `team_categories=<term_id>`.

## 3. Get one person

Call `getWpV2TeamById` — `GET /wp/v2/team/{id}`. `content.rendered` holds the biography as HTML.

## 4. Resolve headshots

Each record's `featured_media` is an attachment id. Either:

- add `_embed` to the list call and read `_embedded['wp:featuredmedia']`, which avoids N+1
  requests; or
- call `getWpV2Media` — `GET /wp/v2/media?include=<id1>,<id2>&_fields=id,source_url,alt_text,media_details`.

Prefer `_embed`. There are 117 media items on the site and fetching them individually is wasteful.

## 5. Sanity-check freshness

`modified` and `modified_gmt` on each record tell you when a bio last changed. A leadership page
that has not moved in a long time is itself a signal — use `modified` rather than assuming the
roster is current.

## Errors

Same WordPress envelope as everywhere on this host: `{"code", "message", "data.status"}`.
`404 rest_post_invalid_id` means the id is not a `team` record — list the collection first.
See `errors/lifemine-problem-types.yml`.

## Cautions

- These are named individuals. This data is published by LifeMine for public consumption, but
  treat it as personal information: do not enrich, cross-reference or profile these people
  beyond what the company itself published.
- `/wp/v2/users` is also open on this host. That is the WordPress *author* surface, not the
  leadership roster — it exposes site editors, not executives. Do not confuse the two, and do not
  enumerate it.
