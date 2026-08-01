---
name: Track Arbor Biotechnologies press releases
description: Pull Arbor Biotechnologies' dated press releases and company announcements from the corporate site's public WordPress REST API, incrementally and without credentials.
api: openapi/arbor-biotechnologies-content-openapi.yml
operations: [listPosts, getPost, listCategories, getCategory]
generated: '2026-07-31'
method: generated
---

# Track Arbor Biotechnologies press releases

Arbor Biotechnologies is a gene-editing company; its news flow is regulatory designations,
partnership announcements, financings and conference presentations. All of it is published as
WordPress posts and is readable with no credential.

## Before you start

- Base URL: `https://arbor.bio/wp-json`
- **No authentication.** Do not look for an API key — Arbor issues none. If a route returns
  401, it is permanently closed to third parties; do not retry.
- **Send a browser `User-Agent`.** The site's WAF answers non-browser agents with HTTP 406
  while still returning the correct JSON body. A browser UA gets a clean 200.
- No rate limit is published and no `RateLimit` headers are returned. A WAF fronts the API, so
  keep concurrency to 1 and pace requests.

## Steps

1. **Find the press release category.** Call `listCategories`
   (`GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count`). The category with slug
   `press-release` is the content one (57 posts as of 2026-07-31); the numeric slugs
   (`2018`…`2026`) are year archives and will double-count if you include them.

2. **List posts, newest first.** Call `listPosts`:

   ```
   GET /wp/v2/posts?per_page=100&orderby=date&order=desc
       &_fields=id,date,modified,slug,link,title,excerpt,categories
   ```

   Always pass `_fields`. A full post object embeds rendered HTML plus a `yoast_head` SEO
   block and a `yoast_head_json` schema.org graph, which is roughly two orders of magnitude
   larger than you need for a listing.

3. **Page correctly.** Read `X-WP-Total` and `X-WP-TotalPages` from the response headers, or
   follow the RFC 8288 `Link` header's `rel="next"`. `per_page` is capped at 100 — a larger
   value returns `400 rest_invalid_param` with the reason in `data.details.per_page`. The
   whole corpus is 59 posts, so one page of 100 covers it today.

4. **Run incrementally on later passes.** Pass `after` with the ISO 8601 timestamp of your
   last sync:

   ```
   GET /wp/v2/posts?after=2026-06-01T00:00:00&orderby=date&order=asc&_fields=id,date,modified,slug,title
   ```

   Track `modified` as well as `date` — Arbor edits published releases, and `after` filters on
   publication date only.

5. **Fetch full text only for what is new.** Call `getPost` (`GET /wp/v2/posts/{id}`) per new
   id. `content.rendered` is HTML; `excerpt.rendered` is a short HTML summary. For structured
   article metadata (headline, datePublished, publisher) read `yoast_head_json` instead of
   parsing the body.

6. **Resolve the hero image if you need it.** `featured_media` is a media id, or `0` when
   absent. Rather than a second call, add `_embed` to the `getPost` request and read
   `_embedded['wp:featuredmedia'][0].source_url`.

## Things that will surprise you

- `author` is an integer id that you cannot resolve — `/wp/v2/users` returns
  `401 rest_user_cannot_view`. There is no byline.
- Tags are always empty. The `post_tag` taxonomy is registered but carries zero terms.
- Comments are permanently disabled (`403 rest_comment_disabled`).
- Single-object responses carry `Cache-Control: no-store`, so do not cache aggressively even
  though the content is static in practice.

## Error handling

Branch on the `code` field, never the `message`. See
`errors/arbor-biotechnologies-problem-types.yml`.

| Code | Status | What to do |
|---|---|---|
| `rest_invalid_param` | 400 | Read `data.details`; usually `per_page` outside 1–100. |
| `rest_post_invalid_id` | 404 | The id does not exist or is not public. Ids are per-post-type — re-resolve from `listPosts`. |
| `rest_no_route` | 404 | Wrong path. Check it against `https://arbor.bio/wp-json`. |
| any `401` | 401 | Terminal. No credential exists. Stop. |

## Alternative

For a simple newest-first feed with no pagination logic, `https://arbor.bio/feed/` is a valid
RSS 2.0 channel of the same posts. Use the API when you need ids, categories, incremental
`after` filtering or structured metadata.
