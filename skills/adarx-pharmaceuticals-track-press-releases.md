---
name: Track ADARx press releases and clinical data readouts
description: Poll the ADARx Pharmaceuticals content API for new press releases, clinical-trial listings, leadership changes and publications, and retrieve full article text.
api: openapi/adarx-pharmaceuticals-content-openapi.yml
operations: [listPosts, getPost, listCategories, searchContent]
generated: '2026-08-06'
method: generated
---

# Track ADARx press releases and clinical data readouts

The corporate site `www.adarx.com` publishes everything ADARx says publicly as WordPress posts,
readable anonymously at `https://www.adarx.com/wp-json`. 46 posts as of 2026-08-06, running from
2021-09-08 to 2026-06-10.

## Authentication

None. Send no credential. Every operation below returns 200 anonymously.

## Steps

1. **Learn the editorial facets.** Call `listCategories` — `GET /wp/v2/categories`. The six
   observed on 2026-08-06 and what they mean:

   | id | slug | label | count | contains |
   |---|---|---|---|---|
   | 4 | news | News | 23 | corporate press releases |
   | 5 | clinical-trials | Clinical Trials | 7 | one record per trial (STOP-HAE, the ADX-038 Phase 2s, …) |
   | 6 | leadership | Leadership | 5 | executive biographies |
   | 7 | bod | BOD | 9 | board-member biographies |
   | 9 | publications | Publications | 3 | conference abstracts and papers |
   | 1 | uncategorized | Uncategorized | 0 | — |

   Do not hardcode these ids without re-reading; they are per-install and can change.

2. **Pull an index cheaply.** Call `listPosts` with a sparse fieldset and a category filter:

   ```
   GET /wp/v2/posts?categories=4&per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title
   ```

   `_fields` matters — the unfiltered object carries a large `yoast_head` markup string plus the
   whole article body. Read `X-WP-Total` and `X-WP-TotalPages` from the **response headers**; the
   body is a bare array with no envelope, so a client that reads only the body cannot tell whether
   more pages exist. Follow `Link: <…>; rel="next"` to page.

3. **Poll for change, do not refetch.** Store the highest `modified` you have seen and poll:

   ```
   GET /wp/v2/posts?modified_after=2026-06-10T14:14:57&per_page=100&_fields=id,slug,modified,title
   ```

   There is no ETag, no Last-Modified and no Cache-Control on these responses, so `modified_after`
   is the only efficient change signal. There are also no rate-limit headers of any kind — treat
   throughput as unspecified and poll no more than hourly.

4. **Fetch the body when you need it.** Call `getPost` — `GET /wp/v2/posts/{id}`. Read
   `content.rendered` (HTML) and `excerpt.rendered`. On this install these **are populated** —
   typically 4,900-5,300 characters on a press release — so you do not need to scrape the HTML
   page. Exception: the three Publications-category posts carry empty bodies and exist only to
   title-and-link an external abstract; for those, follow `link`.

5. **Resolve the hero image.** `featured_media` is a media id, or `0` when unset. Dereference with
   `GET /wp/v2/media/{id}` and read `source_url`, or add `_embed` to the post request to inline
   `wp:featuredmedia`. The `author` relation also resolves here — `/wp/v2/users` returns 200
   anonymously on this install, unlike most corporate WordPress deployments.

6. **Search when you do not know the id.** Call `searchContent` —
   `GET /wp/v2/search?search=onvuzosiran` — which spans posts and pages and returns
   `{id, title, url, type, subtype}`. Then fetch the object by id.

## Error handling

Errors come back as `{"code": …, "message": …, "data": {"status": …}}` — the WordPress envelope,
**not** RFC 9457 problem+json. Branch on `code`, never on `message`.

- `rest_post_invalid_id` (404) — the id does not exist or is not published.
- `rest_invalid_param` (400) — read `data.details.<field>.code`; the usual cause is
  `per_page` outside 1-100.
- `rest_no_route` (404) — wrong path or wrong method; re-read the route index at `/wp-json/`.
- Any 401 (`rest_forbidden`, `rest_cannot_read`) means the route is authenticated-only and is not
  part of the public contract. Do not retry it. `/wp/v2/posts/{id}/revisions` is one of these.

Full catalogue: `errors/adarx-pharmaceuticals-problem-types.yml`.

## What this API will not give you

The ADARx **pipeline** — ADX-038 (agazisiran), ADX-324 (onvuzosiran), ADX-626, ADX-077, ADX-199
and their targets, indications and phases — is not modelled as data anywhere. It exists only as
rendered HTML inside the body of page id 10. If you need it structured, parse
`getPage` on 10, or use ClinicalTrials.gov.
