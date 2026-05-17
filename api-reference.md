# gPdf API Reference

> Status: Public API surface. Authoritative source for the public docs site.
> Last updated: 2026-05-08
>
> This document defines what callers of the gPdf HTTP API can rely on. It does
> not describe internal control-plane behaviour, infrastructure, or any field
> not listed here. Anything not documented here is **not** part of the public
> contract and may change without notice.

---

## 1. Overview

gPdf turns a JSON request into a PDF document. There are three public APIs,
each tuned for a different integration shape:

| API | Endpoint | When to use |
|-----|----------|-------------|
| JSON Render | `POST /api/v1/pdf/render` | You describe pages, elements, coordinates, tables, and pagination yourself. Best for designers, custom reports, and one-off layouts. |
| Template Render | `POST /api/v1/template-render` | You only send `template_id` + business `data`. Best for ERP, OMS, WMS, and any system that wants a stable contract per document type. |
| E-Invoice Render | `POST /api/v1/e-invoice/render` | You need a Factur-X / ZUGFeRD compliant PDF/A-3b with embedded CII XML. |

A simple decision tree:

- "I want pixel control over the layout." → JSON Render
- "My team agreed on a template name and a list of fields." → Template Render
- "I need an EU-mandate-compliant electronic invoice." → E-Invoice Render

All three return `application/pdf` on success and `application/json` on error,
share the same authentication, and share the same error-code namespace.

### 1.1 Request and response basics

Every request uses:

```http
POST /api/v1/<route>
Host: api.gpdf.com
Content-Type: application/json
Authorization: Bearer <token>
X-Request-Id: <optional-client-id>
```

Every successful response carries:

- `Content-Type: application/pdf`
- `Content-Disposition: inline; filename="..."` (default) or `attachment; filename="..."` when `output.mode = "file"`
- `X-Request-Id: <echoed-or-generated>`

Every error response carries:

- `Content-Type: application/json`
- `X-Request-Id: <echoed-or-generated>`

The error body is always:

```json
{
  "error": true,
  "code": "API-002",
  "message": "x must be >= 0",
  "req_id": "7f7d2f5a-4cb0-4c4e-b6ef-8f6d3e0b1fd8"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `error` | `boolean` | Always `true` on the error envelope. |
| `code` | `string` | Public error code. See [§6.1](#61-error-codes). |
| `message` | `string` | Human-readable explanation. Some auth and system errors return a redacted message. |
| `req_id` | `string` | Mirrors `X-Request-Id`. Always present. Use it when filing support tickets. |

### 1.2 What this document does not cover

The following are intentionally **not** part of the public contract:

- Internal storage layout, queue topology, or any specific cloud service used to deliver responses.
- Token issuance, billing, quota provisioning, and policy management. Those happen in the gPdf Console, not over this API.
- Template authoring (how to design and publish a template). See the internal `template-authoring.en.md` if you maintain templates.
- Beta endpoints not listed in this document.

---

## 2. Authentication and Environments

### 2.1 Environments

gPdf runs two public environments. They are isolated: tokens, templates, and
e-invoice jobs do not cross over.

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Production | `https://api.gpdf.com` | Live traffic. SLAs apply. |
| Test | `https://api-test.gpdf.com` | Pre-integration sandbox. Same API shape, separate tokens, separate templates, no SLA. |

Build your client so the base URL is configuration, not a constant. Most
integrations read it from an environment variable like `GPDF_BASE_URL`.

### 2.2 Authentication

Every render endpoint requires a Bearer token:

```http
Authorization: Bearer sk_live_<YOUR_API_KEY>
```

Rules:

- The token is opaque. Do not parse it.
- Tokens are environment-scoped. A test token will be rejected by production with `API-102`.
- Tokens may carry policy constraints (max pages per request, allowed PDF/A profiles, allowed e-invoice standards). Constraint violations return `API-002` with the offending field named in `message`.
- A revoked or expired token returns `API-103` with a redacted message. Treat both as "rotate or contact support".

