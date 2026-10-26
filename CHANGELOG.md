# gPdf Changelog

This is a mirror of the canonical changelog. The link-rich, browseable view is at [gpdf.com/changelog](https://gpdf.com/changelog/).

## v2026.08 — August 2026: Flow Document

**Released:** 2026-08-27  
**Kind:** minor

> JSON Render adds a content-first Flow Document source with reusable Page Masters, automatic pagination, typed layout groups, named styles, and Studio authoring.

### Highlights

- Choose Fixed Canvas pages or a content-first flow_document source in one stable DocumentRequest contract.
- Derive physical pages from reusable portrait, landscape, custom, and binding-aware Page Masters.
- Compose recursive columns, Stack, Grid, break, block, and bounded placed nodes with explicit pagination constraints.
- Author Page Masters, Content Segments, recursive outlines, named styles, and generated-page navigation in gPdf Studio.

### All changes

- ✨ **Added** — The capability-gated flow_document source provides automatic pages, Page Masters, ordered Content Segments, repeated regions, and logical destinations.
- ✨ **Added** — Typed columns, Stack, Grid, break, block, and placed content nodes share the Native, Worker, WASM, Wasmer, PDF, and Preview layout pipeline.
- ✨ **Added** — Named text, table, and container-frame styles support bounded inheritance plus ordered document, Page Master, Segment, and group scopes.
- ✨ **Added** — Studio adds Flow source creation, Page Master and Style libraries, recursive content editing, derived-page navigation, and runtime capability save gates.
- ⚡ **Improved** — OpenAPI JSON/YAML, the API reference, Agent Prompt, machine-readable facts, examples, and Playground now describe both Fixed Canvas and Flow Document.

---

## What's new

JSON Render now has two explicit authoring models under one `DocumentRequest`.
Fixed Canvas keeps `pages` for labels, forms, certificates, and other
coordinate-owned output. Flow Document uses `flow_document` for reports,
contracts, manuals, statements, and long invoices whose physical pages should
be derived from logical content.

Flow Document is one layout pipeline, not a second renderer or a browser-style
language. Native, Worker, Layout WASM, Wasmer, Studio Preview, and PDF replay
share the same measured fragments, generated pages, resource identities, and
logical destinations.

### Page Masters and sections

A Page Master owns page size, portrait or landscape orientation, physical or
binding-aware margins, repeated header/footer/layer regions, and optional body
columns. Ordered Content Segments select a Page Master and may reset or continue
page numbering. Generated continuation pages remain derived output and are
never serialized back into source JSON.

### Typed layout and pagination

Flow content uses typed `block`, `columns`, `stack`, `grid`, `break`, and
`placed` nodes. Pagination constraints include keep-together, keep-with-next,
explicit breaks, and bounded Text line constraints. Logical source order owns
PDF/UA reading order; links target declared logical destinations instead of
guessing generated physical page numbers.

### Styles and Studio

Named styles target Text, Table, or Container frame properties. Bounded
same-target inheritance and ordered scopes provide reusable document styling
without introducing CSS or an untyped property map. Studio edits this same
public source model through Page Master and Style libraries, a recursive
content outline, direct typed inspectors, and generated-page source mapping.

### Availability and compatibility

Use Flow Document only when the selected runtime `/health` response advertises
`layout.flow_document`. The full Worker, Server, standalone, and Wasmer
profiles can expose the capability; `profile-label` / `pdn-label` intentionally
does not. Fixed Canvas remains unchanged. A stored Flow Template must declare
the capability and must be recompiled before activation if it predates the new
required-capability metadata.

See the [JSON Render API reference](/docs/api-reference/), the
[Flow Document example](/gpdf-agent-examples/flow-document-report.json), and
the [Playground](/playground/) for the complete public contract.
