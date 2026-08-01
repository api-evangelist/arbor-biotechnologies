---
name: Search and harvest the Arbor Biotechnologies site
description: Answer questions about Arbor Biotechnologies' pipeline, platform and company from the corporate site's public WordPress REST API — searching across all content and fetching structured pages instead of scraping HTML.
api: openapi/arbor-biotechnologies-content-openapi.yml
operations: [searchContent, getPost, listPages, getPage, listTypes, listTaxonomies, getOembed, getYoastHead]
generated: '2026-07-31'
method: generated
---

# Search and harvest the Arbor Biotechnologies site

Use this when you need to answer a question about Arbor — what is in the pipeline, who runs
the company, what a program targets — rather than track news chronologically. The whole site
is retrievable as JSON, which is more reliable than scraping the rendered pages.

## Before you start

- Base URL: `https://arbor.bio/wp-json`
- No authentication. Send a browser `User-Agent` (the WAF returns 406 otherwise).
- Keep concurrency to 1; no rate limit is published.

## Steps

1. **Search first.** `searchContent` is the cheapest entry point — it spans posts and pages
   and returns lightweight hits:

   ```
   GET /wp/v2/search?search=hyperoxaluria&per_page=20
   ```

   Each hit is `{id, title, url, type, subtype, _links}`. `title` here is a plain string, not
   the `{rendered: ...}` object the full objects use. 60 objects were searchable as of
   2026-07-31.

2. **Resolve a hit to its full object.** Use `_links.self[0].href` from the hit, or branch on
   `subtype`: `post` → `getPost` (`/wp/v2/posts/{id}`), `page` → `getPage`
   (`/wp/v2/pages/{id}`). Adding `_embed` to the search request inlines the full objects under
   `_embedded` and saves the round trip.

3. **Go straight to a known page when you can.** There are only 11 pages, and they are stable.
   Call `listPages` (`GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title`) once and cache
   the slug→id map. The slugs are:

   | Slug | What it holds |
   |---|---|
   | `pipeline` | The program table — ABO-101/103 (liver, LNP) and ABO-202/203/204/206 (CNS, AAV), stages, and partners |
   | `what-we-do` | The editing platform: knockdown+, nuclease excision, compact RT editing |
   | `who-we-are` | Founders, leadership, board, investors, partnerships |
   | `clinical-trial` | The active clinical program |
   | `inside-arbor` | Values, DEI, community, benefits, careers |
   | `stay-updated` | The press release index |
   | `get-in-touch` | Contact |
   | `privacy-policy`, `terms-of-use`, `site-map`, `sample-page` | Policies, map, home |

   Then `getPage` and read `content.rendered`. It is HTML — the pipeline table in particular
   is a rendered table, so parse it as markup rather than expecting structured fields. The
   `acf` field is present on every object but is empty on this site; do not expect the pipeline
   to arrive as custom fields.

4. **Prefer structured metadata over parsing prose.** Every post and page carries
   `yoast_head_json` with a schema.org `@graph` (Organization, WebSite, WebPage, Article) —
   headline, description, datePublished, publisher. `getYoastHead`
   (`GET /yoast/v1/get_head?url=<any arbor.bio URL>`) returns the same for a URL you only have
   as a link. `getOembed` (`GET /oembed/1.0/embed?url=...`) gives title, provider and thumbnail
   for citation cards.

5. **Understand what you are looking at.** `listTypes` and `listTaxonomies` describe the
   content model. Only `post`, `page`, `attachment` and `wpcf7_contact_form` are anonymously
   readable of the 15 registered types; the rest are editor and plugin constructs. See
   `data-model/arbor-biotechnologies-data-model.yml`.

## Limits you will hit

- **Media is mostly invisible.** `listMedia` reports `X-WP-Total: 394` but returns only 4
  items to an anonymous caller. Individual files remain fetchable at their
  `source_url` under `/wp-content/uploads/`, which `robots.txt` explicitly Allows.
- **No author names.** `/wp/v2/users` is 401.
- **No leadership or pipeline API.** Team bios and the program table exist only as rendered
  HTML inside `who-we-are` and `pipeline`. There is no structured endpoint for them, and you
  must not synthesise one.

## Accuracy rules for this content

Arbor's content is clinical and regulatory. Everything in the pipeline is **investigational** —
ABO-101 and ABO-202/203/204/206 are development-stage candidates, not approved medicines. An
orphan drug designation is a regulatory status, not an approval. Partner-run ex vivo programs
(Vertex, Allogene, Edigene, Chiesi) are explicitly not developed by Arbor internally. Never
present any of this as medical advice or as an approved therapy, and always carry the program
stage alongside the program name.

## Error handling

Branch on `code`, not `message`. `rest_post_invalid_id` (404) means the id is wrong for that
post type — ids are not global, so re-resolve from a list or search call. Any 401 is terminal;
no credential exists. Full table in `errors/arbor-biotechnologies-problem-types.yml`.
