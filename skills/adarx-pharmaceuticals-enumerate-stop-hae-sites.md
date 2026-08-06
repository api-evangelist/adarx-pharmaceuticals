---
name: Enumerate the STOP-HAE Phase 3 trial sites
description: Read the custom `clinic` collection on the ADARx STOP-HAE patient site, and understand exactly what it does and does not expose.
api: openapi/adarx-pharmaceuticals-stop-hae-openapi.yml
operations: [getTrialRouteIndex, listTrialClinics, getTrialClinic, listTrialPostTypes, listTrialPages, getTrialPage]
generated: '2026-08-06'
method: generated
---

# Enumerate the STOP-HAE Phase 3 trial sites

`https://stophae.com` is ADARx Pharmaceuticals' patient-recruitment site for **STOP-HAE**, the
Phase 3 randomized, double-blind, placebo-controlled study of ADX-324 (onvuzosiran) in hereditary
angioedema. Its WordPress install calls itself "ADARX1 Patient Website" and registers a **custom
post type** the corporate site does not have: `clinic`.

## Authentication

None. Everything below is anonymous.

## Steps

1. **Confirm the custom type still exists.** Call `getTrialRouteIndex` (`GET /` on the
   `https://stophae.com/wp-json` base) or `listTrialPostTypes` (`GET /wp/v2/types`) and look for
   the `clinic` entry — label "Clinics", `rest_base: clinic`, `rest_namespace: wp/v2`. It exists
   only because a plugin/theme registers it, and ADARx publishes no deprecation policy, so it can
   disappear on an upgrade without notice. Do not assume it.

2. **List the sites.**

   ```
   GET /wp/v2/clinic?per_page=100&orderby=menu_order&order=asc&_fields=id,slug,title,link,menu_order,modified
   ```

   50 published records as of 2026-08-06 (`X-WP-Total: 50`). `orderby=menu_order` reproduces the
   ordering the site UI uses. Each record's identity is a **trial site code** carried in both
   `slug` and `title.rendered` — e.g. `801-01`.

3. **Fetch one.** `getTrialClinic` — `GET /wp/v2/clinic/{id}`.

4. **Know what you will NOT get.** This is the single most important thing about this collection:
   `acf` is an **empty array on every record**. The Advanced Custom Fields payload that presumably
   holds each site's address, city, country, principal investigator and contact number is not
   projected into REST. `categories` is empty on every record too, even though the `category`
   taxonomy is registered against the `clinic` type. So the API gives you **site codes,
   permalinks and dates — and no geography**.

   If you need locations, you have two honest fallbacks:
   - fetch the record's `link` and parse the rendered page, or
   - use ClinicalTrials.gov, which is the authoritative registry. The ADARx studies linked from
     https://www.adarx.com/patients/ are NCT07081503, NCT06989359, NCT06990269, NCT05876312 and
     NCT05691361.

   Do not synthesise locations from site codes. A code like `801-01` encodes nothing you can
   verify.

5. **Read the patient pages if you need context.** `listTrialPages` returns four —
   Home (2), HCP (91), Prescreener (201) and Privacy Policy (211). `getTrialPage` fetches the body.
   The prescreener at https://stophae.com/prescreener/ is a rendered form, not an API; there is no
   REST endpoint for eligibility screening and you must not attempt to submit to one.

6. **Media is large and mostly duplicated.** `listTrialMedia` reports 592 attachments for a
   four-page site — the WPML multilingual configuration (WPML 4.9.5, ~27 language slots) duplicates
   assets per language. Page it, do not pull it all.

## Boundaries

- The `wpml/v1`, `wpml/st/v1`, `wpml/tm/v1`, `wpml/ate/v1` and `otgs/installer/v1` namespaces
  return their namespace index anonymously but every functional route beneath them is
  administrative. Do not probe them.
- `wp-abilities/v1` — the WordPress Abilities API, an agent-facing capability registry — is
  registered but returns 401 `rest_forbidden` on every endpoint. There is no agent surface here.
- `/wp/v2/posts` is registered but empty (`X-WP-Total: 0`).
- This is a patient-recruitment site for a live clinical trial. Read it; do not write to it, do not
  attempt authentication, and do not treat anything retrieved here as medical advice or as a
  complete site list — ClinicalTrials.gov is the registry of record.

## Error handling

WordPress envelope `{code, message, data:{status}}`, not RFC 9457. `rest_post_invalid_id` (404),
`rest_invalid_param` (400) for `per_page` outside 1-100, `rest_forbidden` (401) for anything
administrative. See `errors/adarx-pharmaceuticals-problem-types.yml`.
