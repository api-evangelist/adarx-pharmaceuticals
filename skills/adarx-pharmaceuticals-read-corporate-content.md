---
name: Read the ADARx corporate site as structured content
description: Walk the ADARx Pharmaceuticals page hierarchy, retrieve company/science/pipeline copy, and read the schema.org JSON-LD the site publishes.
api: openapi/adarx-pharmaceuticals-content-openapi.yml
operations: [listPages, getPage, searchContent, getSeoHead, listPostTypes, listTaxonomies, getRouteIndex]
generated: '2026-08-06'
method: generated
---

# Read the ADARx corporate site as structured content

`https://www.adarx.com/wp-json` exposes the 18 published corporate pages as JSON, with a real
parent/child hierarchy. Use it instead of scraping HTML.

## Authentication

None.

## Steps

1. **Confirm the contract before you rely on it.** Call `getRouteIndex` — `GET /` on the
   `/wp-json` base. It returns site identity, the registered namespaces and the full route table.
   ADARx does not version this surface; its shape changes when WordPress or a plugin is upgraded,
   with no announcement, so re-read the index rather than trusting a cached route list.
   `listPostTypes` (`GET /wp/v2/types`) and `listTaxonomies` (`GET /wp/v2/taxonomies`) tell you
   whether any new editorial type has appeared — on 2026-08-06 there was none beyond WordPress
   core.

2. **Pull the whole tree in one call.**

   ```
   GET /wp/v2/pages?per_page=100&_fields=id,slug,link,parent,menu_order,title
   ```

   18 pages. `parent` is `0` for a top-level page, otherwise the id of the parent page. The
   observed tree on 2026-08-06:

   - Company (25) → Company Overview (27), Team (24), Investors (46)
   - Science (12) → Science Overview (14), Technology (15), Publications (969)
   - Home (298), Pipeline (10), Patients (20), News (6), Careers (30), Contact (7)
   - Terms of Use (804), Privacy Policy (3), Clinical Trial-Related Website Privacy Notice (815),
     Privacy Notice for Applicants, Employees and Contractors (807)

3. **Fetch the copy.** Call `getPage` — `GET /wp/v2/pages/{id}` — and read `content.rendered`.
   Bodies are populated. The pages worth reading:

   | id | page | what is in it |
   |---|---|---|
   | 27 | Company Overview | the one-paragraph company definition |
   | 14 | Science Overview | the RNAi/RNA-editing framing and design attributes |
   | 15 | Technology | RNAi, RNA editing and delivery platform detail |
   | 10 | Pipeline | the full program table (see caveat below) |
   | 20 | Patients | the five active clinical trials with ClinicalTrials.gov links |
   | 46 | Investors | the investor syndicate, as logos with outbound links |
   | 7 | Contact | address, phone, info@ and ir@ addresses |

4. **Read the structured data.** Call `getSeoHead` —
   `GET /yoast/v1/get_head?url=https%3A%2F%2Fwww.adarx.com%2F` — for the schema.org JSON-LD
   `@graph` of any URL on the site. The same graph is on every REST object as
   `yoast_head_json.schema`. Expect `WebPage`, `ImageObject`, `BreadcrumbList` and `WebSite` nodes.
   **There is no `Organization` node**, so do not expect to resolve company identity, address or
   `sameAs` from structured data — read page 7 instead.

5. **Search when you do not know the id.** `GET /wp/v2/search?search=technology` spans posts and
   pages and returns `{id, title, url, type, subtype}`.

## Caveats that will bite you

- **The pipeline table is HTML, not data.** Page 10 renders ADX-038 (agazisiran, CFB → IgAN/C3G,
  GA, PNH), ADX-324 (onvuzosiran, PKK → HAE), ADX-626 (FXI → thrombosis), ADX-077 (obesity),
  ADX-199 (APP → neurodegeneration) and several undisclosed extrahepatic programs as an Elementor
  table inside `content.rendered`. There is no custom post type, no taxonomy and no ACF projection
  for it. You must parse the HTML.
- **The investor list is images.** Page 46 carries the syndicate as `<img alt="…-logo">` elements
  with outbound `href`s. The names are recoverable from the `alt` attributes and the link targets,
  not from any field.
- **No caching headers.** No ETag, no Last-Modified, no Cache-Control. Use each object's
  `modified` / `modified_gmt` to decide whether to refetch.
- **No rate-limit signalling.** Nothing advertises a limit. Be conservative.

## Error handling

The WordPress envelope `{code, message, data:{status}}`, not RFC 9457. `rest_post_invalid_id` (404)
for a bad page id; `rest_invalid_param` (400) for `per_page` outside 1-100; 401 `rest_forbidden` on
`/wp/v2/settings`, `/wp/v2/menus`, `/wp/v2/themes` and the whole `wp-abilities/v1` namespace —
those are authenticated-only and are not part of the public contract. See
`errors/adarx-pharmaceuticals-problem-types.yml`.
