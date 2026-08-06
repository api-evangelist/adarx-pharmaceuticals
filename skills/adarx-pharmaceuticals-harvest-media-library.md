---
name: Harvest the ADARx media library
description: Enumerate and dereference the ADARx Pharmaceuticals media library — logos, team headshots, pipeline graphics and investor marks — including the responsive derivatives.
api: openapi/adarx-pharmaceuticals-content-openapi.yml
operations: [listMedia, getMediaItem, listPosts, getPost]
generated: '2026-08-06'
method: generated
---

# Harvest the ADARx media library

`GET https://www.adarx.com/wp-json/wp/v2/media` exposes 105 attachments anonymously (`X-WP-Total:
105` on 2026-08-06). This is the site's whole image corpus: the corporate logo, team and board
headshots, pipeline graphics, the AbbVie partnership marks and the investor-syndicate logos.

## Authentication

None.

## Steps

1. **Page the collection with a sparse fieldset.**

   ```
   GET /wp/v2/media?per_page=100&_fields=id,slug,mime_type,media_type,source_url,alt_text,post,date
   ```

   `per_page` is capped at 100; exceeding it returns 400 `rest_invalid_param` with
   `data.details.per_page.code = rest_out_of_bounds`. Read `X-WP-Total` / `X-WP-TotalPages` from
   the headers and follow `Link: rel="next"`.

2. **Filter by name when you know what you want.** `?search=logo` narrows to the logo set. The
   corporate mark is id 52, `https://www.adarx.com/wp-content/uploads/2024/11/adarx-logo.png`.

3. **Get the derivatives.** `getMediaItem` — `GET /wp/v2/media/{id}` — returns `source_url` (the
   original) plus `media_details.sizes`, a map of WordPress-generated responsive variants each with
   its own `source_url`, `width` and `height`. Pick a size rather than downloading originals.

4. **Attribute an asset to its article.** `post` on a media item is the parent post/page id, or `0`
   when the asset is unattached. Going the other way, `featured_media` on a post is the hero image
   id, and `GET /wp/v2/media?parent={postId}` returns everything attached to that post.

5. **Recover names from `alt_text`.** The investor-syndicate logos on page 46 and the team
   headshots carry meaningful `alt` values (`bain-logo`, `orbimed-logo`, `venrock-logo`, …). This is
   the only place those relationships are machine readable — there is no investor entity, no
   taxonomy and no structured team record anywhere in this API.

## Etiquette

- These are copyrighted corporate assets on a third-party site. Fetch what you need, cache it, and
  respect the site's terms at https://www.adarx.com/terms-of-use/.
- There is no rate-limit header and no documented limit. Page serially, not in parallel bursts.
- No ETag/Last-Modified is returned; use each item's `modified` field to decide whether to refetch.

## Error handling

WordPress envelope `{code, message, data:{status}}`, not RFC 9457. `rest_post_invalid_id` (404) for
a bad media id, `rest_invalid_param` (400) for a bad `per_page`. Media **write** routes
(`POST /wp/v2/media`, `/wp/v2/media/{id}/edit`) require Application Password credentials and return
401 anonymously — do not attempt them. See `errors/adarx-pharmaceuticals-problem-types.yml`.
