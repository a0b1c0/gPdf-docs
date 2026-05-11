# gPdf Changelog

This is a mirror of the canonical changelog. The link-rich, browseable view is at [gpdf.com/changelog](https://gpdf.com/changelog/).

## v2026.05 — May 2026: GEO infrastructure, font subset hardening, expanded locale matrix

**Released:** 2026-05-10  
**Kind:** minor

> GEO infrastructure release: adds /vs/ and /use-cases/ landing-page collections, automates /llms.txt + JSON Feed, scales the marketing site to 20 locales, and tightens CJK / Cyrillic / Vietnamese font subsets.

### Highlights

- Site automation: /llms.txt, /robots.txt, RSS XML, JSON Feed v1.1 and the homepage Pillars JSON-LD now share one typed source-of-truth and auto-update on every content edit.
- New /vs/<competitor>/ comparison landing pages with typed verdict + spec table + JSON-LD Article/Review schemas.
- New /use-cases/<vertical>/ pages with typed job-to-be-done, capability bullets and embedded sample requests.
- Marketing site scaled from 4 → 20 locales (Ethnologue top-20 by speakers + EU e-invoice mandate markets) with hreflang fan-out.
- Font cascade tightened: Inter / JetBrains Mono / Noto Sans CJK·SC/JP/KR + Devanagari + Bengali + Thai + Arabic registered, weights consolidated to 400/700, Cyrillic + Vietnamese subsets added so ru/uk/vi pages no longer fall through to system-ui.

### All changes

- ✨ **Added** — Dynamic /llms.txt route auto-built from the blog, docs, /vs/ and /use-cases/ collections — adding content updates the file on next deploy with no route edits.
- ✨ **Added** — Dynamic /blog/feed.json (JSON Feed v1.1) sibling to the existing /blog/rss.xml; both are advertised via <link rel="alternate"> on /blog/.
- ✨ **Added** — /vs/ comparison content collection + page template with Article + Review JSON-LD.
- ✨ **Added** — /use-cases/ vertical content collection + page template with Article (abstract, audience, featureList) JSON-LD.
- ✨ **Added** — /changelog/ collection + page template + RSS / JSON Feed.
- ⚡ **Improved** — Homepage Pillars block (3 ms p50, $5/100K, PDF/A profiles, 4 ms isolate lifetime) now emits SoftwareApplication.additionalProperty PropertyValue JSON-LD, sharing one source-of-truth with the visible card grid.
- ⚡ **Improved** — robots.txt now advertises sitemap, /llms.txt, /blog/rss.xml and /blog/feed.json in a comment block so AI crawlers can find every machine-readable descriptor from one entry point.
- ⚡ **Improved** — RSS feed adds atom:link rel="self" + atom:link rel="alternate" pointing to the JSON Feed sibling.
- ⚡ **Improved** — Per-article OG SVG cards (1200×630) generated at build time for every blog × locale combination; verified rendering on Twitter/X, LinkedIn, Slack, Discord, Bluesky, Telegram, Mastodon, Apple Messages.
- ⚡ **Improved** — i18n locale set scaled to 20 — adds bn, pt, ru, ja, id, de, fr, tr, ko, vi, it, pl, nl, th, uk on top of en/zh/es/hi/ar.
- ⚡ **Improved** — Font subsets cover Cyrillic + Cyrillic-ext + Vietnamese for Inter and JetBrains Mono so ru/uk/vi native names render in-brand instead of falling through to proprietary OS fonts.
- ⚡ **Improved** — CJK weights consolidated to [400, 700] — saves 50–150 KB per script per visitor by dropping the unused middle weight.
- 🐛 **Fixed** — Locale fallback for /zh/<page>: pages with no explicit title/description now read seo.fallback.* in the visitor's locale instead of bleeding English copy into <head>.
- 🐛 **Fixed** — Inter Latin woff2 is now preloaded — eliminates the ~50–500 ms FOUT where Astro's 134%-wide Arial fallback rendered the H1 oversized before swapping to real Inter.

---

## Highlights in detail

This release is mostly invisible to API callers — the gPdf engine itself is unchanged — but ships the GEO/SEO infrastructure that makes the marketing site discoverable and AI-crawler-friendly.

### Why this work matters

By 2026, a meaningful share of "what's the best PDF API for X" traffic comes from generative engines (ChatGPT, Claude, Perplexity, Gemini) rather than blue-link search. Those engines preferentially quote sites that publish:

1. **Curated machine-readable descriptors** — `llms.txt` with verbatim facts and structured links to deeper content.
2. **Typed JSON-LD** — `SoftwareApplication.additionalProperty` for capability tables, `Article.abstract` for verticals, `Review.reviewBody` for comparisons.
3. **Stable streamable feeds** — RSS for the long tail of feed readers, JSON Feed v1.1 for AI agents that prefer structured streams over scraping HTML.

This release ships all three.

### What changed for content authors

Adding any of the following now requires only a single Markdown file — the route, sitemap entry, llms.txt entry and (for blog/changelog) feed entry follow automatically on the next deploy:

| Drop a file at… | …and you get… |
|---|---|
| `src/content/blog/<locale>/<slug>.md` | `/blog/<slug>/` + RSS + JSON Feed + llms.txt entry + per-locale OG SVG |
| `src/content/vs/<slug>.md` | `/vs/<slug>/` + Article+Review JSON-LD + llms.txt comparison entry |
| `src/content/use-cases/<slug>.md` | `/use-cases/<slug>/` + Article (abstract+audience+featureList) JSON-LD + llms.txt vertical entry |
| `src/content/changelog/<slug>.md` | `/changelog/<slug>/` + RSS + JSON Feed + llms.txt release entry |
| `src/content/docs-source/<slug>.<locale>.md` | `/docs/<slug>/` + llms.txt docs entry |

### What didn't change

The gPdf rendering engine, API surface, error codes, pricing and rate limits are all unchanged in this release. Existing API integrations need no changes.

### Looking ahead

The next release line (v2026.06) will focus on `/use-cases/` content depth (e-invoice, payslips, statements, tickets) and on translating the public docs (api-reference, template-api, e-invoice-api) into zh, es, de, ja so the docs URL fan-out matches the marketing site's locale matrix.
