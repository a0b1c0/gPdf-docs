# gPdf Docs

[![Release](https://img.shields.io/badge/release-v2026.05-E8573A?style=flat-square)](https://github.com/a0b1c0/gPdf-docs/blob/main/CHANGELOG.md)
[![License](https://img.shields.io/badge/license-MIT-0F1114?style=flat-square)](./LICENSE)
[![Runtime](https://img.shields.io/badge/runtime-Cloudflare%20Workers-F38020?style=flat-square&logo=cloudflare&logoColor=white)](https://gpdf.com)
[![Built with](https://img.shields.io/badge/built%20with-Rust%20%2B%20WASM-DEA584?style=flat-square&logo=rust&logoColor=white)](https://gpdf.com)
[![Uptime](https://img.shields.io/badge/uptime-99.99%25%20%2824h%29-success?style=flat-square)](https://gpdf.com/status/)
[![Render p50](https://img.shields.io/badge/render%20p50-3%20ms-blue?style=flat-square)](https://gpdf.com)

Public API documentation, executable examples, and comparison materials for the **gPdf** PDF rendering service.

> gPdf turns a JSON `DocumentRequest` into a PDF document at the edge in 3–5 ms. Rust + WebAssembly running on Cloudflare Workers. No browser, no cold start, no document persistence.

- 🌐 **Service** — [gpdf.com](https://gpdf.com)
- 💵 **Pricing** — $5 / month for 100K pages (Basic tier); free trial 100 pages / day
- 📰 **Blog** — [gpdf.com/blog](https://gpdf.com/blog/)
- 🛠 **Changelog** — [gpdf.com/changelog](https://gpdf.com/changelog/) · also mirrored in [CHANGELOG.md](./CHANGELOG.md)
- 🛡 **Status** — [gpdf.com/status](https://gpdf.com/status/)

## Quick example

```bash
curl -X POST "https://api.gpdf.com/api/v1/pdf/render" \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d @examples/invoice.json \
  -o invoice.pdf
```

See [`examples/`](./examples/) for ready-to-run JSON payloads.

## Documentation

| Doc | Endpoint | When to use |
|---|---|---|
| [API Reference](./api-reference.md) | `POST /api/v1/pdf/render` | Full `DocumentRequest` with pages, elements, coordinates, tables. Use when you need pixel-level control. |
| [Template API](./template-api.md) | `POST /api/v1/template-render` | `template_id` + `data` array. Use when you have a published template and only business data changes per call. |
| [E-Invoice API](./e-invoice-api.md) | `POST /api/v1/e-invoice/render` | Factur-X / ZUGFeRD hybrid invoices: PDF/A-3 with embedded EN 16931 XML. For EU e-invoicing mandate compliance. |
| [Template Authoring](./template-authoring.md) | — | Author reusable templates that bind business data. |

## Verified facts

- **Render latency (p50):** 3 ms for a 1-page A4 invoice (median across 1K invocations on a Cloudflare Workers isolate).
- **Pricing:** $5 / month for 100,000 pages (Basic); equivalent to $0.00005 per page.
- **PDF/A profiles:** PDF/A-1b · 2b · 3b · 4 selectable per request.
- **In-memory document lifetime:** ~4 ms. Stateless: the buffer exists only inside one V8 isolate for the request, then is freed. No persistence, no document store.

## Comparisons

Performance + capability head-to-heads against other PDF tools:

- [gPdf vs Puppeteer](./comparisons/vs-puppeteer.md) — headless Chromium vs edge-native WASM
- More comparisons coming (DocRaptor, PDFShift, Anvil, Prince, wkhtmltopdf, PDFKit, ReportLab).

## Capabilities at a glance

- **Layout:** `pages[]`, `elements[]`, x/y in millimetres, `x_anchor` for content_right alignment.
- **Text:** paragraph, list, page_break blocks. Variables resolved post-layout: `page`, `total_pages`.
- **Tables:** row-span across pages, repeating headers, alternate fill, double-line borders.
- **Stack/block:** invoice-tail subtotal/total blocks that follow a paginated table.
- **Layers:** background, watermark (algorithmic tiled), and stamp (last-page approval marks).
- **Barcodes:** 30+ vector symbologies — QR, GS1-128, PDF417, DataMatrix, Aztec, MaxiCode, ITF-14, SSCC-18, UPS S10, Code 128 family, EAN/UPC family.
- **CJK:** NotoSans CJK embedded with automatic fallback. No OS-level font install required.
- **PDF/A:** 1b · 2b · 3b · 4 · UA-1 selectable per request.
- **E-invoice:** Factur-X / ZUGFeRD attachment streams ride on PDF/A-3. Aligns with EN 16931 mandates rolling across the EU through 2026 and 2027.
- **Output:** binary (`Content-Disposition: inline`) or file (`Content-Disposition: attachment`).

## Where it runs

Inside Cloudflare Workers V8 isolates. No Chromium container, no Lambda warming, no document persistence after the request. Same input renders to a byte-identical PDF across isolates.

## About this repo

This repo is the **public documentation surface** of gPdf:

- ✅ API contract (request shapes, error codes, limits)
- ✅ Executable JSON examples
- ✅ Performance comparisons
- ✅ Changelog mirror

The **gPdf rendering engine source code is not public**. The marketing-site source (Astro app, blog, hreflang infrastructure) is also kept private. This repository contains the public-facing API contract and integration material only.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for issue + PR guidelines.

## License

Documentation content licensed under [MIT](./LICENSE).