The single endpoint that does not require authentication is
`GET /api/v1/e-invoice/capabilities`. See [§5.1](#51-e-invoice-capabilities).

### 2.3 Request IDs

Every request gets a request ID. You may supply one via the `X-Request-Id`
header; if you don't, gPdf generates one. It is echoed in every response —
both success and error — and is included in `error.req_id`.

Recommended client behaviour:

- Generate a UUID v4 per outbound request and pass it as `X-Request-Id`.
- Log the request ID alongside your application's correlation ID.
- Quote it verbatim when reporting issues.

```bash
curl -X POST "https://api.gpdf.com/api/v1/pdf/render" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: $(uuidgen)" \
  --data-binary @request.json \
  --output out.pdf
```

### 2.4 Rate limiting and retries

gPdf does **not** currently publish rate-limit headers (`X-RateLimit-*`,
`Retry-After`) or an idempotency-key contract. This is a deliberate omission
in the v1 surface; future versions may add them.

Recommended client behaviour today:

- Cap concurrency per token at a number you negotiated with your gPdf account contact. Most production tokens are sized for sustained 5-20 req/s; bursts above that should be queued client-side.
- On `5xx` or network error, retry with exponential backoff (initial 500 ms, max 5 s, max 3 attempts).
- On `4xx`, do **not** retry. The request will fail the same way every time. Inspect `code` and `message` and fix the request.
- For at-most-once semantics on PDF generation, deduplicate on your side before calling. The API does not detect duplicate submissions.

---

## 3. Quick Start

The fastest way to verify your token, the route, and PDF byte handling is the
five-second request below. Save it as `quickstart.json`:

```json
{
  "pages": [
    {
      "size": "label_100_150",
      "elements": [
        {
          "type": "text",
          "x": 10,
          "y": 18,
          "content": "Hello gPdf"
        }
      ]
    }
  ]
}
```

Then call the API:

```bash
curl -X POST "https://api.gpdf.com/api/v1/pdf/render" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: quickstart-001" \
  --data-binary @quickstart.json \
  --output quickstart.pdf
```

You should get a 100 × 150 mm one-page PDF on disk. If you don't:

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `401` with `API-101` | Missing or malformed `Authorization` header | Confirm the header is exactly `Bearer <token>`. |
| `401` with `API-102` | Token rejected | Confirm the token belongs to the environment you are calling. |
| `400` with `API-001` | Body is not valid JSON | Check the file with `jq .` or a JSON linter. |
| `400` with `API-002` | Body parsed but failed validation | Read `message` — it names the offending field. |
| `200` with empty body | Saved with `--output` but the response is JSON | Drop `--output`, re-run; you will see a JSON error envelope. |

After the minimum request succeeds, move on to:

- [§4 JSON Render API](#4-json-render-api) for full layout control.
- [§5 E-Invoice Render API](#5-e-invoice-render-api) for compliance documents.
- The Template Render API (`template-api.en.md`) for `template_id + data` integrations.

---

## 4. JSON Render API

`POST /api/v1/pdf/render` is the lower-level rendering endpoint. The request
body is a single `DocumentRequest` JSON object that describes pages, elements,
styles, and pagination behaviour explicitly. The caller has full control.

If you want a higher-level integration where you only send a template ID and
business data, use `POST /api/v1/template-render` (see `template-api.en.md`).

### 4.1 Endpoint

| Property | Value |
|----------|-------|
| Method | `POST` |
| Path | `/api/v1/pdf/render` |
| Auth | Required — `Authorization: Bearer <token>` |
| Request `Content-Type` | `application/json` |
| Success | `200`, `Content-Type: application/pdf` |
| Error | `4xx` / `5xx`, `Content-Type: application/json` |

```bash
curl -X POST "https://api.gpdf.com/api/v1/pdf/render" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: $(uuidgen)" \
  --data-binary @request.json \
  --output out.pdf
```

### 4.2 Request structure

```json
{
  "settings": { },
  "layers": { },
  "header": { },
  "footer": { },
  "pages": [ ]
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `settings` | `Settings` | No | Global defaults, metadata, output mode, PDF/A profile. See §4.14. |
| `layers` | `Layers` | No | Background / watermark / stamp. See §4.13. |
| `header` | `Section` | No | Global page header. See §4.12. |
| `footer` | `Section` | No | Global page footer. See §4.12. |
| `pages` | `Page[]` | **Yes** | One or more pages. |

The smallest valid request:

```json
{
  "pages": [
    {
      "size": "a4",
      "elements": [
        { "type": "text", "x": 10, "y": 20, "content": "Hello gPdf" }
      ]
    }
  ]
}
```

When you omit `settings`, gPdf applies safe defaults for fonts, strokes,
fills, and output mode. You do not need to write a 200-line `DocumentRequest`
to get a usable PDF.

### 4.3 Page

A page is described by its size and the elements it contains. Body elements
position themselves in millimetres from the page's top-left corner unless
`page_margin` is configured.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `size` | `string` | One of `size` or `width+height` | Named preset. Case-insensitive. |
| `width` | `number` | One of `size` or `width+height` | Page width (mm). |
| `height` | `number` | One of `size` or `width+height` | Page height (mm). |
| `margin` | `PageMargin` | No | Per-page margin override. |
| `elements` | `Element[]` | No | Body elements. May be empty. |

Rules:

- `size` and `width/height` are mutually exclusive on the same page. Providing both returns `API-002`.
- A page without `size` must provide both `width` and `height`.

#### 4.3.1 Size presets

| Preset | Dimensions | Typical use |
|--------|------------|-------------|
| `a4` | `210 × 297 mm` | Default office page in most non-US locales. |
| `a6` | `105 × 148 mm` | Postcards, small notices. |
| `letter` | `215.9 × 279.4 mm` | US default. |
| `legal` | `215.9 × 355.6 mm` | US legal documents. |
| `label_100_100` | `100 × 100 mm` | Square labels. |
| `label_100_150` | `100 × 150 mm` | Most common shipping label. |
| `label_4_6_in` | `101.6 × 152.4 mm` | US 4×6" shipping label. |

```json
{
  "pages": [
    { "size": "a4", "elements": [] },
    { "size": "letter", "elements": [] },
    { "width": 100, "height": 150, "elements": [] }
  ]
}
```

#### 4.3.2 Page margin and content box

Configure margins globally (`settings.page_margin`) or per page
(`pages[].margin`):

```json
{
  "settings": {
    "page_margin": { "top": 10, "right": 12, "bottom": 10, "left": 12 }
  },
  "pages": [
    {
      "size": "letter",
      "margin": { "top": 8, "right": 10, "bottom": 12, "left": 10 },
      "elements": []
    }
  ]
}
```

Behaviour:

- Without `page_margin`, body elements use absolute page coordinates.
- Once `page_margin` is set, body element `x`/`y` become relative to the **content box** (the area inside the margins).
- Elements that overflow the content box return `API-002`. There is no automatic clipping.
- `header` and `footer` always use absolute page coordinates and ignore margins.
- Auto-paginated overflow continues from the top of the next page's content box.

### 4.4 Coordinates and units

- All coordinates and lengths are in millimetres (mm).
- The origin is the top-left corner of the page (or the content box if `page_margin` is set).
- The X axis goes right; the Y axis goes down.
- See [§6.5](#65-coordinates-and-units) for `rotation` rules per element.

### 4.5 Element types

The `pages[].elements` array (and the body of `header`, `footer`,
`layers.background`, `layers.stamp`) contains element objects. Every element
has a `type` discriminator.

| `type` | Section | Notes |
|--------|---------|-------|
| `text` | §4.6 | Plain, rich `spans`, or block text. |
| `barcode` | §4.7 | 2D matrix and 1D linear formats. |
| `image` | §4.8 | Asset reference or inline base64. |
| `line` | §4.9 | Single line segment. |
| `rect` | §4.9 | Rectangle, optionally rounded. |
| `circle` | §4.9 | Circle by centre + radius. |
| `ellipse` | §4.9 | Ellipse by centre + radii. |
| `polygon` | §4.9 | Closed polygon from a point list. |
| `link` | §4.9 | Standalone clickable hotspot. |
| `table` | §4.10 | Tabular data with headers, spans, pagination. |
| `stack` | §4.11 | Vertical composition of a table followed by trailing blocks. |

Common fields shared by most elements:

| Field | Type | Notes |
|-------|------|-------|
| `z_index` | `number` | Stacking order. Default `0`. Higher draws on top. |
| `comment` | `string` | Free-form note. Not rendered. |
| `rotation` | `number` | See per-element rules. Most elements support `0/90/180/270`; `text` and `image` accept any integer angle. |
| `link` | `LinkSpec` | Make the element clickable. See §4.9.6. |

Hyperlink modes:

- Attach `link` to an element (`text`, `barcode`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `image`).
- Use a standalone `type: "link"` hotspot when you need to overlay a clickable region.

#### 4.5.1 Horizontal anchor (`x_anchor`)

`x_anchor` aligns elements relative to a reference edge instead of an absolute
X. It is supported on `text`, `barcode`, `rect`, `image`, and `link`. It is
**not** supported on `line`, `circle`, `ellipse`, `polygon`, `table`,
`stack`, or block-text content.

```json
{
  "type": "text",
  "x_anchor": { "reference": "content_right", "offset": 8 },
  "y": 12,
  "content": "$1,235.85",
  "style": {
    "width": 24,
    "text_align": "right"
  }
}
```

| Reference | Meaning |
|-----------|---------|
| `page_left` | Left page edge. |
| `page_right` | Right page edge. |
| `content_left` | Left edge of the content box. Falls back to the page edge if no margin is set. |
| `content_right` | Right edge of the content box. |
| `table_left` | Left edge of the parent table. **Only valid inside `stack > block`.** |
| `table_right` | Right edge of the parent table. **Only valid inside `stack > block`.** |

Resolved X:

- Left references: `resolved_x = reference + offset`
- Right references: `resolved_x = reference - offset - element_width`

Rules:

- `x` and `x_anchor` are mutually exclusive. Sending both returns `API-002`.
- When using `x_anchor`, the element must have a width:
  - `text` (plain or `spans` shorthand): `style.width`
  - `text` (block): `frame.width`
  - `barcode`, `rect`, `image`, `link`: their own `width` field.

### 4.6 Text

The `text` element accepts three input forms:

| Form | Use when |
|------|----------|
| Plain text shorthand | One short string in one style. |
| `spans` rich text | One paragraph mixing multiple inline styles. |
| Block text | Multi-paragraph layout, lists, page breaks, variables, or pagination control. |

All three pass through the same validation and rendering pipeline. Block text
is the most expressive.

Required fields for any form:

- `y` (number).
- `content` (string, `{ spans }`, or `{ blocks }`).
- One of `x` or `x_anchor`.

#### 4.6.1 Plain text

```json
{
  "type": "text",
  "x": 18,
  "y": 18,
  "content": "Hello gPdf"
}
```

With styling:

```json
{
  "type": "text",
  "x": 18,
  "y": 18,
  "content": "Invoice #INV-2026-001",
  "style": {
    "font_family": "NotoSans-Regular",
    "font_mode": "prefer",
    "font_size": 12,
    "font_weight": "bold",
    "color": "#111827",
    "width": 90,
    "text_align": "left",
    "line_height": 1.25,
    "letter_spacing": 0.2
  }
}
```

#### 4.6.2 `spans` rich text

`spans` is a single paragraph composed of style runs. Each span inherits
the element-level `style` and overrides only what it sets — typical use
cases are price lines, badges, and inline footnote markers.

```json
{
  "type": "text",
  "x": 18,
  "y": 30,
  "content": {
    "spans": [
      { "text": "Total ",      "style": { "color": "#6b7280" } },
      { "text": "USD ",        "style": { "color": "#6b7280", "font_size": 9 } },
      { "text": "1,248.50",
        "style": { "font_weight": "bold", "font_size": 14, "color": "#111827" } },
      { "text": " (incl. tax)", "style": { "color": "#6b7280" } },
      { "text": "*",
        "style": { "script": "superscript", "color": "#dc2626" } }
    ]
  },
  "style": {
    "font_family": "NotoSans-Regular",
    "font_size": 11,
    "width": 90
  }
}
```

#### 4.6.3 Block text

Block text uses an explicit document tree of `blocks` and `inlines`. The
example below stitches together every common feature in one element — a
heading paragraph, a justified body paragraph mixing bold / coloured /
italic runs, an ordered list whose items each carry their own bold
emphasis, and a right-aligned footer paragraph that prints `page /
total_pages`. The frame uses `overflow: "paginate"` so the article flows
across as many pages as it needs.

```json
{
  "type": "text",
  "x": 18,
  "y": 38,
  "frame": {
    "width": 174,
    "overflow": "paginate"
  },
  "defaults": {
    "run": {
      "font_family": "NotoSans-Regular",
      "font_mode": "prefer",
      "font_size": 10.5,
      "color": "#1f2937"
    },
    "paragraph": {
      "align": "left",
      "direction": "auto",
      "line_height": 1.45
    }
  },
  "content": {
    "blocks": [
      {
        "type": "paragraph",
        "inlines": [
          {
            "type": "text",
            "text": "Quarterly Review",
            "style": { "font_size": 16, "font_weight": "bold", "color": "#111827" }
          }
        ]
      },
      {
        "type": "paragraph",
        "style": { "align": "justify" },
        "inlines": [
          { "type": "text", "text": "Revenue grew " },
          { "type": "text", "text": "23% year-over-year",
            "style": { "font_weight": "bold", "color": "#0f766e" } },
          { "type": "text", "text": " on the back of three drivers — enterprise expansion, the new self-serve tier, and improved retention. The next two quarters will focus on the " },
          { "type": "text", "text": "EMEA rollout",
            "style": { "font_style": "italic" } },
          { "type": "text", "text": "." }
        ]
      },
      {
        "type": "list",
        "list": { "kind": "ordered" },
        "items": [
          { "blocks": [{ "type": "paragraph", "inlines": [
            { "type": "text", "text": "Enterprise pipeline doubled to " },
            { "type": "text", "text": "$48M qualified ARR",
              "style": { "font_weight": "bold" } },
            { "type": "text", "text": "." }
          ] }] },
          { "blocks": [{ "type": "paragraph", "inlines": [
            { "type": "text", "text": "Self-serve activation reached " },
            { "type": "text", "text": "61%", "style": { "font_weight": "bold" } },
            { "type": "text", "text": " — three points above target." }
          ] }] },
          { "blocks": [{ "type": "paragraph", "inlines": [
            { "type": "text", "text": "Net retention held at " },
            { "type": "text", "text": "118%", "style": { "font_weight": "bold" } },
            { "type": "text", "text": ", a record for the company." }
          ] }] }
        ]
      },
      {
        "type": "paragraph",
        "style": { "align": "right" },
        "inlines": [
          { "type": "text", "text": "Page " },
          { "type": "variable", "name": "page", "scope": "system" },
          { "type": "text", "text": " / " },
          { "type": "variable", "name": "total_pages", "scope": "system" }
        ]
      }
    ]
  }
}
```

> Restriction: a single `text` element cannot mix an explicit `page_break`
> block with `system.page` or `system.total_pages` variables. Page numbers
> are evaluated before pagination, so the renderer rejects the combination
> at validation time. Use `frame.overflow = "paginate"` to break across
> pages when content overflows (as above), or use `page_break` blocks with
> static text only.

Top-level fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `type` | `"text"` | **Yes** | |
| `y` | `number` | **Yes** | |
| `x` or `x_anchor` | | One of | See §4.5.1. |
| `rotation` | `integer` | No | Any integer angle. |
| `z_index` | `number` | No | |
| `comment` | `string` | No | |
| `link` | `LinkSpec` | No | Element-level link. Mutually exclusive with any inline `link`. |
| `style` | `TextStyle` | No | For plain or `spans` form. |
| `frame` | `BlockTextFrame` | No | For block form. |
| `defaults` | `BlockTextDefaults` | No | Default `run` / `paragraph` / `frame` for the block tree. |
| `content` | `string \| { spans } \| { blocks }` | **Yes** | The text content in one of the three forms. |

##### Paragraph block

```json
{
  "type": "paragraph",
  "style": {
    "align": "justify",
    "direction": "auto",
    "line_height": 1.35
  },
  "inlines": [
    { "type": "text", "text": "Line 1: " },
    { "type": "variable", "name": "page", "scope": "system" }
  ]
}
```

`paragraph.style` fields:

- **Currently rendered** (validated and applied at runtime):
  - `align`: `left | center | right | justify`
  - `direction`: `auto | ltr | rtl`
  - `line_height` (number)
- **Accepted but not yet wired into the runtime renderer** (validator
  rejects them with `paragraph layout features … not wired`; the field
  list is reserved so future API additions can land without a contract
  break):
  - `space_before`, `space_after` (mm)
  - `indent_left`, `indent_right`, `indent_first_line`, `hanging_indent` (mm)
  - `keep_together`, `keep_with_next` (boolean)
  - `widow_orphan_control` (boolean)
  - `tabs` (array)

##### List block

```json
{
  "type": "list",
  "list": { "kind": "bullet" },
  "items": [
    {
      "blocks": [
        {
          "type": "paragraph",
          "inlines": [
            { "type": "text", "text": "First item — " },
            { "type": "text", "text": "with bold emphasis",
              "style": { "font_weight": "bold" } }
          ]
        }
      ]
    },
    {
      "blocks": [
        {
          "type": "paragraph",
          "inlines": [{ "type": "text", "text": "Second item, first paragraph." }]
        },
        {
          "type": "paragraph",
          "inlines": [{ "type": "text", "text": "An item may carry multiple paragraphs." }]
        }
      ]
    },
    {
      "blocks": [
        {
          "type": "paragraph",
          "inlines": [{ "type": "text", "text": "Third item." }]
        }
      ]
    }
  ]
}
```

`list.list` accepts only `kind` (`ordered | bullet`) at runtime today.
Spacing / numbering controls (`marker_gap`, `item_spacing`, `start_at`,
continuation rules) are reserved in the schema but the renderer is not yet
wired for them — validator rejects them with `list spacing/continuation
features … not wired`. Stick to `kind` for runnable examples until a
release note opens the rest.

`list` is only allowed in the **full text profile** (see §4.6.4). It is
rejected inside `header`, `footer`, layers, table cells, and barcode text.

##### Page break

```json
{ "type": "page_break" }
```

`page_break` is the only way to force a page break inside block text. The
older `\f` string convention is no longer supported.

##### Inline nodes

Four inline node types are public:

| Type | Example |
|------|---------|
| `text` | `{ "type": "text", "text": "Hello", "style": { "font_weight": "bold" } }` |
| `variable` | `{ "type": "variable", "name": "page", "scope": "system" }` |
| `line_break` | `{ "type": "line_break" }` |
| `tab` | `{ "type": "tab" }` |

`variable.scope` may be:

- `system` — page numbers and totals. JSON Render only resolves `page` and `total_pages`.
- `binding` — values supplied by the template-data pipeline. **Validation fails** in JSON Render. Use Template Render for data substitution.
- `computed` — derived values. **Validation fails** in JSON Render today.

##### Inline text style

`run`-level fields:

- `font_family`, `font_size`
- `font_weight`: `normal | medium | semibold | bold`
- `font_style`: `normal | italic`
- `font_mode`: `strict | prefer`
- `color`, `opacity`, `letter_spacing`
- `script`: `normal | superscript | subscript`
- `background`, `decoration`, `link_style`

Rules:

- `font_mode` cannot appear without a same-level `font_family`.
- `font_mode = "auto"` is **not** a public input. Auto mode is implicit when no font is declared anywhere in the inheritance chain.
- `strict` failure (declared font cannot cover the text) returns `API-002`.
- `auto` or `prefer` total fallback failure returns `API-504`.

##### Block text frame

`frame` fields:

- `width`, `height` (mm)
- `vertical_align`: `top | middle | bottom`
- `overflow`: `visible | clip | ellipsis | paginate`
- `shrink_to_fit` (boolean)
- `min_font_size` (number)
- `padding`, `border`, `background`
- `columns`, `column_gap`

Rules:

- `frame` is rejected inside table cells and barcode text.
- `rotation != 0` cannot combine with `frame.overflow = "paginate"`.
- `frame.height` cannot combine with `page_break`.
- `frame.overflow ∈ { clip, ellipsis }` cannot combine with `page_break`.
- `frame.overflow = "paginate"` cannot combine with `shrink_to_fit = true`.
- `header` / `footer` / layer text cannot use `paginate` or multi-column.

#### 4.6.4 Text feature profiles

The exact subset of text features available depends on **where** the text
appears. There is no `profile` field in the JSON; the profile is implied by
container.

| Profile | Used in | Allowed blocks | Allowed inlines |
|---------|---------|----------------|-----------------|
| Full | `pages[].elements[]` (top-level body text) | `paragraph`, `list`, `page_break` | `text`, `variable`, `line_break`, `tab` |
| Section | `header`, `footer`, `layers.background`, `layers.watermark`, `layers.stamp` | `paragraph` | `text`, `variable`, `line_break`, `tab` |
| Table | Inside `table` cells | `paragraph` | `text`, `variable`, `line_break` |
| Barcode | Inside `barcode_text` | `paragraph` | `text`, `variable`, `line_break` |

All three restricted profiles reject `list` and `page_break`. Table and
barcode profiles additionally reject `frame`, `tab`, and inline `link`. The
section profile keeps `frame` available but with two carve-outs:
`frame.overflow = "paginate"` and multi-column (`frame.columns > 1`) are not
allowed inside a header / footer / layer (the parent section is not itself a
paginating context).

Frame fields enter validation through the plain `text.style` shorthand too —
`style.width` / `style.height` / `style.vertical_align` / `style.text_overflow`
/ `style.shrink_to_fit` / `style.min_font_size` are coerced into a shadow
frame for contract checking. Setting any of these on a `barcode_text.style`
or table-cell text style therefore produces the same `frame is not allowed`
rejection as a literal `frame: { ... }` block.

### 4.7 Barcode

2D / matrix code (square module grid):

```json
{
  "type": "barcode",
  "x": 18,
  "y": 34,
  "format": "qrcode",
  "content": "https://gpdf.com/docs/api-reference/#47-barcode",
  "width": 28,
  "height": 28,
  "style": {
    "color": "#111111",
    "background_color": "#FFFFFF"
  },
  "barcode_text": {
    "enabled": true,
    "position": "bottom",
    "offset": 1.5,
    "style": {
      "font_family": "NotoSans-Regular",
      "font_mode": "prefer",
      "font_size": 8,
      "color": "#374151",
      "text_align": "center"
    }
  }
}
```

1D / linear code (rectangular bars). The same shape works for any format
in the linear group below — only `format` and `content` change:

```json
{
  "type": "barcode",
  "x": 18,
  "y": 70,
  "format": "code128",
  "content": "INV-2026-001",
  "width": 60,
  "height": 16,
  "style": {
    "color": "#000000",
    "background_color": "#FFFFFF"
  },
  "barcode_text": {
    "enabled": true,
    "position": "bottom",
    "offset": 1.0,
    "style": {
      "font_family": "NotoSans-Regular",
      "font_mode": "prefer",
      "font_size": 8,
      "color": "#111111",
      "text_align": "center"
    }
  }
}
```

Required fields:

- `y`, `format`, `content`, `width`, `height`.
- One of `x` or `x_anchor`.

Optional: `style`, `options`, `barcode_text`, `rotation`, `z_index`,
`comment`, `link`.

Rules:

- `rotation` accepts only `0`, `90`, `180`, `270`.
- `barcode_text` inherits the same rotation.
- `format` is case-insensitive. `-` and `_` are equivalent separators.
- 2D / matrix codes encode as a module matrix; 1D / linear codes encode as bars; `maxicode` uses a hexagonal grid.

Supported `format` values (current build):

- 2D / matrix: `qrcode` (`qr`), `microqr` (`micro-qr`), `pdf417`, `micropdf417`, `datamatrix` (`data-matrix`), `gs1datamatrix`, `aztec`, `maxicode`, `gs1qrcode`
- 1D / linear: `code128`, `code128a`, `code128b`, `code128c`, `gs1128`, `code39`, `code93`, `codabar`, `ean8`, `ean13`, `upca`, `upce`, `itf` (`interleaved2of5`), `itf14`, `gtin8`, `gtin12`, `gtin13`, `gtin14`, `isbn`, `sscc`
- Other: `msi`, `msi10`, `msi11`, `msi1010`, `msi1110`, `upus10`, `uspsimb`, `upcacomposite`, `upcecomposite`

### 4.8 Image

```json
{
  "type": "image",
  "x": 4,
  "y": 8,
  "width": 33,
  "height": 11,
  "asset": "aitrack-1",
  "format": "jpg"
}
```

Required fields:

- `y`, `width`, `height`.
- One of `x` or `x_anchor`.

Optional: `rotation`, `z_index`, `comment`, `link`.

Image source — exactly one of:

- **Shorthand**: top-level `asset` (with optional top-level `format`).
- **Explicit**: top-level `source` object.

```json
{ "type": "image", "x": 4, "y": 8, "width": 33, "height": 11,
  "source": { "kind": "asset", "key": "aitrack-1", "format": "jpg" } }
```

```json
{ "type": "image", "x": 4, "y": 8, "width": 33, "height": 11,
  "source": { "kind": "base64", "format": "jpg",
              "payload": "/9j/4AAQSkZJRgABAQAASABIAAD/4QBARXhpZgAATU0AKgAAAAgAAYdpAAQAAAABAAAAGgAAAAAAAqACAAQAAAABAAABSqADAAQAAAABAAAAbgAAAAD/7QA4UGhvdG9zaG9wIDMuMAA4QklNBAQAAAAAAAA4QklNBCUAAAAAABDUHYzZjwCyBOmACZjs+EJ+/+IH2ElDQ19QUk9GSUxFAAEBAAAHyGFwcGwCIAAAbW50clJHQiBYWVogB9kAAgAZAAsAGgALYWNzcEFQUEwAAAAAYXBwbAAAAAAAAAAAAAAAAAAAAAAAAPbWAAEAAAAA0y1hcHBsAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALZGVzYwAAAQgAAABvZHNjbQAAAXgAAAWKY3BydAAABwQAAAA4d3RwdAAABzwAAAAUclhZWgAAB1AAAAAUZ1hZWgAAB2QAAAAUYlhZWgAAB3gAAAAUclRSQwAAB4wAAAAOY2hhZAAAB5wAAAAsYlRSQwAAB4wAAAAOZ1RSQwAAB4wAAAAOZGVzYwAAAAAAAAAUR2VuZXJpYyBSR0IgUHJvZmlsZQAAAAAAAAAAAAAAFEdlbmVyaWMgUkdCIFByb2ZpbGUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAG1sdWMAAAAAAAAAHwAAAAxza1NLAAAAKAAAAYRkYURLAAAAJAAAAaxjYUVTAAAAJAAAAdB2aVZOAAAAJAAAAfRwdEJSAAAAJgAAAhh1a1VBAAAAKgAAAj5mckZVAAAAKAAAAmhodUhVAAAAKAAAApB6aFRXAAAAEgAAArhrb0tSAAAAFgAAAspuYk5PAAAAJgAAAuBjc0NaAAAAIgAAAwZoZUlMAAAAHgAAAyhyb1JPAAAAJAAAA0ZkZURFAAAALAAAA2ppdElUAAAAKAAAA5ZzdlNFAAAAJgAAAuB6aENOAAAAEgAAA75qYUpQAAAAGgAAA9BlbEdSAAAAIgAAA+pwdFBPAAAAJgAABAxubE5MAAAAKAAABDJlc0VTAAAAJgAABAx0aFRIAAAAJAAABFp0clRSAAAAIgAABH5maUZJAAAAKAAABKBockhSAAAAKAAABMhwbFBMAAAALAAABPBydVJVAAAAIgAABRxlblVTAAAAJgAABT5hckVHAAAAJgAABWQAVgFhAGUAbwBiAGUAYwBuAP0AIABSAEcAQgAgAHAAcgBvAGYAaQBsAEcAZQBuAGUAcgBlAGwAIABSAEcAQgAtAHAAcgBvAGYAaQBsAFAAZQByAGYAaQBsACAAUgBHAEIAIABnAGUAbgDoAHIAaQBjAEMepQB1ACAAaADsAG4AaAAgAFIARwBCACAAQwBoAHUAbgBnAFAAZQByAGYAaQBsACAAUgBHAEIAIABHAGUAbgDpAHIAaQBjAG8EFwQwBDMEMAQ7BEwEPQQ4BDkAIAQ/BEAEPgREBDAEOQQ7ACAAUgBHAEIAUAByAG8AZgBpAGwAIABnAOkAbgDpAHIAaQBxAHUAZQAgAFIAVgBCAMEAbAB0AGEAbADhAG4AbwBzACAAUgBHAEIAIABwAHIAbwBmAGkAbJAadSgAUgBHAEKCcl9pY8+P8Md8vBgAIABSAEcAQgAg1QS4XNMMx3wARwBlAG4AZQByAGkAcwBrACAAUgBHAEIALQBwAHIAbwBmAGkAbABPAGIAZQBjAG4A/QAgAFIARwBCACAAcAByAG8AZgBpAGwF5AXoBdUF5AXZBdwAIABSAEcAQgAgBdsF3AXcBdkAUAByAG8AZgBpAGwAIABSAEcAQgAgAGcAZQBuAGUAcgBpAGMAQQBsAGwAZwBlAG0AZQBpAG4AZQBzACAAUgBHAEIALQBQAHIAbwBmAGkAbABQAHIAbwBmAGkAbABvACAAUgBHAEIAIABnAGUAbgBlAHIAaQBjAG9mbpAaAFIARwBCY8+P8GWHTvZOAIIsACAAUgBHAEIAIDDXMO0w1TChMKQw6wOTA7UDvQO5A7oDzAAgA8ADwQO/A8YDrwO7ACAAUgBHAEIAUABlAHIAZgBpAGwAIABSAEcAQgAgAGcAZQBuAOkAcgBpAGMAbwBBAGwAZwBlAG0AZQBlAG4AIABSAEcAQgAtAHAAcgBvAGYAaQBlAGwOQg4bDiMORA4fDiUOTAAgAFIARwBCACAOFw4xDkgOJw5EDhsARwBlAG4AZQBsACAAUgBHAEIAIABQAHIAbwBmAGkAbABpAFkAbABlAGkAbgBlAG4AIABSAEcAQgAtAHAAcgBvAGYAaQBpAGwAaQBHAGUAbgBlAHIAaQENAGsAaQAgAFIARwBCACAAcAByAG8AZgBpAGwAVQBuAGkAdwBlAHIAcwBhAGwAbgB5ACAAcAByAG8AZgBpAGwAIABSAEcAQgQeBDEESQQ4BDkAIAQ/BEAEPgREBDgEOwRMACAAUgBHAEIARwBlAG4AZQByAGkAYwAgAFIARwBCACAAUAByAG8AZgBpAGwAZQZFBkQGQQAgBioGOQYxBkoGQQAgAFIARwBCACAGJwZEBjkGJwZFAAB0ZXh0AAAAAENvcHlyaWdodCAyMDA3IEFwcGxlIEluYy4sIGFsbCByaWdodHMgcmVzZXJ2ZWQuAFhZWiAAAAAAAADzUgABAAAAARbPWFlaIAAAAAAAAHRNAAA97gAAA9BYWVogAAAAAAAAWnUAAKxzAAAXNFhZWiAAAAAAAAAoGgAAFZ8AALg2Y3VydgAAAAAAAAABAc0AAHNmMzIAAAAAAAEMQgAABd7///MmAAAHkgAA/ZH///ui///9owAAA9wAAMBs/8AAEQgAbgFKAwEiAAIRAQMRAf/EAB8AAAEFAQEBAQEBAAAAAAAAAAABAgMEBQYHCAkKC//EALUQAAIBAwMCBAMFBQQEAAABfQECAwAEEQUSITFBBhNRYQcicRQygZGhCCNCscEVUtHwJDNicoIJChYXGBkaJSYnKCkqNDU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6g4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2drh4uPk5ebn6Onq8fLz9PX29/j5+v/EAB8BAAMBAQEBAQEBAQEAAAAAAAABAgMEBQYHCAkKC//EALURAAIBAgQEAwQHBQQEAAECdwABAgMRBAUhMQYSQVEHYXETIjKBCBRCkaGxwQkjM1LwFWJy0QoWJDThJfEXGBkaJicoKSo1Njc4OTpDREVGR0hJSlNUVVZXWFlaY2RlZmdoaWpzdHV2d3h5eoKDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uLj5OXm5+jp6vLz9PX29/j5+v/bAEMAAQEBAQEBAgEBAgMCAgIDBAMDAwMEBgQEBAQEBgcGBgYGBgYHBwcHBwcHBwgICAgICAkJCQkJCwsLCwsLCwsLC//bAEMBAgICAwMDBQMDBQsIBggLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLC//dAAQAFf/aAAwDAQACEQMRAD8A/vwooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAK+OP2ov2+v2WP2PLQn43+I3t79o/Mi0zT7S41K/kB6Ygto5GGexfavvX2PQMA5A61UbX95aCd7aH8lH7Qv8AwdFT6PJcaV+y3+z/AOK9cYZWLUPEcMthCTzg/Z4I5pGHTgyIfpX40fGT/g4P/wCC1HxPmnh8JacPAlpNkLFonhuSSRFPT97eJcNkeox9K/0b9xo3HrXZDE0Y7Ul83c55Uaj3n+B/lq+F/GP/AAXU/b68RyeGPDOrfFLxdJuxMiXF3YWMO/8A56NmC3jX2Ygegr9APhf/AMGt/wDwUi+MZi1j4/8Ai/QPCSv8zpf30+s3qg/7MQMWfUef+Nf6FRYmkrSWZz2pxSJWDjvNtn8dvw4/4NA/gbp0cc3xY+MWualL8pkj0jTrexTPcBpWuGI9DgV93fDr/g1+/wCCXPgTUrXWNTtPFHiO4tJY50/tLWCqb4mDDKW0cIIJHIOQRxX9ElFc8sbXlvM0WGpLaJ8iftO/sF/siftm3Wg3n7T3giz8YHwwJ10xLySZY7cXOzzMJFIitu8tfvA9OO9ea+Fv+CT/APwTS8F3CXXh74F+DI5ExhpdJgnPHr5qvn8a/QWisVVmlZSdvU1cIt3aOH074ZfDbR/CNl8P9J8PaZa6DpoVbTTYrSJLS3CZ2iOELsQDJxtUYzXXWVjY6bbraabBHbxL91IlCKPoBgVaoqLsoKKKKQBRRRQAUUUUAFFFFABRRRQAUUUUAf/Z" } }
```

Rules:

- `asset` and `source` are mutually exclusive.
- `source.kind` accepts `asset` and `base64`.
- The `payload` for `base64` is the raw base64 content **without** a `data:image/...;base64,` prefix. Data URIs are rejected.
- Supported formats: `jpg`, `jpeg`, `png`, `webp`, `svg`.
- `rotation` accepts any integer angle (e.g. `45`, `-30`).
- See [§6.2](#62-limits) for image and total request body size limits.

### 4.9 Shapes and links

Shape elements share the same stroke, fill, link, and stacking rules. Each
may attach an optional `link: LinkSpec`.

#### 4.9.1 Line

```json
{
  "type": "line",
  "x1": 4, "y1": 99,
  "x2": 96, "y2": 99,
  "stroke": {
    "color": "#000000",
    "width": 0.4,
    "dash": { "preset": "solid" }
  }
}
```

Stroke fields fall back through `settings.defaults.stroke` → service defaults
when omitted (see §4.15).

#### 4.9.2 Rect

```json
{
  "type": "rect",
  "x_anchor": { "reference": "content_right", "offset": 6 },
  "y": 20,
  "width": 60,
  "height": 20,
  "fill": { "color": "#FFFFFF" },
  "stroke": { "color": "#222222", "width": 0.6 },
  "corner_radius": 2
}
```

#### 4.9.3 Circle

```json
{
  "type": "circle",
  "cx": 40, "cy": 40, "r": 12,
  "fill": { "color": "#E6F4FF" },
  "stroke": { "color": "#2B6CB0", "width": 0.5 }
}
```

#### 4.9.4 Ellipse

```json
{
  "type": "ellipse",
  "cx": 70, "cy": 40, "rx": 16, "ry": 10,
  "rotation": 0,
  "fill": { "color": "#FFF7E6" },
  "stroke": { "color": "#C05621", "width": 0.5 }
}
```

#### 4.9.5 Polygon

```json
{
  "type": "polygon",
  "points": [
    { "x": 20, "y": 80 },
    { "x": 35, "y": 60 },
    { "x": 50, "y": 80 },
    { "x": 40, "y": 95 }
  ],
  "fill": { "color": "#F0FFF4" },
  "stroke": { "color": "#2F855A", "width": 0.5 }
}
```

#### 4.9.6 Link spec and standalone link

A `LinkSpec` is reused everywhere a hyperlink is allowed:

```json
{
  "target": { "type": "url", "url": "https://gpdf.com/docs/api-reference/#496-link-spec-and-standalone-link" },
  "alt": "Open the official site",
  "padding": 1.0,
  "border": { "color": "#1A202C", "width": 0.3 }
}
```

`LinkTarget` variants:

- URL: `{ "type": "url", "url": "https://..." }`
- In-document jump: `{ "type": "page", "page": 2, "x": 10, "y": 20 }`

Rules:

- URL schemes allowed: `http://`, `https://`, `mailto:`, `tel:`. Others return `API-002`.
- URL strings are trimmed before being written to the PDF annotation.
- `page` is 1-indexed and must not exceed the request's page count.
- `padding` and `border.width` must be finite and `>= 0`.
- `border.color` must be a valid hex colour if provided.

Standalone link element:

```json
{
  "type": "link",
  "x_anchor": { "reference": "content_left", "offset": 10 },
  "y": 10,
  "width": 40,
  "height": 8,
  "target": { "type": "url", "url": "https://gpdf.com/docs/api-reference/#496-link-spec-and-standalone-link" },
  "alt": "Open website"
}
```

Use the standalone form for clickable region overlays not bound to a specific
element.

### 4.10 Table

A table renders tabular data with optional grouped headers, row headers, cell
merging, alternate row fills, configurable grids, and page-break behaviour.

```json
{
  "type": "table",
  "x": 12,
  "y": 24,
  "width": 180,
  "columns": [
    { "key": "sku",    "header": "SKU",    "width": { "mode": "fixed",   "value": 30 } },
    { "key": "name",   "header": "Name",   "width": { "mode": "auto" } },
    { "key": "qty",    "header": "Qty",    "width": { "mode": "fixed",   "value": 18 } },
    { "key": "amount", "header": "Amount", "width": { "mode": "fixed",   "value": 30 },
      "cell": { "text": { "text_align": "right" } } }
  ],
  "rows": [
    { "sku": "A001", "name": "Widget", "qty": 2, "amount": "$120.00" },
    { "sku": "A002", "name": "Gadget", "qty": 1, "amount": "$60.00" }
  ],
  "header": {
    "show": true,
    "repeat_on_page_break": true,
    "cell": { "fill": { "color": "#F3F4F6" }, "text": { "font_weight": "bold" } }
  },
  "grid": {
    "horizontal": { "color": "#D1D5DB", "width": 0.2 },
    "vertical": false
  }
}
```

Top-level fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `x` | `number` | **Yes** | Top-left X (mm). |
| `y` | `number` | **Yes** | Top-left Y (mm). |
| `width` | `number` | No | Total table width. Required when any column uses `percent` or `auto`. |
| `columns` | `TableColumn[]` | **Yes** | Column definitions. At least 1. |
| `rows` | `TableRow[]` | **Yes** | Row data. May be empty. |
| `cell` | `TableCellStyle` | No | Default cell style for the whole table. |
| `header` | `TableHeaderConfig` | No | Column header configuration. |
| `row_header` | `TableZoneConfig` | No | Row-header zone configuration. |
| `body` | `TableBodyConfig` | No | Body-zone configuration. |
| `grid` | `TableGridConfig` | No | Grid lines. |
| `pagination` | `TablePaginationConfig` | No | Page-break behaviour. |
| `z_index`, `comment` | | No | Common element fields. |

#### 4.10.1 Column

```json
{
  "key": "amount",
  "header": "Amount",
  "width": { "mode": "fixed", "value": 30 },
  "role": "data",
  "cell": { "text": { "text_align": "right" } },
  "header_cell": { "text": { "font_weight": "bold" } }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `key` | `string` | **Yes** | Unique within the table. |
| `header` | `string` | No | Default empty string. |
| `width` | `TableColumnWidth` | **Yes** | One of `fixed | percent | auto`. |
| `role` | `string` | No | `data` (default) or `row_header`. |
| `cell` | `TableCellStyle` | No | Per-column body cell style. |
| `header_cell` | `TableCellStyle` | No | Per-column header cell style. |

`TableColumnWidth`:

```json
{ "mode": "fixed",   "value": 30 }
{ "mode": "percent", "value": 25 }
{ "mode": "auto" }
```

Rules:

- `columns[].key` must be unique.
- `role = "row_header"` columns must be **contiguous on the left**. You may have multiple row-header columns.

#### 4.10.2 Row and cell

`rows` is an array of objects keyed by `columns[].key`.

Shorthand cell (scalar value):

```json
{ "name": "Apple", "qty": 2, "enabled": true, "note": null }
```

Complex cell:

```json
{
  "group": {
    "content": "Fruit",
    "row_span": 2,
    "col_span": 1,
    "style": { "text": { "font_weight": "bold" } },
    "link": { "target": { "type": "url", "url": "https://gpdf.com/docs/api-reference/#4102-row-and-cell" } }
  }
}
```

Complex cell fields:

| Field | Type | Notes |
|-------|------|-------|
| `content` | `string \| number \| boolean \| null \| BlockTextContent` | Cell content. Scalars, or block text under the table profile (§4.6.4). |
| `row_span` | `integer` | `>= 1`. Merge downward. |
| `col_span` | `integer` | `>= 1`. Merge rightward. |
| `style` | `TableCellStyle` | Per-cell override. |
| `link` | `LinkSpec` | Cell-level link. Only on complex cells. |

Rules:

- `null` renders as an empty string. `boolean` renders as `"true"` / `"false"`.
- Spans cannot exceed the table boundary or overlap each other. Violations return `API-002`.
- Rows containing a key not declared in `columns[].key` return `API-002` (no silent ignore).

#### 4.10.3 Cell style

```json
{
  "padding": { "x": 1, "y": 1 },
  "text": { "font_size": 9, "color": "#111111" },
  "fill": { "color": "#FFFFFF" },
  "content_offset_x": 1.5,
  "content_offset_y": 0.5,
  "borders": {
    "top": false,
    "right": { "color": "#111111", "width": 0.2 },
    "bottom": { "color": "#111111", "width": 0.2 },
    "left": false
  }
}
```

Borders may be `false` (suppress) or a full `StrokeStyle`. Diagonal borders
are `diagonal_tl_br` and `diagonal_bl_tr`.

#### 4.10.4 Header / row header / body zones

Header:

```json
{
  "show": true,
  "repeat_on_page_break": true,
  "rows": [
    {
      "cells": [
        { "content": "Product", "col_span": 2 },
        { "content": "Stock", "row_span": 2 }
      ]
    }
  ],
  "cell": {
    "fill": { "color": "#F3F4F6" },
    "text": { "font_weight": "bold" }
  }
}
```

Row header:

```json
{
  "cell": {
    "fill": { "color": "#F8F8F8" },
    "text": { "font_weight": "bold" }
  }
}
```

Body:

```json
{
  "cell": {},
  "alternate_fill": { "color": "#FAFAFA" }
}
```

Rules:

- Grouped column headers go in `header.rows`. Leaf headers come from `columns[].header`.
- `header.rows[].cells[].content` and `columns[].header` accept the same value types as body cells.
- The top-left corner cell is the leftmost row-header column's own `header` value.
- `header.show = false` ignores `header.rows`, `columns[].header`, and `header_cell`.

#### 4.10.5 Grid

```json
{
  "top":    { "color": "#111111", "width": 0.3 },
  "right":  { "color": "#111111", "width": 0.3 },
  "bottom": { "color": "#111111", "width": 0.3 },
  "left":   { "color": "#111111", "width": 0.3 },
  "horizontal": { "color": "#D1D5DB", "width": 0.2 },
  "vertical":   false
}
```

Each side accepts `false` (suppress) or a full `StrokeStyle`. Double lines
use `compound: { "kind": "double", "gap": <mm> }` on the stroke.

#### 4.10.6 Pagination

```json
{
  "keep_spans_together": true,
  "row_min_height": 10,
  "header_min_height": 12
}
```

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `keep_spans_together` | `boolean` | `true` | Required `true` whenever any cell has `row_span > 1`. |
| `row_min_height` | `number` | — | Minimum body row height (mm). |
| `header_min_height` | `number` | — | Minimum header row height (mm). |

#### 4.10.7 Width and style precedence

Width:

- `columns.length >= 1`.
- All `columns[].width` use `fixed`, `percent`, or `auto`.
- If `table.width` is set:
  - `fixed` columns reserve their mm.
  - `percent` columns take a share of `table.width`.
  - `auto` columns absorb the remainder, sized by content measurement.
  - With no `auto` column, the resolved widths must fill `table.width` exactly (otherwise `API-002`).
- If `table.width` is omitted, all columns must be `fixed`.
- The sum of `percent` widths cannot exceed `100`.

Style precedence (later wins):

1. `settings.defaults`
2. `table.cell`
3. Zone-level `header.cell` / `row_header.cell` / `body.cell`
4. `columns[].cell` / `columns[].header_cell`
5. Per-cell `cell.style`

Border precedence (later wins):

1. `grid`
2. `table.cell.borders`
3. Zone-level `cell.borders`
4. Column-level `cell.borders` / `header_cell.borders`
5. Per-cell `cell.style.borders`

### 4.11 Stack and block

`stack` solves the "table followed by trailing content" problem common in
invoices and statements. It composes a table with one or more block sections
that flow under it.

```json
{
  "type": "stack",
  "gap": 6,
  "children": [
    {
      "type": "table",
      "x": 18,
      "y": 123,
      "width": 180,
      "columns": [
        { "key": "description", "header": "Description", "width": { "mode": "auto" } },
        { "key": "amount",      "header": "Amount",      "width": { "mode": "fixed", "value": 40 },
          "cell": { "text": { "text_align": "right" } } }
      ],
      "rows": [
        { "description": "Cloud compute",   "amount": "$823.10" },
        { "description": "Replica storage", "amount": "$214.50" },
        { "description": "Cache warmup",    "amount": "$306.05" }
      ]
    },
    {
      "type": "block",
      "elements": [
        { "type": "text", "x": 128, "y": 0, "content": "Subtotal" },
        { "type": "text", "x": 168, "y": 0, "content": "$1,343.65" },
        { "type": "line", "x1": 128, "y1": 7, "x2": 178, "y2": 7 }
      ]
    }
  ]
}
```

Top-level fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `gap` | `number` | No | Vertical gap (mm) between consecutive children. Default `0`. |
| `children` | `StackChild[]` | **Yes** | Length `>= 2`. |

Rules:

- `stack` may only appear directly inside `pages[].elements`.
- `children[0]` must be a `table`.
- `children[1..]` must be `block` elements.

Block element:

```json
{
  "type": "block",
  "elements": [
    { "type": "text", "x": 128, "y": 0, "content": "Subtotal" },
    { "type": "text", "x": 168, "y": 0, "content": "$1,343.65" }
  ]
}
```

Rules:

- `block` does not accept `x` / `y` / `width` / `height`.
- `block.elements[].x` keeps the existing body-element semantics.
- `block.elements[].y` is relative to the block's start.
- `block` cannot nest `table`, `stack`, or another `block`.
- Block height is measured from its inner elements.

Pagination:

- The table paginates using its own remainder logic.
- Trailing blocks only start laying out **after** the table is fully rendered.
- A block that does not fit on the current page moves entirely to the next page. The preceding `gap` is dropped on the new page.
- A block taller than the available page height returns `API-002`.

### 4.12 Header and footer

`header` and `footer` are page-global decorations applied to every page.

```json
{
  "header": {
    "height": 14,
    "elements": [
      {
        "type": "text",
        "x": 12,
        "y": 8,
        "content": "Monthly Report",
        "style": {
          "font_family": "NotoSans-Regular",
          "font_mode": "prefer",
          "font_size": 10,
          "font_weight": "bold",
          "color": "#111827",
          "width": 80
        }
      }
    ]
  },
  "footer": {
    "height": 12,
    "elements": [
      {
        "type": "text",
        "x": 150,
        "y": 6,
        "content": "Page 1 / 12",
        "style": {
          "font_family": "NotoSans-Regular",
          "font_mode": "prefer",
          "font_size": 8,
          "color": "#6B7280",
          "width": 40,
          "text_align": "right"
        }
      }
    ]
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `height` | `number` | **Yes** | Region height (mm). |
| `elements` | `Element[]` | No | Region elements. May be empty. |

Coordinate semantics:

- `header.elements` use absolute page coordinates. By convention place them inside `y ∈ [0, header.height]`.
- `footer.elements` are auto-shifted by `page.height - footer.height` at render time. Write them as if `y = 0` is the top of the footer region.
- Header height does **not** push body elements down; body elements still use absolute coordinates.
- Footer height **is** subtracted from the auto-paginated body region, so body overflow starts a new page above the footer.

Recommended:

- Set `header.height` and `footer.height` close to actual content height. Oversized regions waste body space.
- For per-page-different headers/footers, do not use these global regions; embed the content in `pages[].elements` instead.

Text inside `header` / `footer` follows the **section profile** (§4.6.4): no
`list`, no `page_break`, no `frame.overflow = paginate`, no multi-column.

### 4.13 Layers

`layers` are document-level decorative layers. They do not participate in
body pagination and do not consume `header` / `footer` height.

```json
{
  "layers": {
    "background": {
      "repeat": "all_pages",
      "elements": [
        {
          "type": "rect",
          "x": 0, "y": 0,
          "width": 215.9, "height": 279.4,
          "fill": { "color": "#FFFBEB" }
        }
      ]
    },
    "watermark": {
      "repeat": "all_pages",
      "opacity": 0.12,
      "template": {
        "type": "text",
        "content": "PRIVATE COPY"
      },
      "style": {
        "font_family": "NotoSans-Regular",
        "font_size": 10.5,
        "font_weight": "bold",
        "color": "#B91C1C",
        "width": 56,
        "text_align": "center"
      },
      "layout": {
        "preset": "diagonal_tile",
        "angle": 330,
        "gap_x": 18, "gap_y": 16,
        "offset_x": 8, "offset_y": 10,
        "stagger_x": 34
      }
    },
    "stamp": {
      "repeat": "last_page",
      "elements": [
        {
          "type": "circle",
          "cx": 171, "cy": 228, "r": 18,
          "stroke": { "color": "#B91C1C", "width": 0.9 }
        },
        {
          "type": "text",
          "x": 152, "y": 220,
          "content": "PAID",
          "style": {
            "font_size": 12,
            "font_weight": "bold",
            "color": "#B91C1C",
            "width": 38,
            "text_align": "center"
          }
        }
      ]
    }
  }
}
```

Three layer slots:

| Slot | Render order | Spec shape | Use for |
|------|--------------|------------|---------|
| `background` | Behind body | `repeat`, `elements[]` | Page colour, paper backgrounds, decorative frames. |
| `watermark` | Above body | Algorithmic spec: `template` + `style` + `layout` + `opacity` | Diagonal-tiled "DRAFT" / "PRIVATE COPY" / brand text. |
| `stamp` | Top-most | `repeat`, `elements[]` | "PAID" / approval marks, often on the last page. |

Common fields:

- `repeat`: `all_pages` (default), `first_page`, `last_page`.

`background` / `stamp` element rules:

- Allowed types: `text`, `image`, `rect`, `line`, `circle`, `ellipse`, `polygon`, `link`.
- Forbidden types: `table`, `stack`.

`watermark` rules:

- `template.type` is currently only `text`.
- `layout.preset`: `center`, `tile`, `diagonal_tile`.
- `opacity` is in `[0, 1]`.
- `layout.angle` accepts any degree value and is intended for tiling.

### 4.14 Settings

```json
{
  "settings": {
    "defaults": {
      "text":   { "font_family": "NotoSans-Regular", "font_size": 11, "color": "#111111" },
      "stroke": { "color": "#000000", "width": 0.4 },
      "fill":   { "color": "#FFFFFF", "opacity": 1.0 },
      "shape":  { "corner_radius": 0 }
    },
    "metadata": {
      "title": "gPdf Document",
      "author": "Acme Cloud Inc."
    },
    "output": {
      "mode": "file",
      "file_name": "invoice-20260310.pdf"
    },
    "profile": "pdfa-2b",
    "page_margin": { "top": 10, "right": 12, "bottom": 10, "left": 12 }
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `defaults` | `Defaults` | Global default styles. See §4.14.1. |
| `metadata` | `Metadata` | PDF metadata. See §4.14.2. |
| `output` | `OutputSettings` | Response shape. See §4.14.3. |
| `profile` | `string` | PDF/A profile. See §6.4. |
| `page_margin` | `PageMargin` | Global margin. See §4.3.2. |
| `e_invoice` | `EInvoiceSettings` | **Only valid on `POST /api/v1/e-invoice/render`.** Sending it to JSON Render returns `API-002`. See §5. |
| `security` | `SecuritySettings` | PDF password + permission protection. See §4.14.4. |

#### 4.14.1 Defaults

```json
{
  "defaults": {
    "text":   { "font_family": "NotoSans-Regular", "font_size": 11, "color": "#111111" },
    "stroke": { "color": "#000000", "width": 0.4 },
    "fill":   { "color": "#FFFFFF", "opacity": 1.0 },
    "shape":  { "corner_radius": 0 }
  }
}
```

| Subkey | Type | Notes |
|--------|------|-------|
| `text` | `TextStyle` | Default text style. If `font_family` is set here without `font_mode`, `strict` is used. |
| `stroke` | `StrokeStyle` | Default stroke for shapes and table grids. |
| `fill` | `FillStyle` | Default fill. Default opacity is `0` (transparent) when omitted. |
| `shape` | `ShapeDefaults` | `corner_radius` (mm) for default rounded rectangles. |

`settings.defaults` only accepts the four nested groups above. The legacy
flat fields `font_family`, `font_size`, `color`, `unit` are rejected with
`API-002`.

#### 4.14.2 Metadata

```json
{
  "metadata": {
    "title": "Monthly Report",
    "author": "Acme Cloud Inc.",
    "subject": "Operations digest",
    "creator": "OpsBot 4.2",
    "producer": "gPdf",
    "language": "en"
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `title` | `string` | |
| `author` | `string` | |
| `subject` | `string` | |
| `creator` | `string` | The application that created the source content. |
| `producer` | `string` | The renderer. |
| `language` | `string` | BCP-47 language tag (e.g. `en`, `de`, `zh-Hans`). |

Normalisation:

- Empty or whitespace-only strings are treated as not provided.
- A token's policy may strip metadata fields it has no permission for; the request still succeeds.
- Stripped fields fall back to the policy's `default_metadata`.
- `title`, `creator`, `producer`, `language` use system fallbacks when no value or default exists.
- `author` and `subject` remain empty if no value or default exists.

#### 4.14.3 Output

```json
{
  "output": {
    "mode": "file",
    "file_name": "invoice-20260310.pdf"
  }
}
```

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `mode` | `"binary" \| "file"` | `binary` | Both return the same PDF bytes. Differs in `Content-Disposition`. |
| `file_name` | `string` | Auto-generated `gPdf-MMDDHHmmssSSS.pdf` | Sanitised; `.pdf` is appended automatically. |

Behaviour:

- `binary` (or omitted): `Content-Disposition: inline; filename="..."`. Browsers preview inline.
- `file`: `Content-Disposition: attachment; filename="..."`. Browsers download.
- Any value other than `binary` / `file` returns `API-002`.

#### 4.14.4 Security (password + permissions)

`settings.security` turns on PDF standard-security-handler encryption
on the output PDF. **Only valid on `POST /api/v1/pdf/render`.** Mutually
exclusive with `settings.profile` (PDF/A) and `settings.e_invoice` —
combining either returns `API-002`.

```json
{
  "settings": {
    "security": {
      "algorithm": "aes_256",
      "open_password": "***",
      "owner_password": "***",
      "permissions": {
        "print": true,
        "modify": false,
        "copy": false,
        "annotate": false,
        "fill_forms": true,
        "extract_accessibility": true,
        "assemble": false,
        "print_high_quality": true
      }
    }
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `algorithm` | `"aes_128" \| "aes_256"` | Default `aes_128`. AES-128 = standard-security-handler revision 4 (`R=4`, `V=4`, AESV2 crypt filter, PDF 1.7 output). AES-256 = revision 6 (`R=6`, `V=5`, AESV3 crypt filter, PDF 2.0 output). |
| `open_password` | `string` | Password required to open the PDF. Omitted, `null`, empty or whitespace-only = encryption disabled. Max 32 UTF-8 bytes. |
| `owner_password` | `string` | Owner password granting full rights regardless of `permissions`. Must differ from `open_password`. Max 32 UTF-8 bytes. **Enterprise policy.** |
| `permissions` | `Permissions` | 8 booleans, default `true`. Restricting any flag requires `owner_password`. **Enterprise policy.** |

**Permissions** (each defaults to `true`):

| Flag | Effect when `false` |
|------|---------------------|
| `print` | Printing is blocked. |
| `print_high_quality` | High-quality printing is blocked (a low-res render may still be allowed by `print`). |
| `modify` | Content edits other than annotations / form-field fill are blocked. |
| `copy` | Selecting and copying text / graphics is blocked. |
| `annotate` | Adding or modifying annotations AND form-field definitions is blocked. |
| `fill_forms` | Filling existing form fields is blocked (independent of `annotate`). |
| `extract_accessibility` | Extracting text / graphics for accessibility tools is blocked. |
| `assemble` | Inserting / rotating / deleting pages, and creating bookmarks / thumbnails, is blocked. |

**Tier policy:**

| | Pro | Enterprise |
|---|---|---|
| Algorithms | AES-128 only | AES-128 or AES-256 |
| `open_password` | ✓ | ✓ |
| `owner_password` | — | ✓ |
| `permissions` | — | ✓ |

**Behaviour:**

- The metadata stream is encrypted alongside the rest of the document; `/EncryptMetadata` writes `true`.
- The token's xAdmin PDF Policy must allow `document.security.allow = true`, and `document.security.allowed_algorithms` must contain the requested algorithm.
- A `settings.security` object that contains no valid `open_password`, no `owner_password`, and no restricted `permissions` is a no-op — the request renders an unencrypted PDF and does NOT enter the encryption write path.
- Unrecognised `algorithm` values are rejected at JSON deserialisation with `API-001` (`settings.security.algorithm must be aes_128 or aes_256`).
- All other `settings.security` misuses (password >32 UTF-8 bytes, policy disallow, combination with `profile` / `e_invoice`, semantic violation) return `API-002` with the offending field in `message`.

### 4.15 Default value precedence

When a field is omitted from an element, gPdf walks this chain to fill it in:

1. The element's own field (e.g. `line.stroke.width`).
2. `settings.defaults` (e.g. `defaults.stroke.width`).
3. Service-level defaults.

Font-family inheritance is **not** "fall back to auto when missing". It is
"keep walking up the chain until a font is declared, and only enter auto
mode if the entire chain is silent". Practical consequences:

- An element with `font_family` set continues to use that family even if children omit it.
- Children inherit the explicit family until one explicitly overrides.
- Auto mode only activates when no ancestor has set a family at all.

---

## 5. E-Invoice Render API

The e-invoice family — Factur-X / ZUGFeRD packaging, capabilities, render,
async strict-validation jobs, and artifact download — has its own dedicated
reference at **[/docs/e-invoice-api/](/docs/e-invoice-api/)**.

It moved out of this document on 2026-05-09 to give the e-invoice flows
(`inline_pdf` vs `object`, `basic` vs `strict` validation, residency
profiles) the room they need without bloating the JSON Render reference.


## 6. Reference

### 6.1 Error codes

All gPdf errors share the JSON envelope shown in [§1.1](#11-request-and-response-basics).
Codes fall into four families: client (`API-0xx`), authentication (`API-1xx`),
billing/entitlement (`API-2xx`), and rendering/system (`API-5xx` / `API-9xx`).

| Code | HTTP | Family | Trigger | Typical `message` | What to do |
|------|------|--------|---------|-------------------|------------|
| `API-001` | `400` | Client | Body is not valid JSON. | `Invalid JSON payload` | Validate with `jq .` or a JSON linter before sending. |
| `API-002` | `400` | Client | Body parsed but failed schema or business validation. | `<field> must be >= 0`, `Missing required field <name>`, `Field type mismatch` | Read `message`. The field name is always included. |
| `API-004` | `400` | Client | Total page count exceeds the per-request limit (token policy or platform max, whichever is smaller). | `page count exceeds max_pages_per_request` | Split the request into smaller batches, or request a higher policy limit. |
| `API-007` | `400` | Client | Embedded image bytes exceed the per-image limit set by the active token policy. | `image bytes exceeds max_image_bytes` | Re-encode the image at a smaller size, or use `source.kind = "asset"` to reference a pre-uploaded asset. |
| `API-008` | `413` | Client | Request body exceeds the platform body limit (default `16 MiB`; some deployments differ). | `Request body too large` | Reduce inline payload (especially base64 images). Consider splitting the document. |
| `API-101` | `401` | Auth | `Authorization` header is missing or not in `Bearer <token>` form. | `Missing or malformed Authorization header` | Add the header. The format is exactly `Bearer ` followed by the token. |
| `API-102` | `401` | Auth | Authentication failed (unknown token, environment mismatch, signature failure). | `Authentication failed` (redacted) | Verify the token belongs to the environment you are calling. |
| `API-103` | `401` | Auth | Token is blacklisted (revoked, suspended, or otherwise invalidated). | `Authentication failed` (redacted) | Rotate the token via the Console, or contact support. |
| `API-201` | `402` | Billing | No active subscription entitlement for this token. | `No active subscription` | Activate or renew a plan in the Console. |
| `API-202` | `402` | Billing | Subscription expired. | `Subscription expired` | Renew in the Console. |
| `API-203` | `402` | Billing | Subscription quota exceeded. | `Quota exceeded` | Wait for the next billing cycle, top up, or upgrade the plan. |
| `API-204` | `402` | Billing | Wallet balance insufficient for overage. | `Insufficient wallet balance` | Top up via the Console. |
| `API-501` | `500` | Render | PDF generation failed during rendering. | Detailed message describing the cause | Inspect `message`. Often points to invalid font, asset, or coordinate. |
| `API-502` | `500` | Render | PDF/A compliance check failed after rendering. | `PDF/A compliance check failed: <reason>` | Adjust the offending field (commonly fonts not in the embedded set, or non-PDF/A images). |
| `API-503` to `API-507` | `500` | Render | Specific rendering subsystem failures (font, asset resolution, layout). | Detailed message | Inspect `message`. These preserve actionable detail intentionally. |
| `API-504` | `500` | Render | Font resolution exhausted all fallbacks (auto / `prefer` mode). | `Font fallback failed for: <run>` | Provide a `font_family` that covers the script, or upload the font as an asset. |
| `API-900` | `500` | System | Internal system error. | Redacted message | Retry once. If it persists, contact support with the `req_id`. |
| `API-999` | `500` | System | Unknown internal error. | Redacted message | Same as `API-900`. |

Notes on validation errors (`API-002`):

- `API-002` is the most common error. Common triggers include:
  - `x` and `x_anchor` provided on the same element (mutually exclusive).
  - `font_mode` provided without a same-level `font_family`.
  - Explicit font in `strict` mode that does not cover the submitted text.
  - Invalid `link` (unsupported URL scheme, page index out of bounds, malformed `padding` / `border`).
  - Invalid `table` (unknown column key, `table.width` cannot allocate a positive width to undeclared columns, invalid span).
  - Invalid `profile` value.
  - Invalid `settings.security` (password >32 UTF-8 bytes, policy doesn't permit algorithm, combined with `settings.profile` or `settings.e_invoice`, or `permissions` provided without `owner_password`). Unrecognised `algorithm` value is rejected at JSON deserialisation with `API-001`.

Notes on redaction:

- `API-102`, `API-103`, and `API-9xx` deliberately use generic messages. They will not tell you whether a token exists or why authentication failed. This is by design — to defeat token enumeration.
- `API-201` to `API-204` (billing) preserve actionable text so end users know whether to renew, top up, or wait for the cycle.
- `API-501` to `API-507` (render) preserve actionable text so engineering can debug.

### 6.2 Limits

Three kinds of limits apply to every request:

1. **Platform limits** — fixed across all tenants, set per environment.
2. **Policy limits** — bound to the token, set when the plan is provisioned.
3. **Request-shape limits** — encoded in the request schema itself.

| Limit | Default | Scope | Override path | Triggered error |
|-------|---------|-------|---------------|-----------------|
| Request body size | `16 MiB` | Platform | Per-deployment env var; some private deployments raise it. | `API-008` |
| Pages per request | No platform default — entirely policy-driven | Token policy | Plan or per-token policy in the Console. | `API-004` |
| Image bytes per element | No platform default — only enforced if the policy sets `max_image_bytes` | Token policy | Plan or per-token policy in the Console. | `API-007` |
| Template batch size (`data` array) | `10` items | Platform | Not configurable. Split into multiple requests if you need more. | `API-002` (`Template render data item count ... exceeds max 10`) |
| URL TTL for e-invoice artifacts | `900` seconds | Request | `delivery.url_ttl_seconds`, range `1..900` | `API-002` if out of range |
| Retention for e-invoice artifacts | `23` hours | Request | `retention.ttl_hours`, range `1..23` | `API-002` if out of range |
| `xml.content` (e-invoice) | `2 MiB` | Platform | Not configurable. | `API-002` |

How to pre-check on the client side:

- Sum your request body bytes and reject above ~`14 MiB` before sending. This leaves headroom for proxies and avoids `API-008`.
- If you embed images via `source.kind = "base64"`, the base64 form is ~33% larger than the raw bytes. A 5 MiB raw JPEG becomes ~6.7 MiB in JSON.
- If you batch templates, never put more than `10` items in `data`. Use multiple HTTP calls for larger batches.

### 6.3 Response headers

| Header | Direction | Always present | Notes |
|--------|-----------|----------------|-------|
| `Content-Type` | Response | Yes | `application/pdf` on success, `application/json` on error. E-invoice job and artifact endpoints also return `application/json`. |
| `Content-Disposition` | Response | Yes (PDF responses only) | `inline; filename="..."` by default; `attachment; filename="..."` when `output.mode = "file"`. |
| `X-Request-Id` | Both | Yes | Echoed from the request if the client supplied it; otherwise generated. |

Headers gPdf does **not** currently emit (and the absence is intentional in v1):

- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- `Retry-After`
- `Idempotency-Key` echo
- `X-Render-Time-Ms`
- `X-Render-Engine-Version`

Do not depend on these. If a future version adds any of them, this document
will list them and clients can opt in.

### 6.4 Fonts and PDF/A profiles

Font resolution modes:

- **Auto** (no `font_family` declared anywhere in the chain): the renderer chooses fonts that cover the text from the bundled font set.
- **`prefer`** (declared `font_family` + `font_mode = "prefer"`): the renderer tries the declared family first; if it cannot cover all glyphs, it falls back through bundled families.
- **`strict`** (declared `font_family` + `font_mode = "strict"`, or `font_family` declared at the `defaults` / element level without `font_mode`): the renderer must use the declared family. If the family cannot cover the text, returns `API-002`.

Failure modes:

- Strict miss → `API-002`. Adjust the text or pick a font that covers it.
- Auto / prefer total fallback miss → `API-504`. The bundled set could not cover the text in any family.

Bundled fonts cover Latin, Greek, Cyrillic, CJK (Simplified Chinese, Japanese,
Korean), Arabic, Devanagari, Bengali, Thai, plus a JetBrains-style monospace.
Custom fonts can be uploaded as assets via the Console and referenced by
`font_family`.

PDF/A profiles:

| Profile | Use case |
|---------|----------|
| `pdfa-1b` | PDF/A-1b. Long-term archival, oldest baseline. |
| `pdfa-2b` | PDF/A-2b. Common archival profile. |
| `pdfa-3b` | PDF/A-3b. Required for embedded XML (e-invoice). |
| `pdfa-4` | PDF/A-4. Newer archival profile. |
| `pdfa-2u` | PDF/A-2u. Unicode-mapped variant of 2b. |
| `pdfa-3u` | PDF/A-3u. Unicode-mapped variant of 3b. |
| `pdfa-ua1` | PDF/UA-1. Accessibility profile. |

Set the profile via `settings.profile`. The chosen profile gates which
features can be used (e.g. transparency, embedded files). Violations return
`API-502`.

### 6.5 Coordinates and units

- Length unit: **millimetre** (mm). Floating-point values are accepted; the renderer rounds to PDF user-space at output.
- Origin: top-left of the page, or the content box if `page_margin` is set.
- X axis: rightward. Y axis: downward.
- `rotation` is in degrees, clockwise. Most elements accept only `0/90/180/270`. `text` and `image` accept any integer angle. Barcodes and their attached `barcode_text` accept only the four cardinal angles.

---

## 7. Changelog

This document tracks the public API contract. Internal changes that do not
affect callers are not listed here.

### 2026-05-08

- First publication of the consolidated public API reference.
- Documented the e-invoice job and artifact endpoints (`GET /api/v1/e-invoice/jobs/{job_id}` and `.../artifacts/{artifact}`).
- Single consolidated error-code reference at §6.1.

(Future entries will be added as the API evolves.)
