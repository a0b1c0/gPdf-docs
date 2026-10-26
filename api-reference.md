# Render API

> Status: Public API contract.
> Last updated: 2026-10-07
>
> This document defines what callers of the gPdf HTTP API can rely on. It does
> not describe internal control-plane behaviour, infrastructure, or any field
> not listed here. Anything not documented here is **not** part of the public
> contract and may change without notice.

---

## 1. Overview

gPdf turns JSON into PDF at the edge. Start with the smallest JSON Render
request in [§1.2](#12-five-second-start-minimum-json-render-request), then
choose Fixed Canvas for coordinate-first documents or Flow Document for
content-first, automatically paginated documents. Add defaults, metadata,
tables, containers, charts, PDF/A, security, or e-invoice settings only when
the document needs them.

This English document is the canonical prose contract for the Render API. The
machine-readable contract is the OpenAPI file linked below. Localized docs must
derive from this English source and must not introduce independent API meaning.

### 1.1 API surface and route map

There are three public API families, each tuned for a different integration
shape:

| API | Endpoint | When to use |
|-----|----------|-------------|
| JSON Render | `POST /api/v1/pdf/render` | You choose a Fixed Canvas `pages` root or a Flow Document `flow_document` root. Best for designers, custom reports, and programmatic documents. |
| Template Render | `POST /api/v1/template-render` | You only send `template_id` + business `data`. Best for ERP, OMS, WMS, and any system that wants a stable contract per document type. |
| E-Invoice Render | `POST /api/v1/e-invoice/render` | You need a Factur-X / ZUGFeRD compliant PDF/A-3b with embedded CII XML. |

A simple decision tree:

- "I want precise coordinate control." → JSON Render with `pages`
- "I want content to create and paginate pages." → JSON Render with `flow_document`
- "My team agreed on a template name and a list of fields." → Template Render
- "I need an EU-mandate-compliant electronic invoice." → E-Invoice Render

JSON Render and Template Render return `application/pdf` on success.
E-Invoice Render returns either `application/pdf` or an object-delivery job
descriptor, depending on `settings.e_invoice.delivery.mode`. All three share
the same Bearer-token authentication and the same error-code namespace.

> **Machine-readable spec:** the full API contract is published as an
> OpenAPI 3.1 document at [`/openapi.json`](/openapi.json)
> (YAML mirror at [`/openapi.yaml`](/openapi.yaml)). Use it to generate
> SDKs (`openapi-generator`), import into Postman / Insomnia, drive IDE
> autocomplete, or hand to an AI coding assistant — the OpenAPI is the
> canonical contract this document narrates. The root `openapi` value is the
> OpenAPI Specification version; `info.version` is the gPdf public contract
> document version.

Public route map:

| Route | Auth | Purpose |
|-------|------|---------|
| `POST /api/v1/pdf/render` | `Authorization: Bearer YOUR_TOKEN` | Primary JSON Render endpoint. |
| `POST /api/v1/template-render` | `Authorization: Bearer YOUR_TOKEN` | Render one or more business records through a published template. |
| `POST /api/v1/e-invoice/render` | `Authorization: Bearer YOUR_TOKEN` | Render a Factur-X / ZUGFeRD e-invoice PDF. |
| `GET /api/v1/e-invoice/jobs/{job_id}` | `Authorization: Bearer YOUR_TOKEN` | Poll an object-delivery e-invoice job. |
| `GET /api/v1/e-invoice/jobs/{job_id}/artifacts/{artifact}` | `Authorization: Bearer YOUR_TOKEN` | Download `pdf`, `xml`, `request`, or `report` artifacts. |
| `GET /api/v1/e-invoice/capabilities` | None | Read the public e-invoice capability registry. |

Operational notes:

- `POST /api/v1/pdf/render` does not accept `settings.e_invoice`; e-invoice settings belong only to `POST /api/v1/e-invoice/render`.
- `POST /api/v1/e-invoice/render` returns different content by `delivery.mode`: `inline_pdf` returns PDF bytes, while `object` returns a JSON job descriptor.
- Object-mode job lookup and artifact download require a token for the same owner that created the job.
- `GET /api/v1/e-invoice/capabilities` is a static product capability registry. It does not guarantee that a specific token is authorized for every listed capability.
- JSON parsing, request validation, token policy application, preparation, and rendering are one server-side render flow. They are not separate public endpoints.

<span id="3-quick-start"></span>

### 1.2 Five-second start: minimum JSON Render request

Many integrations do not need a complete `DocumentRequest` on day one. If you
only want to prove that JSON can become a PDF, save this as `quickstart.json`:

```json
{
  "pages": [
    {
      "size": "label_100_150",
      "elements": [
        {
          "type": "text",
          "layout": { "left": 10, "top": 18 },
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

You should get a 100 × 150 mm one-page PDF on disk. If you do not:

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `401` with `API-101` | Missing or malformed `Authorization` header | Confirm the header is exactly `Bearer YOUR_TOKEN`. |
| `403` with `API-102` | Token rejected | Confirm the token is active and valid for the public API. |
| `400` with `API-001` | JSON syntax or request-shape decoding failed | Check JSON syntax, unknown fields, enum values, union shapes, and parser safety bounds. |
| `400` with `API-002` | Body decoded but failed semantic or business validation | Read `message` — it names the offending relationship, policy, or aggregate bound. |
| Non-2xx response with JSON saved as `quickstart.pdf` | `--output` saved the error body under a PDF filename | Inspect the existing response body and HTTP status; repair the reported error before resending. |

The example intentionally omits:

- `settings`
- `defaults`
- `text.style`
- `output`

gPdf fills in the safe style and output defaults needed to produce a PDF. Use
the full structure in [§3](#3-document-model-and-layout-choices) and
[§4](#4-json-render-api) when you need metadata, default styles, PDF/A
profiles, layers, headers, footers, or pagination behaviour.

The public `text` element accepts three content forms:

- Plain text shorthand: `"content": "Hello gPdf"`.
- Rich text spans: `"content": { "spans": [...] }`.
- Block text: `frame` + `defaults` + `content.blocks`.

All three forms pass through the same validation and rendering pipeline. Block
text is the most expressive form for complex layout, variables, list content,
and controlled pagination.

### 1.3 Common default fallbacks

When a request omits a field, gPdf resolves values in this order:

1. Element-local fields.
2. `settings.defaults`.
3. Service-level defaults.

The high-frequency cases are:

| Omitted field | Behaviour |
|---------------|-----------|
| Text font family | Plain text / `spans` inherit through `text.style.font_family`; block text inherits through `defaults.run.font_family`. If the whole chain is silent, automatic font selection is used. |
| Text font size | Falls back through the same style/default chain to the service default. |
| Text color | Falls back through the same style/default chain to the service default. |
| Text line height | Plain text / `spans` inherit through `text.style.line_height`; block text inherits through `defaults.paragraph.line_height`. |
| `line.stroke` | Missing stroke width and color are filled from defaults. |
| Shape `stroke` | `rect`, `circle`, `ellipse`, `polygon`, and `path` do not draw an outline unless `stroke` is declared; partially declared stroke fields still inherit missing subfields. |
| `fill` | Transparent by default; no fill is drawn unless declared. |
| `settings.output.mode` | Defaults to `binary`. |
| `settings.output.file_name` | A safe file name is generated and `.pdf` is appended. |

See [§4.16](#416-default-value-precedence) for the full precedence chain.

### 1.4 Complete request skeleton

Move from the minimum request to the full shape when you need document-wide
defaults, metadata, repeated layers, a header/footer, security, PDF/A, or page
margins:

```json
{
  "settings": {
    "defaults": {
      "text": {
        "font_family": "NotoSans-Regular",
        "font_size": 11,
        "color": "#111111"
      },
      "stroke": {
        "color": "#000000",
        "width": 0.4
      },
      "fill": {
        "color": "#FFFFFF",
        "opacity": 1
      },
      "shape": {
        "corner_radius": 0
      }
    },
    "metadata": {
      "title": "gPdf Document",
      "author": "gPdf"
    },
    "output": {
      "mode": "file",
      "file_name": "archive-report-20260310.pdf"
    },
    "profile": "pdfa-2b"
  },
  "layers": {
    "background": {
      "elements": []
    },
    "watermark": {
      "template": {
        "type": "text",
        "content": "Draft"
      },
      "layout": {
        "preset": "diagonal_tile"
      }
    },
    "stamp": {
      "elements": []
    }
  },
  "header": {
    "layout": { "height": 12 },
    "elements": []
  },
  "footer": {
    "layout": { "height": 10 },
    "elements": []
  },
  "pages": [
    {
      "size": "label_100_150",
      "elements": [
        { "type": "text", "layout": { "left": 8, "top": 24 }, "content": "Full request skeleton" }
      ]
    }
  ]
}
```

This skeleton is not required for simple documents. It is a map of the major
surfaces that can participate in a production document.

### 1.5 AI sandbox and trial testing

For AI coding assistants such as Cursor, Copilot, custom GPTs, and developers
who want to quickly test or debug `DocumentRequest` JSON payloads without
registering or managing API keys, gPdf provides a public sandbox proxy endpoint.

```http
POST /api/playground?endpoint=pdf-render
Host: gpdf.com
Content-Type: application/json
Accept: application/pdf
```

> [!NOTE]
> **Sandbox guidelines and policies:**
> 1. **No Authorization header required**: this trial sandbox automatically binds a secure developer key at the CDN edge. Do **not** include an `Authorization` header.
> 2. **Development and trial only**: this endpoint is restricted to local debugging, layout validation, and interactive AI evaluation. It is **not** for production workloads.
> 3. **Fair use**: there is no fixed daily sandbox quota at this time, but reasonable fair-use applies. If expected usage exceeds **100 PDF renders per day**, register in the gPdf Console and use the paid API endpoint with a live token.
> 4. **Rate limits and payload boundaries**:
>    - **Rate limit**: maximum **60 requests per minute per IP address**. Exceeding this returns `429 Too Many Requests`.
>    - **Max request size**: request JSON is limited to **256 KB**.
>    - **Output format**: the response is an `application/pdf` binary stream.
>    - The sandbox IP rate limit is separate from production token throughput and does not imply production entitlement. On sandbox `429`, wait and retry with client-side exponential backoff plus jitter; do not retry immediately in a tight loop.
> 5. **Registered Free tier**: after signing up, Free-tier accounts receive **100 free PDF page credits per day** for evaluation with a live token. A one-page PDF consumes one page credit; multi-page PDFs consume one credit per generated page. Free-tier requests may be intentionally paced and are typically about **1 second slower per request** than paid plans.
> 6. **Commercial use**: for production traffic, higher throughput, or advanced features, register in the gPdf Console and use a paid live token.

<span id="11-request-and-response-basics"></span>

### 1.6 Request and response basics

Every request uses:

```http
POST /api/v1/<route>
Host: api.gpdf.com
Content-Type: application/json
Authorization: Bearer YOUR_TOKEN
X-Request-Id: <optional-client-id>
```

Every successful response carries:

- `Content-Type: application/pdf`
- `Content-Disposition: inline; filename="..."` (default) or `attachment; filename="..."` when `output.mode = "file"`; Unicode or legacy-unsafe names also include RFC 5987 `filename*=UTF-8''...`
- `X-Request-Id: <echoed-or-generated>`

Every error response carries:

- `Content-Type: application/json`
- `X-Request-Id: <echoed-or-generated>`

The error body always has these four fields:

```json
{
  "error": true,
  "code": "API-002",
  "message": "pages[0].elements[0].layout.gap_after must be a finite number >= 0",
  "req_id": "7f7d2f5a-4cb0-4c4e-b6ef-8f6d3e0b1fd8"
}
```

When the request needs a capability that the target build does not compile or
enable, the body adds an optional fifth field, `capability`:

```json
{
  "error": true,
  "code": "API-002",
  "message": "layout.flow_document is not enabled by this build profile",
  "req_id": "7f7d2f5a-4cb0-4c4e-b6ef-8f6d3e0b1fd8",
  "capability": "layout.flow_document"
}
```

| Field | Type | Notes |
|-------|------|-------|
| `error` | `boolean` | Always `true` on the error envelope. |
| `code` | `string` | Public error code. See [§6.1](#61-error-codes). |
| `message` | `string` | Human-readable explanation. Some auth and system errors return a redacted message. |
| `req_id` | `string` | Mirrors `X-Request-Id`. Always present. Use it when filing support tickets. |
| `capability` | `string` | Optional; absent on every other error. The capability id that the request needs and the target build does not compile or enable. Today these rejections are `data.projection` and `layout.flow_document` (`API-002`), including a Template Render runtime that requires one of them. A Math or e-invoice request on a build without that capability fails request decoding with `API-001` and no `capability`, and a Chart rejection carries only its `message`. Clients that do not use the field can ignore it. |

#### Capability discovery

A deployment serves the set of capabilities its build compiles. Before sending
a request that needs `layout.chart`, `layout.math`, `data.projection` or
`layout.flow_document`, check the target:

- The hosted API answers `POST /health` with JSON: `enabled_capabilities` lists
  the capability ids the deployment serves, and `catalog_version` is the
  version of the capability id vocabulary.
- Native servers answer `POST /health` with the body `OK` and send the same
  facts as headers: `X-GPDF-Enabled-Capabilities` (comma-separated ids) and
  `X-GPDF-Catalog-Version`. The Wasmer package also answers with `OK` and
  `X-GPDF-Enabled-Capabilities`; it adds `X-GPDF-Catalog-Version` from its
  next release.

Treat the ids as a set; their order is not part of the contract.
`catalog_version` is a gPdf-owned integer. It changes when a capability id is
added or removed or its meaning changes, never because a deployment compiles a
different set of capabilities, and it is unrelated to the OpenAPI
`x-gpdf-contract-version`. Neither value shows whether a token is authorized
for a capability.

`render.svg`, added in catalog version 2, is the rendering of SVG images,
emoji and OpenType SVG glyphs in PDF output. Every deployed gPdf build that
renders PDFs compiles it, so a deployment that reports `catalog_version` 2 lists
it; one that reports an earlier version or none renders SVG the same way
without listing the id.

### 1.7 What this document does not cover

The following are intentionally **not** part of the public contract:

- Internal storage layout, queue topology, or any specific cloud service used to deliver responses.
- Token issuance, billing, quota provisioning, and policy management. Those happen in the gPdf Console, not over this API.
- Template authoring (how to design and publish a template). See the internal `template-authoring.en.md` if you maintain templates.
- Beta endpoints not listed in this document.

---

## 2. Authentication and Environments

### 2.1 Environments

gPdf exposes one public customer-facing API base URL:

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Production | `https://api.gpdf.com` | Live traffic. SLAs apply. |

Build your client so the base URL is configuration, not a constant. Most
integrations read it from an environment variable like `GPDF_BASE_URL`.
Internal debug and test domains are not part of the public API contract and are
intentionally omitted from public docs and OpenAPI.

### 2.2 Authentication

Every render endpoint requires a Bearer token:

```http
Authorization: Bearer YOUR_TOKEN
```

Rules:

- The token is opaque. Do not parse it.
- Use the issued token unchanged; do not add a prefix. The current HTTP parser accepts a 32-character lowercase hexadecimal value, and malformed token values return `API-101`.
- Tokens are scoped to the account and deployed API surface. A token not valid for the public API is rejected with `API-102`.
- Tokens may carry policy constraints (max pages per request, allowed PDF/A profiles, allowed e-invoice standards). Constraint violations return `API-002` with the offending field named in `message`.
- A blacklisted token returns `403` / `API-103` with a redacted message. Rotate it or contact support. Subscription expiry is a separate entitlement condition (`402` / `API-202`), not a token-format or blacklist error.

The single endpoint that does not require authentication is
`GET /api/v1/e-invoice/capabilities`. See the dedicated
[E-invoice API reference](/docs/e-invoice-api/#1-capabilities).

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
`Retry-After`).

Usage reporting is internal and does not make PDF render requests idempotent.

Recommended client behaviour today:

- Cap concurrency per token at a number you negotiated with your gPdf account contact. Most production tokens are sized for sustained 5-20 req/s; bursts above that should be queued client-side.
- On `5xx` or network error, retry with exponential backoff (initial 500 ms, max 5 s, max 3 attempts).
- On non-throttling `4xx`, do **not** retry. The request will fail the same way every time. Inspect `code` and `message` and fix the request.
- Sandbox IP throttling returns HTTP `429` without implying subscription quota state. For automated sandbox callers, queue requests and retry with exponential backoff plus jitter, for example start at 1 second and cap at 30 seconds, to avoid parallel retry storms. Authenticated production quota exhaustion returns HTTP `429` with `API-203`; treat it as an entitlement condition and wait for reset, top up, or upgrade rather than blindly retrying.
- For at-most-once semantics on PDF generation, deduplicate on your side before calling. The API does not detect duplicate submissions.

---

## 3. Document model and layout choices

The JSON Render request is intentionally small at the top level. A document
defines exactly one source root: `pages` for Fixed Canvas or `flow_document`
for Flow Document.

```json
{
  "settings": { },
  "datasets": { },
  "projections": { },
  "layers": { },
  "header": { "layout": { "height": 12 }, "elements": [] },
  "footer": { "layout": { "height": 10 }, "elements": [] },
  "pages": [
    {
      "size": "label_100_150",
      "elements": [
        { "type": "text", "layout": { "left": 8, "top": 24 }, "content": "Body content" }
      ]
    }
  ]
}
```

Read the Fixed Canvas example as five surfaces:

| Surface | Role |
|---------|------|
| `settings` | Document-wide defaults, metadata, output mode, page margins, PDF/A profile, security, and route-specific e-invoice settings. |
| `layers` | Repeated background, watermark, and stamp content. |
| `header` / `footer` | Page-global decorations with fixed heights. |
| `pages[]` | Ordered pages. Each page defines size or width/height and its body elements. |
| `elements[]` | The actual visual objects: text, table, container, barcode, image, shapes, and links. |

### 3.1 Source root: Fixed Canvas or Flow Document

The two roots are authoring modes, not two rendering engines. Both enter the
same resolved document plan, layout pass, Preview/LayoutPlan representation,
and PDF replay path.

| Mode | Root | Choose it when | Page ownership |
|------|------|----------------|----------------|
| Fixed Canvas | `pages` | Coordinates and explicit physical pages are part of the design. | The source owns each page. |
| Flow Document | `flow_document` | Content structure, reusable page masters, automatic pagination, columns, layout groups, and named styles are primary. | The layout engine derives physical pages. |

Rules:

- `pages` and `flow_document` are mutually exclusive and exactly one is
  required. Sending neither fails JSON decoding with `API-001`, and so does
  sending both to a deployment that serves Flow Document.
- A Flow Document cannot use root `layers`, `header`, or `footer`; those
  surfaces belong to each Page Master. Root `settings`, `datasets`, and
  `projections` remain shared.
- `settings.layout.flow` and `settings.layout.page_margin` are Fixed Canvas
  controls and are rejected for Flow Document. Flow blocks participate in
  document flow by definition, and each Page Master owns its margin.
- Flow Document availability is capability-gated. Use it only when `/health`
  includes `layout.flow_document`. Contract files alone are not deployment
  proof. The shipping-label profile does not advertise the capability.
- A deployment that does not list `layout.flow_document` does not decode
  `flow_document`. If it reports `catalog_version` 2 or later, every request
  that carries one, valid or not and with or without `pages`, gets `API-002`
  with `capability` `layout.flow_document`, unless the body is not valid
  JSON, another root field fails decoding, or the request needs
  `data.projection` (root `datasets` or `projections`), which is checked
  first; a Template Render runtime with a Flow Document layout gets the
  Template Render form of that rejection before binding, whether or not it
  declares the capability. An older deployment without the capability may
  reject the root as an unknown field with `API-001`.
- Generated continuation pages are output facts. Do not serialize Preview
  pages back into `flow_document` or target a guessed generated page number.
  Use named logical destinations for Flow links.

Layout WASM/Preview output may include `pages[].clip_ranges`; an empty list is
omitted, and absence means no command-range clips. Each entry applies to the
zero-based, half-open `draw_commands[command_start..command_end]` range, using
the physical-page rectangle `x`, `y`, `width`, `height` in millimetres and an
optional corner `radius` in millimetres. Preview consumers must apply every
applicable clip, including nested clips, when painting these commands. Clipping
is visual only: original draw/text content remains and may still be extracted;
it is not redaction. These are output fields, not `DocumentRequest` inputs.

Layout WASM/Preview output may also include top-level `internal_links`, resolved
from named logical destinations by the same layout pass as `pages`. The array
is omitted when empty; absence means no exported internal-link hit regions.
Each entry contains a zero-based source physical `page_index`, a hit rectangle
`bounds` (`x`, `y`, `width`, `height`), and a `target` with a zero-based physical
`page_index` and `x`, `y`. Coordinates and sizes are in millimetres, measured
from the respective page's top-left corner. Physical page indexes are not
display page numbers, which may restart at a segment boundary. URL links are
not included in this array. Consumers must replace these regions and targets
after each successful layout and must not use coordinates from an older source
revision for navigation. These resolved output fields do not add fields to
`DocumentRequest` or replace its source links and logical destinations.

See [§4.2.1](#421-flow-document-contract) for the complete Flow Document
source contract.

### 3.2 Fixed Canvas layout choice: element, table, or container

Within Fixed Canvas, choose the smallest layout primitive that matches the
document:

| Need | Use |
|------|-----|
| A single absolute-positioned object | A normal element in `pages[].elements`. |
| Rows and columns with headers, row heights, spans, and page splitting | `table`. |
| A table followed by subtotal, tax, grand total, payment terms, or signature blocks | A body `table` followed by one or more sibling body `container` elements. Use `layout.flow` and `layout.gap_after` to keep them in source order. |
| A card, address block, status badge, totals block, clipped panel, local default-style scope, or group that should move as one unit | `container`. |
| A published business contract where callers send only `template_id + data` | Template Render API, not JSON Render. |
| Factur-X / ZUGFeRD output with embedded CII XML | E-Invoice Render API. |

Default body layout is explicit positioning. Elements placed directly in
`pages[].elements` keep their own `left/top` coordinates and do not move down just
because an earlier table, text block, or container became taller. There are two
exceptions to that base rule:

- Use `settings.layout.flow` or element-level `layout.flow` when you want body elements to
  be planned in source order on the vertical axis. `flow` is a body-only planner;
  it does not affect `header`, `footer`, `layers`, or watermark content. Effective
  flow, whether inherited or explicit, forbids `layout.top` and `layout.bottom`.
  Vertical placement comes from source order, `settings.layout.start_top`, actual
  element height, and `layout.gap_after`; it does not infer gaps from coordinates.
- For table-followed-by-content layouts, place the `table` and following
  `container` elements as siblings in `pages[].elements`; set `layout.flow:
  true` on each item or enable `settings.layout.flow`, and use the table's
  `layout.gap_after` to create the vertical space before the totals, notes, or
  signature container.

Important boundaries:

- Each explicit Fixed Canvas source page owns an independent layout run. An
  overflowing body can generate continuation pages before the next explicit
  source page starts. Body flow does not convert `pages` into `flow_document`.
- `container` may contain normal visual descendants and nested `container`
  elements, but not `table`; keep the table and the following visual group as
  sibling body elements.
- `layers` reject `table`. `header` and `footer` accept it only for rare
  repeated tabular page furniture; keep body line-item tables in
  `pages[].elements`.

Use this rule for invoices: line items are a `table`; subtotal, tax, grand
total, payment instructions, and approval/signature rows are sibling
`container` elements that follow the table in source order. Use `layout.flow`
and `layout.gap_after` to keep the vertical spacing stable when rows paginate
or text wraps.

---

## 4. JSON Render API

`POST /api/v1/pdf/render` is the lower-level rendering endpoint. The request
body is a single `DocumentRequest` JSON object that describes either explicit
Fixed Canvas pages or a Flow Document source. The caller has full control over
the selected authoring model.

If you want a higher-level integration where you only send a template ID and
business data, use `POST /api/v1/template-render` (see `template-api.en.md`).

### 4.1 Endpoint

| Property | Value |
|----------|-------|
| Method | `POST` |
| Path | `/api/v1/pdf/render` |
| Auth | Required — `Authorization: Bearer YOUR_TOKEN` |
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
| `datasets` | `object<string, DatasetDefinition>` | No | Named typed source rows for Table and Chart projections. A runtime must advertise `data.projection` before this field is used. See §4.9.10. |
| `projections` | `object<string, TableProjection \| ChartProjection>` | No | Named consumer-specific mappings. A Table and a Chart may consume the same Dataset without reading one another. See §4.9.10. |
| `layers` | `Layers` | Fixed Canvas only | Background / watermark / stamp. See §4.13. |
| `header` | `Section` | Fixed Canvas only | Global page header. See §4.12. |
| `footer` | `Section` | Fixed Canvas only | Global page footer. See §4.12. |
| `pages` | `Page[]` | One source root | One or more explicit Fixed Canvas pages. Mutually exclusive with `flow_document`. |
| `flow_document` | `FlowDocument` | One source root | Content-first source with Page Masters and derived physical pages. Mutually exclusive with `pages`; requires `layout.flow_document`. |

The smallest valid request:

```json
{
  "pages": [
    {
      "size": "label_100_150",
      "elements": [
        { "type": "text", "layout": { "left": 8, "top": 20 }, "content": "Hello gPdf" }
      ]
    }
  ]
}
```

When you omit `settings`, gPdf applies safe defaults for fonts, strokes,
fills, and output mode. You do not need to write a 200-line `DocumentRequest`
to get a usable PDF.

#### 4.2.1 Flow Document contract

Flow Document replaces manual page creation with a typed source tree. A Page
Master defines reusable page geometry and repeated regions; a Content Segment
selects one master and supplies logical content; the layout engine derives the
physical page sequence.

A minimal request is:

```json
{
  "flow_document": {
    "page_masters": {
      "body": {
        "size": "a4",
        "layout": {
          "page_margin": { "top": 18, "right": 16, "bottom": 18, "left": 16 }
        }
      }
    },
    "segments": [
      {
        "id": "main",
        "page_master": "body",
        "content": [
          {
            "type": "block",
            "semantics": { "role": "heading", "heading_level": 1 },
            "element": {
              "type": "text",
              "content": "Quarterly report",
              "style": { "font_size": 20 }
            }
          },
          {
            "type": "block",
            "element": {
              "type": "text",
              "content": "Body content creates continuation pages automatically.",
              "style": { "font_size": 10, "width": 178 }
            }
          }
        ]
      }
    ]
  }
}
```

Flow elements do not need Fixed Canvas `layout.top` placement. A normal
`block` occupies the next logical body slot. Existing element contracts remain
the visual primitives; Flow Document adds typed page, group, pagination,
semantic, and style ownership around them.

##### Root, registries, and source limits

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `page_masters` | `object<string, FlowPageMaster>` | Yes | `1..=32` author definitions. Names match `^[a-z][a-z0-9_]{0,63}$`. |
| `segments` | `FlowContentSegment[]` | Yes | `1..=100`, processed in source order. Each segment starts on a new physical page after all continuation pages of the previous segment. |
| `styles` | `object<string, FlowStyleDefinition>` | No | At most `64` typed definitions. Names are non-empty, have no leading/trailing whitespace, and are at most `128` UTF-8 bytes. |
| `style_scope` | `FlowStyleScope` | No | Document-level ordered selector rules. |

The full Flow source is bounded to `1024` content nodes, `1024` source
elements, `1024` logical destinations, and nesting depth `32`. Repeated
elements owned by one Page Master are bounded to `32`. Limits are validated
before layout; the runtime does not truncate content.

##### Page Masters

A Page Master defines `size` plus optional `orientation`, or custom `width`
plus `height` in millimetres. These geometry forms are mutually exclusive.
Custom sides are `10..=2000 mm`. `orientation` is `portrait` or `landscape` and
is available only with a named preset.

```json
{
  "size": "a4",
  "orientation": "landscape",
  "layout": {
    "page_margin": { "top": 14, "inside": 18, "bottom": 14, "outside": 12 },
    "binding_side": "left",
    "columns": { "count": 2, "gap": 8, "direction": "left_to_right" }
  },
  "header": {
    "layout": { "height": 10 },
    "elements": []
  },
  "footer": {
    "layout": { "height": 10 },
    "elements": [
      { "type": "text", "content": "{page} / {total_pages}", "layout": { "left": 0, "top": 1 } }
    ]
  }
}
```

Page Master fields:

| Field | Notes |
|-------|-------|
| `layout.page_margin` | Owns Flow body margins. Use physical `left/right` or binding-aware `inside/outside`, never both. `binding_side` is required with binding-aware margins and forbidden otherwise. |
| `layout.columns` | Optional page-body columns: `count` `1..=32`, non-negative `gap`, and `left_to_right` or `right_to_left` logical direction. |
| `style_scope` | Ordered rules applied after document scope and before segment/group scopes. |
| `header` / `footer` | Repeated regions with an explicit `layout.height` and bounded element list. Their heights plus margins must leave a positive body. |
| `layers` | Page-owned `background`, `watermark`, and `stamp`, using the same typed element/layer contracts as their Fixed Canvas counterparts. |
| `variants` | Optional `first`, `odd`, `even`, and `last` overrides. A header/footer variant replaces the corresponding base region's elements and therefore requires that base region. |

Flow repeated regions reserve body space in addition to the Page Master's
top and bottom margins. In physical page coordinates:

- The header starts at `layout.page_margin.top`.
- The body starts at `layout.page_margin.top + header.layout.height`.
- The body ends at `page_height - layout.page_margin.bottom - footer.layout.height`,
  which is also the footer's start. The footer ends before the bottom margin.

Omitted margins and absent regions contribute zero. These positions apply to
every Flow page, including continuation pages; Fixed Canvas
`settings.layout.pagination` continuation gaps do not apply. Header/footer
element coordinates remain section-local. Region heights reserve space but
do not automatically size or clip their elements.

`{page}` uses the segment's page-numbering policy. `{total_pages}` is the
final physical page count. Header, footer, and decorative layers are excluded
from body reading order by default.

##### Content Segments

```json
{
  "id": "appendix",
  "page_master": "landscape_appendix",
  "destination": "appendix_start",
  "page_numbering": { "mode": "reset", "start": 1 },
  "style_scope": {
    "rules": [
      { "selector": { "kind": "text", "role": "paragraph" }, "style": "body_text" }
    ]
  },
  "content": []
}
```

`id` and `page_master` use the lowercase registry-name grammar. `page_master`
must reference a declared master. `destination` is an optional logical link
target. `page_numbering` defaults to `{ "mode": "continue" }`; reset mode
requires a positive `start`. When the object is supplied, `mode` is required;
continue mode forbids `start`. Segment `content` may be omitted and defaults to
an empty array; an explicitly supplied value must be an array, not null.

##### Content nodes and groups

Every content node uses a `type` discriminator:

| `type` | Required payload | Optional fields | Behaviour |
|--------|------------------|-------------------|-----------|
| `block` | `element` | `destination`, `styles`, `semantics`, `pagination` | Places one existing element in normal document flow. |
| `break` | `kind` | None | `page` or `column`; a column break requires page or local columns. |
| `columns` | `layout` | `content`, `destination`, `style_scope`, `pagination` | Local sequential or balanced columns; omitted content defaults to an empty array. |
| `stack` | `layout` | `content`, `destination`, `style_scope`, `pagination` | Vertical or horizontal grouped layout with gap/alignment/wrap; omitted content defaults to an empty array. |
| `grid` | `layout` | `content`, `destination`, `style_scope`, `pagination` | Row-major fixed/fraction/content track grid; omitted content defaults to an empty array. |
| `placed` | `layout`, `element` | `destination`, `styles`, `semantics` | Explicit coordinate escape hatch relative to the page, body, or containing group. |

`columns.layout` accepts `count`, `gap`, `direction`, `mode`, and optional
`height`. `mode` is `sequential` or `balanced`; balanced columns do not accept
an explicit height. `stack.layout` accepts `axis`, `gap`, `main_align`,
`cross_align`, `wrap`, and optional `height`. A vertical wrapped or non-start
aligned stack requires `height`.

`grid.layout.columns` is a non-empty sequence of at most `32` tagged tracks:

```json
[
  { "mode": "fixed", "value": 42 },
  { "mode": "fraction", "value": 1 },
  { "mode": "content" }
]
```

Grid also accepts `column_gap`, `row_gap`, `direction`, and `strict_rows`.
Break nodes must occur at row boundaries. `strict_rows: true` rejects an
incomplete final row. Fixed tracks and gaps must fit the available inline
size.

Placed layout is:

```json
{
  "reference": "body",
  "bounds": "clip"
}
```

`reference` is `page`, `body`, or `group`; `group` requires a containing
layout group. `bounds` defaults to `error` and may be `visible` or `clip`.
Placed nodes do not accept flow pagination constraints; repeat-on-every-page
content belongs in a Page Master.

##### Pagination and semantic metadata

`pagination` is a typed object:

| Field | Type | Notes |
|-------|------|-------|
| `keep_together` | boolean | Defers the node when it can fit a fresh region; deterministic oversize failure prevents an infinite retry. |
| `keep_with_next` | boolean | Extends a bounded source-order keep chain, up to the document's `1024` node bound. |
| `break_before` / `break_after` | `page` or `column` | Column breaks require an active page/local column scope. |
| `orphan_lines`, `widow_lines` | integer `1..=32` | Splittable Text blocks only. |
| `min_lines_before_split`, `min_lines_after_split` | integer `1..=32` | Splittable Text blocks only. |

`semantics.role` is `paragraph`, `heading`, `list`, `list_item`, `figure`,
`table`, `caption`, or `note`. A `heading` requires positive
`heading_level`; `list` and `list_item` may carry positive `list_depth`.
`decorative: true` cannot be combined with role or level metadata. Logical
source order, not visual position or z-order, drives PDF/UA reading order.

##### Named styles and scope rules

Flow named styles are typed definitions, not an unvalidated property map:

```json
{
  "styles": {
    "body_text": {
      "target": "text",
      "properties": { "font_size": 10, "color": "#111827", "line_height": 1.4 }
    },
    "emphasis": {
      "target": "text",
      "extends": "body_text",
      "properties": { "font_weight": "bold" }
    },
    "report_table": {
      "target": "table",
      "properties": { "cell": { "padding": {
        "space": "physical",
        "left": {
          "value": 2,
          "unit": "mm"
        },
        "right": {
          "value": 2,
          "unit": "mm"
        },
        "top": {
          "value": 2,
          "unit": "mm"
        },
        "bottom": {
          "value": 2,
          "unit": "mm"
        }
      } } }
    },
    "callout_frame": {
      "target": "container_frame",
      "properties": { "fill": { "color": "#EFF6FF" }, "corner_radius": 3 }
    }
  }
}
```

Targets are `text`, `table`, and `container_frame`. Text properties use the
existing `TextStyle`; Table exposes typed `cell`; Container frame exposes
`fill`, `stroke`, `corner_radius`, and `padding`. `extends` must target the
same style kind, is cycle-free, and is bounded to depth `32`. Named-style
properties recursively reject null, including shared Insets padding and sides;
omit a property to inherit it. Container frame `corner_radius` must be finite
and non-negative.

A `style_scope.rules` sequence contains ordered `{selector, style}` entries.
Selectors may combine `kind`, `role`, `style_identity`, `heading_level`, and
`list_depth`. `heading_level` requires `role: "heading"`; `list_depth` requires
`role: "list"` or `"list_item"`; the two level fields cannot coexist. Selector
fields reject null. Scopes cascade from document to Page Master to Segment to nested
layout group. A Block or Placed node may then apply up to four explicit
`styles` in order, without duplicate references; element-local typed properties remain the final override. Across all
scopes a document accepts at most `256` rules, with at most `64` in one scope.
Unknown, target-incompatible, cyclic, or over-capacity styles fail closed.

### 4.3 Page

A page is described by its size and the elements it contains. Body elements
position themselves in millimetres from the page's top-left corner unless
`page_margin` is configured.

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `size` | `string` | One of `size` or `width+height` | Named preset. Input trims surrounding whitespace and ignores case; emit the canonical lowercase spelling. The same rule applies to Page Masters. |
| `width` | `number` | One of `size` or `width+height` | Custom page width in mm. Must be between `10` and `2000`. |
| `height` | `number` | One of `size` or `width+height` | Custom page height in mm. Must be between `10` and `2000`. |
| `layout.page_margin` | `PageMargin` | No | Per-page margin override inside `layout`; top-level `margin` is rejected, including null. |
| `elements` | `Element[]` | No | Body elements. May be empty. |

Rules:

- `size` and `width/height` are mutually exclusive on the same page. Providing both returns `API-002`.
- A page without `size` must provide both `width` and `height`.
- Unknown page fields are rejected. The public page layout container is `layout`.
- Custom `width` / `height` values are in millimetres and are limited to `10..=2000` per side. This covers standard paper, labels, engineering drawings, and common posters while catching common unit mistakes such as inches or pixels submitted as millimetres.

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
    {
      "size": "a4",
      "elements": [
        { "type": "text", "content": "A4 preset", "layout": { "left": 10, "top": 14 } }
      ]
    },
    {
      "size": "letter",
      "elements": [
        { "type": "text", "content": "Letter preset", "layout": { "left": 10, "top": 14 } }
      ]
    },
    {
      "width": 100,
      "height": 150,
      "elements": [
        { "type": "text", "content": "100 x 150 mm custom page", "layout": { "left": 8, "top": 14 } }
      ]
    }
  ]
}
```

#### 4.3.2 Page margin and content box

Configure margins globally for the whole PDF (`settings.layout.page_margin`) or override
selected edges per page (`pages[].layout.page_margin`):

```json
{
  "settings": {
    "layout": {
      "page_margin": { "top": 10, "right": 12, "bottom": 10, "left": 12 },
      "gap_after": 4
    }
  },
  "pages": [
    {
      "size": "label_100_150",
      "layout": {
        "page_margin": { "top": 8, "right": 10, "bottom": 12, "left": 10 }
      },
      "elements": [
        {
          "type": "text",
          "content": "Content box origin",
          "layout": { "left": 0, "top": 0 }
        }
      ]
    }
  ]
}
```

Behaviour:

- Without `settings.layout.page_margin` or `pages[].layout.page_margin`, body elements use absolute page coordinates and body `layout.left` values must not be negative.
- Once `settings.layout.page_margin` or `pages[].layout.page_margin` is set, body element `layout.left`/`layout.top` become relative to the **content box** (the area inside the margins).
- Body horizontal positioning is content-box relative when page margins are active: `layout.left: 0` starts at the effective left margin; negative `layout.left` may move content toward the paper edge but must not be less than `-effective_left_margin`; right-side content may enter the right margin but must not pass the physical page edge.
- Coordinate-placement elements that exceed the allowed horizontal page bounds or vertical content box return `API-002`. There is no automatic clipping. Final effective-flow fragments follow the element-specific pagination policies in §4.14; this input-coordinate check is not a guarantee that every Fixed Canvas drawing stays inside the body region.
- `header` and `footer` use section-local vertical coordinates. Explicit `layout.left` values are margin-relative when `settings.layout.page_margin` or `pages[].layout.page_margin` is active: rendered left = effective left margin + `layout.left`. Negative section `layout.left` values may move content toward the paper edge, but they must not be less than `-effective_left_margin`; without margins, negative section `layout.left` values are invalid. `layout.anchor` keeps normal anchor semantics. Layers remain physical-page coordinates.
- Auto-paginated overflow continues from the top of the next page's content box.
- Without page margins, auto-paginated overflow is rebased to `header.layout.height + settings.layout.pagination.continuation_top_gap_with_header` when `header.layout.height > 0`, otherwise to `settings.layout.pagination.continuation_top_gap`. This also applies to effective-flow continuations; `start_top` only initializes each explicit source page.

Common content boxes with `15mm` margins:

| Page size | Page size (mm) | Content box (mm) |
|-----------|----------------|------------------|
| A4 | `210 x 297` | `180 x 267` |
| Letter | `215.9 x 279.4` | `185.9 x 249.4` |
| Legal | `215.9 x 355.6` | `185.9 x 325.6` |

### 4.4 Coordinates and units

- Page coordinates and untagged geometry lengths are in millimetres (mm). Font sizes and typed Insets lengths use their documented units; Insets may explicitly use mm, pt, or em where its consumer permits them.
- The origin is the top-left corner of the page (or the content box if `settings.layout.page_margin` / `pages[].layout.page_margin` is set).
- The Horizontal axis goes right; the Vertical axis goes down.
- Layout tip: without page margins, `layout.left: 0` and `layout.top: 0` start at the physical page edge. This is valid for full-page backgrounds, edge-aligned decoration, or bleed-like elements, but regular visible content such as text, tables, barcodes, and business graphics should keep at least `5mm` from physical page edges for print safety. Prefer `settings.layout.page_margin` for document-level spacing, usually `15mm` for business documents, or set positive `layout.left/top` values when no page margin is configured.
- See [§6.5](#65-coordinates-and-units) for `rotation` rules per element.

#### 4.4.1 Shared padding: `Insets`

Every public `padding` field uses the same `Insets` object. `space` is required:

| Space | Optional sides |
|---|---|
| `physical` | `top`, `right`, `bottom`, `left` |
| `logical` | `block_start`, `inline_end`, `block_end`, `inline_start` |

Every supplied side requires both `value` (finite and non-negative) and `unit`
(`mm`, `pt`, or `em`). Mixed spaces, unknown fields, scalar padding, numeric
sides, untagged objects, and `x/y` or `inline/block` shorthands are rejected.

```json
{
  "padding": {
    "space": "physical",
    "top": { "value": 1, "unit": "mm" },
    "right": { "value": 2, "unit": "mm" },
    "bottom": { "value": 1, "unit": "mm" },
    "left": { "value": 2, "unit": "mm" }
  }
}
```

Container, Chart, Table, and Link padding accepts `physical` space and `mm` or
`pt` only: these geometric boxes have no intrinsic text-run direction or font
size. TextHighlight also accepts `logical` space and `em`, resolved after the
text run's font size and direction are known. With horizontal text, block start
and end map to top and bottom; inline start and end follow LTR or RTL.

Omitted or null sides remain absent during style inheritance. Null `padding`
is also treated as absent outside Flow named-style properties; that complete
typed style tree rejects null, including padding and its sides. The required
length `value` and `unit` cannot be null. Matching-space styles
merge per side; a different space replaces the whole object. Explicit zero
clears an inherited side. Final defaults remain consumer-specific: container
and link sides default to zero, while table sides default to 1 mm with a further
0.4 mm content safety inset per side. Padding shrinks a box's content region,
expands a text highlight background, or expands a link's click region according
to its existing consumer. It does not reposition the element itself.

`BlockTextFrame` does not support padding. `frame.padding` and
`defaults.frame.padding` are unknown fields and must be omitted, including
when their value is null. Page margins and ordered line endpoints are separate
contracts and do not use `Insets`.

#### 4.4.2 Presence, null, and validation boundaries

Field ownership is checked before input normalization. An unknown field or a
field forbidden in the current scope must be omitted; `null`, `false`, and `0`
do not bypass that rule. This is distinct from a supported optional field whose
documented null value means absence.

| Context | Rule |
|---------|------|
| Fixed Canvas settings and supported optional layout fields | Null means absent. Horizontal placement chooses one effective, non-null source among `left`, `right`, and `anchor`. |
| Effective Fixed Canvas flow | `top` and `bottom` are forbidden by presence. When effective flow is false, `gap_after` is forbidden by presence. |
| Header, footer, layers, and Container children | `flow` and `gap_after` are forbidden by presence. The direct parent's placement mode determines the other allowed layout fields. |
| Flow `block.element.layout` | The parent owns placement. Only applicable child-layout configuration is allowed; placement and `z_index` fields are forbidden, including zero or null. |
| Fixed-height Container | A non-null `height` forbids any `min_height` field. A null `height` selects automatic height, where `min_height` remains supported. |
| Page and Page Master geometry | Supplied `size`, `width`, and `height` must be non-null. Custom width/height cannot coexist with preset size or any `orientation` field. Preset orientation may be null. |
| Flow Page Master margins | Physical `left/right` and binding-aware `inside/outside` cannot coexist, even with null values. Presence of either binding-aware edge requires a non-null `binding_side`; otherwise `binding_side` is forbidden. Supported null edges remain absent. |
| Flow nodes and layouts | Foreign node fields are forbidden by presence. Concrete defaulted fields, such as columns direction, may be omitted but cannot be null. Ordinary optional node fields retain their documented null behavior. |
| Flow named-style properties | The complete typed property tree rejects null; omit an inherited property instead. |

OpenAPI checks request structure and the static constraints represented in its
schemas. Runtime validation remains required for references, recursive parent
scope, arithmetic across columns, available body geometry, and finite f32/PDF
unit conversion. A successful Schema check alone does not prove that layout or
rendering will succeed.

#### 4.4.3 Colour values

Every request colour uses one format: `#` followed by exactly six hexadecimal
digits (`#RRGGBB`); letters may be upper- or lower-case. This applies to text,
span, run, `highlight`, `decoration`, and `link_style` colours, link borders,
`fill` and `stroke` on shapes, containers, tables, and backgrounds, barcode
foreground, fill, and `barcode_text` colours, watermark text,
`settings.defaults`, Flow named styles, Chart colours, gradient stops, patterns,
and Display Math.

Any other form returns `API-002` with an error that names the field, for
example `pages[0].elements[0].fill.color must be a #RRGGBB hex color`. Rejected
forms include a missing `#` (`2563EB`), 3- or 8-digit values (`#fff`,
`#2563EB80`), names (`red`), `rgb()`, signs (`#+f+f+f`), a repeated `#`, and
leading or trailing whitespace. Express transparency with the matching
`opacity` field where the object has one.

### 4.5 Element types

The `pages[].elements` array, `header.elements`, `footer.elements`,
`layers.background.elements`, and `layers.stamp.elements` contain element
objects. Every element has a `type` discriminator.

| `type` | Section | Notes |
|--------|---------|-------|
| `text` | §4.6 | Plain, rich `spans`, or block text. |
| `math` | §4.5.2 | Body-only display formula; no calculation or verification. |
| `barcode` | §4.7 | 2D matrix and 1D linear formats. |
| `image` | §4.8 | Asset reference or inline base64. |
| `chart` | §4.9.9 | Bar, line, area, combo, pie, donut, waterfall, scatter/bubble, range, or heatmap chart. |
| `line` | §4.9 | Single line segment. |
| `rect` | §4.9 | Rectangle, optionally rounded. |
| `circle` | §4.9 | Circle by centre + radius. |
| `ellipse` | §4.9 | Ellipse by centre + radii. |
| `polygon` | §4.9 | Closed polygon from a point list. |
| `path` | §4.9 | Structured native vector path. |
| `link` | §4.9 | Standalone clickable hotspot. |
| `container` | §4.9 | Geometric group for cards, badges, clipped panels, and nested groups. |
| `table` | §4.10 | Tabular data with headers, spans, pagination. |

Headers and footers accept the body element union except `math`, including
`chart` and `table`, for repeated page furniture. `layers.background`,
`layers.stamp`, and container
descendants use the lightweight subset: `text`, `barcode`, `image`, `line`,
`rect`, `circle`, `ellipse`, `polygon`, `path`, `link`, `chart`, and `container`.
`table` is not allowed in layers or container descendants.

Parent-child hierarchy summary:

| Parent | Allowed children | Forbidden children / notes |
|--------|------------------|----------------------------|
| `pages[].elements` | Full body union: `text`, `math`, `barcode`, `image`, `chart`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `link`, `container`, `table` | Top-level body flow. |
| `header.elements`, `footer.elements` | Section union, including `chart` and `table` | No `math`; use for repeated page furniture, not body line-item tables or table-followed-by-content flows. |
| `layers.background.elements`, `layers.stamp.elements` | Lightweight subset: `text`, `barcode`, `image`, `chart`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `link`, `container` | No `math` or `table`. `layers.watermark` uses the separate watermark contract. |
| `container.elements` | Lightweight subset, including `chart` and nested `container` | No `math` or `table`; place body elements as siblings when needed. |

Layout ownership is determined by the direct parent. Document flow never leaks
through a container boundary:

| Direct parent | Placement owner | `flow` / `gap_after` | Position fields |
|---------------|-----------------|----------------------|-----------------|
| `pages[].elements` | Document body | Allowed | Forbidden while effective flow is `true`; required as applicable while effective flow is `false`. |
| `header.elements`, `footer.elements` | Section | Forbidden | Section-local coordinates. |
| `layers.background.elements`, `layers.stamp.elements` | Layer | Forbidden | Physical-page coordinates. |
| Coordinate `container.elements` | Container content box | Forbidden | Content-box-local coordinates; page anchors are forbidden. |
| Linear `container.elements` | Container layout | Forbidden | `left`, `top`, `right`, `bottom`, and `anchor` are forbidden. |

Common fields shared by most elements:

| Field | Type | Notes |
|-------|------|-------|
| `layout` | Scope-specific element layout | Positioning is owned by the direct parent. Only direct body children may use `flow` and `gap_after`; `container` adds `children` to manage its own direct descendants. |
| `comment` | `string` | Free-form note. Not rendered. |
| `rotation` | `number` | See per-element rules. Only elements whose section lists `rotation` accept it; `table` and `container` do not. `text` accepts arbitrary integer angles, `image` and `path` accept arbitrary numeric angles, and barcode/rect/ellipse use constrained renderer semantics. |
| `link` | `LinkSpec` | Make the element clickable. See §4.9.7. |

Hyperlink modes:

- Attach `link` to an element (`text`, `barcode`, `chart`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `image`, `container`).
- Use a standalone `type: "link"` hotspot when you need to overlay a clickable region.

#### 4.5.1 Horizontal anchor (`anchor`)

In coordinate-owned scopes, `layout.left`, `layout.right`, and `layout.anchor`
are the three horizontal positioning forms for box-like elements. `left` is the
left-edge coordinate. `right` positions the element by distance from the current
content right edge. `anchor` aligns an element relative to a named reference edge.
Direct Body flow elements still choose one horizontal form, while a child of a
linear Container omits all three because its parent owns the slot. They are
supported on `text`, `math`,
`barcode`, `chart`, `rect`, `path`, `image`, `link`, and `container`. They are
**not** supported on `line`, `circle`, `ellipse`, `polygon`, `table`,
or block-text content.

```json
{
  "type": "text",
  "content": "$1,235.85",
  "style": {
    "width": 24,
    "text_align": "right"
  },
  "layout": {
    "anchor": { "reference": "content_right", "offset": 8 },
    "top": 12
  }
}
```

Normal references for ordinary positioned elements:

| Reference | Meaning |
|-----------|---------|
| `page_left` | Left page edge. |
| `page_right` | Right page edge. |
| `content_left` | Left edge of the content box. Falls back to the page edge if no margin is set. |
| `content_right` | Right edge of the content box. |

Resolved horizontal position:

- Left references: `resolved_left = reference + offset`
- Right references: `resolved_left = reference - offset - element_width`
- `right`: `resolved_left = content_right - right - element_width`

Rules:

- `left`, `right`, and `anchor` are mutually exclusive. Sending more than one returns `API-002`.
- When using `right` or `anchor`, the element must have a width:
  - `text` (plain or `spans` shorthand): `style.width`
  - `text` (block): `frame.width`
  - `math`: its own required `width` field
  - `barcode`, `rect`, `path`, `image`, `link`, `container`: their own `width` field.

Vertical positioning in coordinate-owned scopes:

- `top` is the default top-edge coordinate.
- `bottom` is supported on the same box-like elements as `right`; it positions
  the element by distance from the current content bottom edge:
  `resolved_top = content_bottom - bottom - element_height`.
- `top` and `bottom` are mutually exclusive. Use one vertical positioning
  source per element.
- A direct Body element with effective `flow=true` omits both `top` and
  `bottom`; the Body cursor supplies its vertical position.
- A child of a linear Container also omits both fields; its parent supplies the
  complete slot position.
- For text, `bottom` requires an explicit `style.height` or block `frame.height`.
- For barcode, chart, rect, path, image, and link, `bottom` uses the element `height`.
- For container, `bottom` requires a fixed `height`; auto-height containers use
  `top`.
- `table` does not support `right` or `bottom`; keep using its normal
  `left/top` and flow semantics.
- `math` is auto-height and does not support `bottom`.

#### 4.5.2 Display Math

`math` renders a bounded formula source as display geometry. It does not
calculate, simplify, validate dimensions, or verify whether an equation is
true. It is a focused formula syntax, not general LaTeX.

```json
{
  "type": "math",
  "content": "\\frac{-b+\\sqrt{b^2-4ac}}{2a}",
  "width": 80,
  "style": {
    "font_size": 14,
    "color": "#123456",
    "opacity": 1,
    "align": "center"
  },
  "alt": "Quadratic formula",
  "layout": { "left": 15, "top": 30 }
}
```

RaTeX handles the supported ASCII LaTeX source, including scripts, fractions,
roots, sums, integrals, matrices, chemistry, and units notation. Use LaTeX
commands instead of direct Unicode symbols; non-ASCII source is rejected,
including inside text groups. Omit outer math delimiters. The runtime limit
is 4096 source bytes. This is a bounded rendering subset, not a general
LaTeX document processor; unsupported commands fail instead of being passed through.

Display `style.font_size` defaults to 11 pt. It must be finite and in
`0 < font_size <= 1000` pt; preparation rejects values above that limit.

An entire document may contain at most 256 unique prepared Math resources and
4 MiB (4,194,304 bytes) of estimated prepared drawing and fragment data in total. This is
a document-wide formula resource budget, not a PDF-size or font-file-size limit.

Math is accepted only in Fixed Canvas `pages[].elements` and Flow Document
block/placed body elements. It is rejected in headers, footers, layers,
containers, table cells, and `pdfa-ua1` output. RaTeX selects the bundled
KaTeX math fonts and vector geometry; there is no per-element `font_family`
or fallback setting. Ordinary text is literal; use a typed inline `math` node
for mixed paragraphs (see Inline nodes below). Display formulas preserve their
font size and wrap at supported mathematical boundaries, preferring relations
over binary operators. Fractions, roots, scripts, matrices, and paired groups
remain intact. If an indivisible part still exceeds the frame, rendering fails
instead of clipping or silently shrinking. Supported `aligned` and `gathered`
environments retain explicit rows; bare top-level `\\` and `\newline` are not
accepted. Independent `math` elements remain unbreakable across pages.
Use Math only when the target runtime's `/health` response includes
`layout.math`; contract files alone do not prove deployment, and the
shipping-label profile intentionally does not advertise this capability.
See the [display Math example](/gpdf-agent-examples/math-display.json) for a
complete request.

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

- `content` (string, `{ spans }`, or `{ blocks }`).
- Outside a linear Container, one of `layout.left`, `layout.right`, or
  `layout.anchor`.
- When the direct parent owns coordinate placement and effective Body flow is
  false, one of `layout.top` or `layout.bottom`. Body flow and linear Container
  children omit both vertical fields.

#### 4.6.1 Plain text

```json
{
  "type": "text",
  "layout": { "left": 18, "top": 18 },
  "content": "Hello gPdf"
}
```

With styling:

```json
{
  "type": "text",
  "layout": { "left": 18, "top": 18 },
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
  "layout": { "left": 18, "top": 30 },
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

Text content shape summary:

```text
content = string
        | { "spans": [ { "text": "...", "style": { ... }, "link": { ... } } ] }
        | { "blocks": [
            { "type": "paragraph", "style": { ... }, "inlines": [ ... ] },
            { "type": "list", "list": { ... }, "items": [ ... ] },
            { "type": "page_break" }
          ] }

paragraph.inlines[] = { "type": "text", "text": "..." }
                    | { "type": "variable", "name": "page", "scope": "system" }
                    | { "type": "line_break" }
                    | { "type": "tab" }
                    | { "type": "math", "content": "E=mc^2" }
```

Plain string text, span text, and table-cell text can use `{page}` and
`{total_pages}`. Do not write `${page}` or `${total_pages}`; the dollar sign
is literal while the variable still resolves separately. For structured block
text, prefer inline variable nodes such as
`{ "type": "variable", "name": "page", "scope": "system" }`.

Profile restrictions still apply: table-cell text and `barcode_text` allow
only paragraph blocks and do not allow tabs or inline links.

```json
{
  "type": "text",
  "layout": { "left": 8, "top": 20 },
  "frame": {
    "width": 84,
    "overflow": "paginate"
  },
  "defaults": {
    "run": {
      "font_family": "NotoSans-Regular",
      "font_mode": "prefer",
      "font_size": 8.5,
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
            "style": { "font_size": 12, "font_weight": "bold", "color": "#111827" }
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
| `layout.top` or `layout.bottom` | `number` | Scope-dependent | Required for coordinate placement; forbidden for effective Body flow and linear Container children. See §4.5.1. |
| `layout.left`, `layout.right`, or `layout.anchor` | | Scope-dependent | Choose one outside a linear Container; linear Container children omit all three. See §4.5.1. |
| `rotation` | `integer` | No | Any integer angle. |
| `layout.z_index` | `integer` | No | Signed 32-bit integer, from -2147483648 to 2147483647; default 0. Forbidden on Flow block elements, including an explicit 0. |
| `comment` | `string` | No | |
| `link` | `LinkSpec` | No | Element-level link. Mutually exclusive with any inline `link`. |
| `style` | `TextStyle` | No | For plain or `spans` form. |
| `frame` | `BlockTextFrame` | No | For block form. |
| `defaults` | `BlockTextDefaults` | No | Default `run` / `paragraph` / `frame` for the block tree. |
| `content` | `string \| { spans } \| { blocks }` | **Yes** | The text content in one of the three forms. |

Every text content form rejects top-level `width` and `height`, including null.
String and `spans` content use `style.width` / `style.height` and reject `frame`
and `defaults`, including null. Block content uses `frame.width` / `frame.height`
or `defaults.frame.width` / `defaults.frame.height`. Choose dimensions in the
field supported by the selected content form before using right, bottom, or
anchor placement.

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
- **Reserved in the schema but rejected by validation today**. Do not send
  these fields in runnable requests; the validator returns
  `paragraph layout features … not wired` so callers get an explicit error
  instead of silent layout differences:
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
continuation rules) are reserved in the schema but rejected by validation
today. Do not send them in runnable requests; the validator returns
`list spacing/continuation features … not wired` so callers get an explicit
error instead of silent layout differences.

`list` is only allowed in the **full text profile** (see §4.6.4). It is
rejected inside `header`, `footer`, layers, table cells, and barcode text.

##### Page break

```json
{ "type": "page_break" }
```

`page_break` is the only way to force a page break inside block text. The
older `\f` string convention is no longer supported.

##### Inline nodes

Five inline node types are public; `math` requires a math-enabled build:

| Type | Example |
|------|---------|
| `text` | `{ "type": "text", "text": "Hello", "style": { "font_weight": "bold" } }` |
| `variable` | `{ "type": "variable", "name": "page", "scope": "system" }` |
| `line_break` | `{ "type": "line_break" }` |
| `tab` | `{ "type": "tab" }` |
| `math` | `{ "type": "math", "content": "E=mc^2", "style": { "font_size": 12 }, "alt": "Mass-energy equivalence" }` |

Inline math is supported in Full body text only, including Flow paragraphs.
Section/header/footer/layer text, container text, table cells, barcode text,
rotated math-containing text, and `pdfa-ua1` output reject it explicitly.
Its optional `style` accepts `font_size` (pt), `color`, and `opacity`; `align`
is only valid on independent display math and is rejected on inline nodes.
Omitted inline size/color/opacity inherit the surrounding resolved text style.
The same source limits and supported RaTeX syntax as display math apply.

Inline formulas use mathematical text style, share the paragraph baseline,
and expand the natural line height when needed. Short formulas stay together
when practical. Long formulas use supported relation/binary-operator breaks;
protected structures are never split by raw characters. A remaining unbreakable
overflow fails explicitly. Literal `$...$` in a text node is never reparsed.
PDF and Studio replay the same prepared fragments, including line decisions.

Explicit paragraph `shrink_to_fit: true` scales text and formulas together and
requires a positive `min_font_size` for math-containing text. Both fields belong
on `text.style` or block-text `frame`, not on inline math style. Shrinking is off
by default. A fixed frame that is still too small at the minimum size fails;
formulas are not silently clipped or replaced with ellipses. Use a math-only
Full text paragraph when bounded proportional shrinking is required for a
formula. Flow paragraphs use normal line pagination; a formula line itself
does not split across pages.

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
- `highlight`, `decoration`, `link_style`

Plain `TextStyle` is the compatibility style object used by simple
`text.style`, `table.cell.text`, `columns[].cell.text`,
`columns[].header_cell.text`, `cell.style.text`, and `barcode_text.style`. It
contains three groups of fields:

| Group | Fields | Notes |
|-------|--------|-------|
| Run style | `font_family`, `font_size`, `font_weight`, `font_style`, `font_mode`, `color`, `opacity`, `letter_spacing`, `script`, `highlight`, `decoration`, `link_style`, `wrap_policy` | Visual run-level styling. |
| Paragraph shorthand | `text_align`, `direction`, `line_height` | Converted into paragraph defaults for compatibility text. |
| Frame-like shorthand | `width`, `height`, `vertical_align`, `text_overflow`, `shrink_to_fit`, `min_font_size` | Converted into a shadow `frame` for validation. Rejected in table-cell text and `barcode_text.style`. |

Rules:

- `font_mode` cannot appear without a same-level `font_family`.
- `font_mode = "auto"` is **not** a public input. Auto mode is implicit when no font is declared anywhere in the inheritance chain.
- `strict` failure (declared font cannot cover the text) returns `API-002`.
- `auto` or `prefer` total fallback failure returns `API-504`.
- `highlight` is the inline text-run background. It accepts `color`,
  `opacity`, `corner_radius`, and shared `Insets` padding (physical or logical).
- `wrap_policy` is a `TextStyle` field only. Legal values are `normal`,
  `keep_together`, and `no_wrap`. It is valid in plain `TextStyle` locations
  such as `text.style`, `table.cell.text`, `columns[].cell.text`,
  `columns[].header_cell.text`, `cell.style.text`, and `barcode_text.style`.
  Do not put it in `ParagraphStyle`, `InlineTextStyle`, table column width
  modes, or path watermark styles.
- Do not write `style.align` in simple text, table-cell text,
  `barcode_text.style`, or any `TextStyle` location. Use `style.text_align`
  there. The field name `align` belongs to block `ParagraphStyle` and
  `MathStyle` only.

##### Block text frame

`frame` fields:

- `width`, `height` (finite mm values strictly greater than 0)
- `vertical_align`: `top | middle | bottom`
- `overflow`: `visible | clip | ellipsis | paginate`
- `shrink_to_fit` (boolean)
- `min_font_size` (finite number strictly greater than 0)

Rules:

- The six supported fields are optional and accept null as omitted.
- Only the fields listed above are accepted in `frame` and `defaults.frame`.
  `padding`, `stroke`, `fill`, `columns`, and `column_gap` are not supported;
  supplying any of them, including null, is rejected as an unknown field.
- `frame` is rejected inside table cells and barcode text.
- `rotation != 0` cannot combine with `frame.overflow = "paginate"`.
- `frame.height` cannot combine with `page_break`.
- `frame.overflow ∈ { clip, ellipsis }` cannot combine with `page_break`.
- `frame.overflow = "paginate"` cannot combine with `shrink_to_fit = true`.
- `header` / `footer` / layer text cannot use `paginate`.

#### 4.6.4 Text feature profiles

The exact subset of text features available depends on **where** the text
appears. There is no `profile` field in the JSON; the profile is implied by
container.

| Profile | Used in | Allowed blocks | Allowed inlines |
|---------|---------|----------------|-----------------|
| Full | `pages[].elements[]` and Flow body text | `paragraph`, `list`, `page_break` | `text`, `variable`, `line_break`, `tab`, `math` |
| Section | `header`, `footer`, `layers.background`, `layers.stamp` | `paragraph` | `text`, `variable`, `line_break`, `tab` |
| Table | Inside `table` cells | `paragraph` | `text`, `variable`, `line_break` |
| Barcode | Inside `barcode_text` | `paragraph` | `text`, `variable`, `line_break` |

`layers.watermark` does not use the normal `text` element profiles. It uses the
separate `WatermarkLayer` contract: the public template type is currently
`text`, layout is controlled by `layout.preset`, and validation follows the
watermark-specific rules.

All three restricted profiles reject `list` and `page_break`. Table and
barcode profiles additionally reject `frame`, `tab`, and inline `link`. The
section profile keeps supported `frame` fields available but rejects
`frame.overflow = "paginate"` inside a header / footer / layer (the parent
section is not itself a paginating context). Multi-column frame fields are
not part of any text profile; Flow Document columns use their own group layout.

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
  "layout": { "left": 18, "top": 34 },
  "format": "qrcode",
  "content": "https://gpdf.com/docs/api-reference/#47-barcode",
  "width": 28,
  "height": 28,
  "style": {
    "color": "#111111",
    "fill": { "color": "#FFFFFF" }
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
  "layout": { "left": 18, "top": 70 },
  "format": "code128",
  "content": "INV-2026-001",
  "width": 60,
  "height": 16,
  "style": {
    "color": "#000000",
    "fill": { "color": "#FFFFFF" }
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

- `format`, `content`, `width`, and `height`.
- Outside a linear Container, one of `layout.left`, `layout.right`, or
  `layout.anchor`.
- Coordinate placement requires `layout.top` or `layout.bottom`; effective Body
  flow and linear Container children omit both.

Optional: `style`, `options`, `barcode_text`, `rotation`, `layout.z_index`,
`comment`, `link`.

Rules:

- For stable output, use only `0`, `90`, `180`, or `270`. The current PDF renderer treats other integer angles as `0`.
- `barcode_text` inherits the same rotation.
- `format` is case-insensitive. `-` and `_` are equivalent separators.
- `content` must contain at least one non-whitespace character. Empty or whitespace-only content returns `API-002`, the same rule as table-cell barcodes and GFL.
- `maxicode` uses a 30-column, 33-row logical module grid. Shared geometry supplies native hexagonal cells, finder rings and one X of quiet zone, uniformly fitted within the requested box. PDF rendering draws the native paths; preview derives rectangle geometry from the same logical grid and keeps the existing preview JSON schema. `module_shape` does not restyle MaxiCode hexagons.
- Resource limits: `content` holds at most `4096` UTF-8 bytes, and individual formats may accept less. `format`, `options.module_shape`, `options.datamatrix_shape` and `options.bearer_bar.shape` hold at most `64` bytes. When the format draws bearer bars (Code 128, ITF and other linear formats with a recognized `bearer_bar.shape`), `options.bearer_bar.thickness` is a finite number from `0` to `64` modules (default `2`; fractional values are truncated); ignored bearer settings are not checked. The encoded symbol holds at most `1048576` modules, or `65536` when `options.module_shape` is exactly `circle`; MaxiCode keeps its native geometry and is exempt from the circle limit. Violations return `API-002`.

Supported `format` values (current build):

- 2D / matrix: `qrcode` (`qr`), `microqr` (`micro-qr`), `pdf417`, `micropdf417`, `datamatrix` (`data-matrix`), `gs1datamatrix`, `aztec`, `maxicode`, `gs1qrcode`
- 1D / linear: `code128`, `code128a`, `code128b`, `code128c`, `gs1128`, `code39`, `code93`, `codabar`, `ean8`, `ean13`, `upca`, `upce`, `itf` (`interleaved2of5`), `itf14`, `gtin8`, `gtin12`, `gtin13`, `gtin14`, `isbn`, `sscc`
- Other: `msi`, `msi10`, `msi11`, `msi1010`, `msi1110`, `upus10`, `uspsimb`, `upcacomposite`, `upcecomposite`

### 4.8 Image

```json
{
  "type": "image",
  "layout": { "left": 4, "top": 8 },
  "width": 33,
  "height": 11,
  "asset": "aitrack-1",
  "format": "jpg"
}
```

Required fields:

- `width` and `height`, plus exactly one image source.
- Outside a linear Container, one of `layout.left`, `layout.right`, or
  `layout.anchor`.
- Coordinate placement requires `layout.top` or `layout.bottom`; effective Body
  flow and linear Container children omit both.

Optional: `rotation`, `layout.z_index`, `comment`, `link`.

Image source — exactly one of:

- **Shorthand**: top-level `asset` (with optional top-level `format`).
- **Explicit**: top-level `source` object.

```json
{ "type": "image", "layout": { "left": 4, "top": 8 }, "width": 33, "height": 11,
  "source": { "kind": "asset", "key": "aitrack-1", "format": "jpg" } }
```

```json
{ "type": "image", "layout": { "left": 4, "top": 8 }, "width": 33, "height": 11,
  "source": { "kind": "base64", "format": "jpg",
              "payload": "/9j/4AAQSkZJRgABAQAASABIAAD/4QBARXhpZgAATU0AKgAAAAgAAYdpAAQAAAABAAAAGgAAAAAAAqACAAQAAAABAAABSqADAAQAAAABAAAAbgAAAAD/7QA4UGhvdG9zaG9wIDMuMAA4QklNBAQAAAAAAAA4QklNBCUAAAAAABDUHYzZjwCyBOmACZjs+EJ+/+IH2ElDQ19QUk9GSUxFAAEBAAAHyGFwcGwCIAAAbW50clJHQiBYWVogB9kAAgAZAAsAGgALYWNzcEFQUEwAAAAAYXBwbAAAAAAAAAAAAAAAAAAAAAAAAPbWAAEAAAAA0y1hcHBsAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALZGVzYwAAAQgAAABvZHNjbQAAAXgAAAWKY3BydAAABwQAAAA4d3RwdAAABzwAAAAUclhZWgAAB1AAAAAUZ1hZWgAAB2QAAAAUYlhZWgAAB3gAAAAUclRSQwAAB4wAAAAOY2hhZAAAB5wAAAAsYlRSQwAAB4wAAAAOZ1RSQwAAB4wAAAAOZGVzYwAAAAAAAAAUR2VuZXJpYyBSR0IgUHJvZmlsZQAAAAAAAAAAAAAAFEdlbmVyaWMgUkdCIFByb2ZpbGUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAG1sdWMAAAAAAAAAHwAAAAxza1NLAAAAKAAAAYRkYURLAAAAJAAAAaxjYUVTAAAAJAAAAdB2aVZOAAAAJAAAAfRwdEJSAAAAJgAAAhh1a1VBAAAAKgAAAj5mckZVAAAAKAAAAmhodUhVAAAAKAAAApB6aFRXAAAAEgAAArhrb0tSAAAAFgAAAspuYk5PAAAAJgAAAuBjc0NaAAAAIgAAAwZoZUlMAAAAHgAAAyhyb1JPAAAAJAAAA0ZkZURFAAAALAAAA2ppdElUAAAAKAAAA5ZzdlNFAAAAJgAAAuB6aENOAAAAEgAAA75qYUpQAAAAGgAAA9BlbEdSAAAAIgAAA+pwdFBPAAAAJgAABAxubE5MAAAAKAAABDJlc0VTAAAAJgAABAx0aFRIAAAAJAAABFp0clRSAAAAIgAABH5maUZJAAAAKAAABKBockhSAAAAKAAABMhwbFBMAAAALAAABPBydVJVAAAAIgAABRxlblVTAAAAJgAABT5hckVHAAAAJgAABWQAVgFhAGUAbwBiAGUAYwBuAP0AIABSAEcAQgAgAHAAcgBvAGYAaQBsAEcAZQBuAGUAcgBlAGwAIABSAEcAQgAtAHAAcgBvAGYAaQBsAFAAZQByAGYAaQBsACAAUgBHAEIAIABnAGUAbgDoAHIAaQBjAEMepQB1ACAAaADsAG4AaAAgAFIARwBCACAAQwBoAHUAbgBnAFAAZQByAGYAaQBsACAAUgBHAEIAIABHAGUAbgDpAHIAaQBjAG8EFwQwBDMEMAQ7BEwEPQQ4BDkAIAQ/BEAEPgREBDAEOQQ7ACAAUgBHAEIAUAByAG8AZgBpAGwAIABnAOkAbgDpAHIAaQBxAHUAZQAgAFIAVgBCAMEAbAB0AGEAbADhAG4AbwBzACAAUgBHAEIAIABwAHIAbwBmAGkAbJAadSgAUgBHAEKCcl9pY8+P8Md8vBgAIABSAEcAQgAg1QS4XNMMx3wARwBlAG4AZQByAGkAcwBrACAAUgBHAEIALQBwAHIAbwBmAGkAbABPAGIAZQBjAG4A/QAgAFIARwBCACAAcAByAG8AZgBpAGwF5AXoBdUF5AXZBdwAIABSAEcAQgAgBdsF3AXcBdkAUAByAG8AZgBpAGwAIABSAEcAQgAgAGcAZQBuAGUAcgBpAGMAQQBsAGwAZwBlAG0AZQBpAG4AZQBzACAAUgBHAEIALQBQAHIAbwBmAGkAbABQAHIAbwBmAGkAbABvACAAUgBHAEIAIABnAGUAbgBlAHIAaQBjAG9mbpAaAFIARwBCY8+P8GWHTvZOAIIsACAAUgBHAEIAIDDXMO0w1TChMKQw6wOTA7UDvQO5A7oDzAAgA8ADwQO/A8YDrwO7ACAAUgBHAEIAUABlAHIAZgBpAGwAIABSAEcAQgAgAGcAZQBuAOkAcgBpAGMAbwBBAGwAZwBlAG0AZQBlAG4AIABSAEcAQgAtAHAAcgBvAGYAaQBlAGwOQg4bDiMORA4fDiUOTAAgAFIARwBCACAOFw4xDkgOJw5EDhsARwBlAG4AZQBsACAAUgBHAEIAIABQAHIAbwBmAGkAbABpAFkAbABlAGkAbgBlAG4AIABSAEcAQgAtAHAAcgBvAGYAaQBpAGwAaQBHAGUAbgBlAHIAaQENAGsAaQAgAFIARwBCACAAcAByAG8AZgBpAGwAVQBuAGkAdwBlAHIAcwBhAGwAbgB5ACAAcAByAG8AZgBpAGwAIABSAEcAQgQeBDEESQQ4BDkAIAQ/BEAEPgREBDgEOwRMACAAUgBHAEIARwBlAG4AZQByAGkAYwAgAFIARwBCACAAUAByAG8AZgBpAGwAZQZFBkQGQQAgBioGOQYxBkoGQQAgAFIARwBCACAGJwZEBjkGJwZFAAB0ZXh0AAAAAENvcHlyaWdodCAyMDA3IEFwcGxlIEluYy4sIGFsbCByaWdodHMgcmVzZXJ2ZWQuAFhZWiAAAAAAAADzUgABAAAAARbPWFlaIAAAAAAAAHRNAAA97gAAA9BYWVogAAAAAAAAWnUAAKxzAAAXNFhZWiAAAAAAAAAoGgAAFZ8AALg2Y3VydgAAAAAAAAABAc0AAHNmMzIAAAAAAAEMQgAABd7///MmAAAHkgAA/ZH///ui///9owAAA9wAAMBs/8AAEQgAbgFKAwEiAAIRAQMRAf/EAB8AAAEFAQEBAQEBAAAAAAAAAAABAgMEBQYHCAkKC//EALUQAAIBAwMCBAMFBQQEAAABfQECAwAEEQUSITFBBhNRYQcicRQygZGhCCNCscEVUtHwJDNicoIJChYXGBkaJSYnKCkqNDU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6g4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2drh4uPk5ebn6Onq8fLz9PX29/j5+v/EAB8BAAMBAQEBAQEBAQEAAAAAAAABAgMEBQYHCAkKC//EALURAAIBAgQEAwQHBQQEAAECdwABAgMRBAUhMQYSQVEHYXETIjKBCBRCkaGxwQkjM1LwFWJy0QoWJDThJfEXGBkaJicoKSo1Njc4OTpDREVGR0hJSlNUVVZXWFlaY2RlZmdoaWpzdHV2d3h5eoKDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uLj5OXm5+jp6vLz9PX29/j5+v/bAEMAAQEBAQEBAgEBAgMCAgIDBAMDAwMEBgQEBAQEBgcGBgYGBgYHBwcHBwcHBwgICAgICAkJCQkJCwsLCwsLCwsLC//bAEMBAgICAwMDBQMDBQsIBggLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLCwsLC//dAAQAFf/aAAwDAQACEQMRAD8A/vwooooAKKKKACiiigAooooAKKKKACiiigAooooAKKKKACiiigAooooAK+OP2ov2+v2WP2PLQn43+I3t79o/Mi0zT7S41K/kB6Ygto5GGexfavvX2PQMA5A61UbX95aCd7aH8lH7Qv8AwdFT6PJcaV+y3+z/AOK9cYZWLUPEcMthCTzg/Z4I5pGHTgyIfpX40fGT/g4P/wCC1HxPmnh8JacPAlpNkLFonhuSSRFPT97eJcNkeox9K/0b9xo3HrXZDE0Y7Ul83c55Uaj3n+B/lq+F/GP/AAXU/b68RyeGPDOrfFLxdJuxMiXF3YWMO/8A56NmC3jX2Ygegr9APhf/AMGt/wDwUi+MZi1j4/8Ai/QPCSv8zpf30+s3qg/7MQMWfUef+Nf6FRYmkrSWZz2pxSJWDjvNtn8dvw4/4NA/gbp0cc3xY+MWualL8pkj0jTrexTPcBpWuGI9DgV93fDr/g1+/wCCXPgTUrXWNTtPFHiO4tJY50/tLWCqb4mDDKW0cIIJHIOQRxX9ElFc8sbXlvM0WGpLaJ8iftO/sF/siftm3Wg3n7T3giz8YHwwJ10xLySZY7cXOzzMJFIitu8tfvA9OO9ea+Fv+CT/APwTS8F3CXXh74F+DI5ExhpdJgnPHr5qvn8a/QWisVVmlZSdvU1cIt3aOH074ZfDbR/CNl8P9J8PaZa6DpoVbTTYrSJLS3CZ2iOELsQDJxtUYzXXWVjY6bbraabBHbxL91IlCKPoBgVaoqLsoKKKKQBRRRQAUUUUAFFFFABRRRQAUUUUAf/Z" } }
```

Rules:

- `asset` and `source` are mutually exclusive.
- `source.kind` accepts `asset` and `base64`.
- The `payload` for `base64` is the raw base64 content **without** a `data:image/...;base64,` prefix. Data URIs are rejected.
- Supported formats: `jpg`, `jpeg`, `png`, `webp`, `svg`.
- `rotation` accepts any integer angle (e.g. `45`, `-30`).
- An asset key names an image that the operator stores for the deployment. A key with an extension names one stored image; a key without one is looked up as the bare key and with each supported extension, the `format`'s extension first. An asset follows the renderer's per-image byte limit, as an inline image does ([§6.2](#62-limits)). An asset that is not stored returns `API-506`, one whose stored candidates all exceed the byte limit returns `API-007`, and a storage failure returns `API-504`; see [§6.1](#61-error-codes).
- PNG and WebP images are limited by the dimensions in their headers before decoding: at most `8192` pixels per side, `4194304` pixels, and `16777216` decoded bytes (16-bit PNG counts two bytes per sample). Every WebP frame is checked, not only the canvas. Larger images, and WebP files whose chunk or frame header extends past the image data, return `API-007`. JPEG is embedded without decoding and has no pixel limit.
- SVG images load only embedded `data:` images, which follow the PNG/WebP/GIF/JPEG dimension limits (a GIF frame must also lie within its logical screen); other references are ignored and never read from a file system. Compressed SVG (`svgz`) is rejected. Filters are rasterized at a lower resolution when needed and are rejected beyond `1048576` pixels per filter region, `64` primitives, `numOctaves` `8` or `1024` convolution kernel taps. These limits also cover everything rendered inside a filtered group, including nested SVG images, patterns, masks, clip paths and raster image decodes, and the filtered groups of one SVG share its work limit in proportion to their demand. Work that does not shrink with the resolution, such as raster images rendered repeatedly inside a filtered group, is rejected when it exceeds the `16777216`-unit work limit at every resolution. These failures return `API-002`.
- See [§6.2](#62-limits) for image and total request body size limits.

### 4.9 Shapes, links, containers, and charts

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
when omitted (see §4.16).

Line endpoints `x1/y1/x2/y2` are ordered geometric coordinates. Their order
defines the line direction and start/end markers; layout may translate the
whole line while retaining those endpoints. They are not padding sides and
remain unchanged by the shared `Insets` contract.

#### 4.9.2 Rect

```json
{
  "type": "rect",
  "layout": {
    "anchor": { "reference": "content_right", "offset": 6 },
    "top": 20
  },
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

`points[].x` and `points[].y` are geometry-only point coordinates for the
polygon shape. They are not element placement fields; positioned elements use
`layout.left`/`layout.top`, `layout.right`/`layout.bottom`, or `layout.anchor`.

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

#### 4.9.6 Path

`path` is a controlled native vector outline element. Callers send SVG path
data in `d`; the renderer does not accept SVG DOM, `<path>` XML, CSS style,
transform, or external resources.

```json
{
  "type": "path",
  "layout": {
    "anchor": { "reference": "content_right", "offset": 10 },
    "top": 210
  },
  "width": 50,
  "height": 18,
  "view_box": { "x": 0, "y": 0, "width": 500, "height": 180 },
  "d": "M 12 90 C 80 20 160 170 240 80 Q 360 10 488 92",
  "stroke": {
    "color": "#111111",
    "width": 0.6,
    "cap": "round",
    "join": "round"
  }
}
```

Rules:

- Outside a linear Container, use one of `layout.left`, `layout.right`, or
  `layout.anchor`; a linear parent supplies the Path slot. `width` and `height`
  are page-space millimetres.
- `view_box` defines the local coordinate system for `d`; its geometry fields remain `x`, `y`, `width`, and `height`, and its `width` and `height` must be positive. These are geometry-only fields, not element placement fields.
- `d` supports SVG path data letters `M/L/H/V/Q/C/S/T/A/Z` and lowercase relative variants.
- `d` must start with `M/m`, expand to at most 1024 normalized segments, and contain at least one drawing segment.
- Coordinates and control points must stay within `view_box`; enlarge `view_box` when control points need to extend farther.
- `stroke.width` is in page-space millimetres and is not scaled by `view_box`.
- `stroke.compound` is not supported.
- Attached `link` regions use the resolved `left/top/width/height` bounding box.

#### 4.9.7 Link spec and standalone link

A `LinkSpec` is reused everywhere a hyperlink is allowed:

```json
{
  "target": { "type": "url", "url": "https://gpdf.com/docs/api-reference/#497-link-spec-and-standalone-link" },
  "alt": "Open the official site",
  "padding": {
    "space": "physical",
    "top": {
      "value": 1.0,
      "unit": "mm"
    },
    "right": {
      "value": 1.0,
      "unit": "mm"
    },
    "bottom": {
      "value": 1.0,
      "unit": "mm"
    },
    "left": {
      "value": 1.0,
      "unit": "mm"
    }
  },
  "border": { "color": "#1A202C", "width": 0.3 }
}
```

`LinkTarget` variants:

- URL: `{ "type": "url", "url": "https://..." }`
- In-document jump: `{ "type": "page", "page": 2, "left": 10, "top": 20 }`

Rules:

- URL schemes allowed: `http://`, `https://`, `mailto:`, `tel:`. Others return `API-002`.
- URL strings are trimmed before being written to the PDF annotation.
- `page` is 1-indexed and must not exceed the request's page count.
- `padding` uses physical `Insets` in `mm` or `pt` and expands each side of the click region independently; omitted sides default to zero.
- `border.width` must be finite and `>= 0`.
- `border.color` must be a `#RRGGBB` colour (see [§4.4.3](#443-colour-values)) if provided.

Standalone link element:

```json
{
  "type": "link",
  "layout": {
    "anchor": { "reference": "content_left", "offset": 10 },
    "top": 10
  },
  "width": 40,
  "height": 8,
  "target": { "type": "url", "url": "https://gpdf.com/docs/api-reference/#497-link-spec-and-standalone-link" },
  "alt": "Open website",
  "border": { "color": "#2563EB", "width": 0.4 }
}
```

Use the standalone form for clickable region overlays not bound to a specific
element.

#### 4.9.8 Container

`container` is a first-class geometric grouping element. Use it when a visual
group should move, edit, inherit descendant defaults, clip, or paginate as one
unit. It is designed for cards, address blocks, status badges, totals blocks,
bordered groups, clipped panels, and nested groups.

It is not an HTML/CSS container and does not accept DOM, flexbox,
percentage sizing, auto margins, grow/shrink fields, `children`, `background`,
`border`, `radius`, `row`, `column`, `rotation`, or `y_anchor` aliases.
Container padding and overflow are accepted only at `layout.children.padding`
and `layout.children.overflow`. Top-level `padding` or `overflow` is rejected,
including null values and requests that also provide the canonical location.

Top-level fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `type` | `"container"` | **Yes** | |
| `layout.top` | `number` | Conditional | Outer-box top placement for non-flow coordinate placement. Forbidden while the container effectively participates in body flow. |
| `layout.bottom` | `number` | Fixed height only | Outer-box bottom placement. Requires `height`. |
| `layout.left`, `layout.right`, or `layout.anchor` | | Scope-dependent | Choose one for direct Body or coordinate-parent placement. All three are forbidden when the direct parent Container uses linear layout. |
| `width` | `number` | **Yes** | Finite positive outer-box width. |
| `height` | `number` | No | Fixed finite positive outer-box height. Omit it to auto-size from descendants. |
| `min_height` | `number` | No | Auto-height lower bound in mm. Valid only when `height` is omitted; it is not added to the measured height. |
| `fill` | `FillStyle` | No | Draws the container background only when declared. |
| `stroke` | `StrokeStyle` | No | Draws the container outline only when declared. |
| `corner_radius` | `number` | No | Single radius for all corners. |
| `layout.children.padding` | `Insets` | No | Physical sides with explicit `mm` or `pt` lengths; child coordinates start in the content box. |
| `layout.children.overflow` | string | No | `visible`, `clip`, or `paginate`. Default `visible`. |
| `layout.children.mode` | string | **When layout is supplied** | `coordinate` or `linear`. A Flow block Container may omit the entire `layout`, selecting default coordinate children; every other Container declares its child-layout owner explicitly. |
| `layout.children.axis/gap/main_align/cross_align/wrap` | | No | Linear-layout controls; valid only when `mode` is `linear`. |
| `defaults` | `Defaults` | No | Applies only to descendants, not to the container's own fill/stroke. |
| `elements` | array | **Yes** | Descendant elements. May be empty. |

Allowed descendants:

- `text`, `barcode`, `image`, `chart`, `line`, `rect`, `circle`, `ellipse`,
  `polygon`, `path`, `link`, and `container`.
- `table` is not a valid container child.

Height:

- If `height` is non-null, the container uses that fixed outer-box height and must omit `min_height`, including null.
- If `height` is omitted or null, the container measures descendant bounds plus
  padding and uses that value as its outer-box height.
- `min_height` clamps an auto-height container to at least that many
  millimetres. It is a minimum, not `measured_height + min_height`.
- Auto-height containers cannot use `layout.bottom`, direct child
  `layout.bottom`, `layout.children.overflow: "paginate"`, vertical linear
  `wrap`, or vertical linear `main_align` values other than `start`.
- To shrink text inside a fixed-width, fixed-height area, keep the container
  fixed and put the constraint on the child text with `frame.width`,
  `frame.height`, `frame.shrink_to_fit`, and `frame.min_font_size`.

Layout:

- Coordinate layout keeps each child at its local `layout.left/layout.top`,
  `layout.right/layout.bottom`, or `layout.anchor/layout.top`.
- Coordinate child positions are relative to the container content box. Page
  anchors are forbidden inside every container descendant.
- Linear layout uses `layout.children.mode: "linear"` plus optional
  `layout.children.axis`, `gap`, `main_align`, `cross_align`, and `wrap`.
- Linear layout is the sole placement owner for its direct children. Those
  children must omit `layout.left`, `layout.top`, `layout.right`,
  `layout.bottom`, and `layout.anchor`.
- Container children never inherit document `flow` or `gap_after`. A nested
  container first occupies a slot assigned by its direct parent, then applies
  its own `layout.children` rules to its direct descendants.
- Linear children are measured from their own rendered bounds. A text child
  without `style.width`, `frame.width`, or an explicit box width is not
  automatically stretched to the container width; set child widths when
  wrapping, alignment, or `cross_align: "stretch"` must be predictable.
- `axis`: `horizontal | vertical`; default `vertical`.
- `main_align`: `start | center | end | space_between`; default `start`.
- `cross_align`: `start | center | end | stretch`; default `start`.
- `wrap`: boolean; default `false`.
- With `axis = "horizontal"`, `wrap: true` starts a new row when the next child
  would exceed the content-box width.
- With `axis = "vertical"`, `wrap: true` starts a new column when the next child
  would exceed the content-box height.
- Each wrapped row or column uses the largest cross-axis size among the children
  in that row/column. If content still exceeds the container, the container's
  `layout.children.overflow` behaviour applies.

Overflow:

- `visible` draws children normally.
- `clip` clips descendants to the container outer box.
- `paginate` creates continuation fragments across pages. It is allowed only
  on a fixed-height Container whose child layout is vertical linear with `wrap: false`.
  The Container must be a direct `pages[].elements` item with effective body
  flow `true`, or the element of a Flow Document `block` (whose parent already
  owns flow; do not add `element.layout.flow`). Flow block fragments may
  continue into the next body column or page. ContainerItem descendants,
  coordinate/non-flow body items, placed content, header/footer, and layers
  cannot paginate. Descendant text splits only when that text item explicitly uses
  `frame.overflow: "paginate"`; otherwise children move whole by item boundary.

Link rule: if the container itself has `link`, descendants must not declare
element-level links and must not include standalone `type: "link"` items.

Paginated container with splittable text:

```json
{
  "type": "container",
  "layout": {
    "left": 10,
    "flow": true,
    "children": {
      "mode": "linear",
      "axis": "vertical",
      "wrap": false,
      "overflow": "paginate",
      "padding": {
        "space": "physical",
        "top": {
          "value": 2,
          "unit": "mm"
        },
        "right": {
          "value": 2,
          "unit": "mm"
        },
        "bottom": {
          "value": 2,
          "unit": "mm"
        },
        "left": {
          "value": 2,
          "unit": "mm"
        }
      }
    }
  },
  "width": 80,
  "height": 100,
  "elements": [
    {
      "type": "text",
      "frame": { "width": 76, "overflow": "paginate" },
      "content": {
        "blocks": [
          {
            "type": "paragraph",
            "inlines": [
              { "type": "text", "text": "Long text that may flow across pages." }
            ]
          }
        ]
      }
    }
  ]
}
```

```json
{
  "type": "container",
  "layout": {
    "left": 20,
    "top": 24,
    "children": {
      "mode": "coordinate",
      "padding": {
        "space": "physical",
        "top": {
          "value": 5,
          "unit": "mm"
        },
        "right": {
          "value": 6,
          "unit": "mm"
        },
        "bottom": {
          "value": 5,
          "unit": "mm"
        },
        "left": {
          "value": 6,
          "unit": "mm"
        }
      }
    }
  },
  "width": 84,
  "height": 34,
  "fill": { "color": "#F8FAFC", "opacity": 1 },
  "stroke": { "color": "#CBD5E1", "width": 0.3 },
  "corner_radius": 3,
  "elements": [
    {
      "type": "text",
      "layout": { "left": 0, "top": 0 },
      "style": { "width": 72, "font_size": 8, "color": "#64748B" },
      "content": "Payment status"
    },
    {
      "type": "text",
      "layout": { "left": 0, "top": 10 },
      "style": { "width": 72, "font_size": 15, "font_weight": "bold", "color": "#0F172A" },
      "content": "Paid"
    }
  ]
}
```

#### 4.9.9 Chart

`chart` is a fixed-size vector element for static PDF data visualisation. It
supports ten public kinds: `bar`, `line`, `area`, `combo`, `pie`, `donut`,
`waterfall`, `scatter`, `range`, and `heatmap`. Category-shaped Cartesian
charts use one category axis and one or two value axes; Scatter uses continuous
X and Y axes; Heatmap uses category and row axes. `combo` and dual value axes
remain part of the original v0.1 contract.

Use Chart only when the target runtime's `/health` advertises `layout.chart`.
Build profiles can omit Chart; source documentation and OpenAPI are not proof
that a deployed target supports it.

Smallest useful chart:

```json
{
  "type": "chart",
  "kind": "area",
  "layout": { "left": 15, "top": 35 },
  "width": 180,
  "height": 90,
  "data": {
    "labels": ["Jan", "Feb", "Mar", "Apr"],
    "series": [
      { "name": "Revenue", "values": [31, 40, 28, 51] },
      { "name": "Cost", "values": [18, 24, 20, 32] }
    ]
  },
  "alt": "Revenue remains above cost and ends at its highest value."
}
```

The omitted theme resolves to `dashboard_light`. Chart strings are single-line
plain strings: they reject form feed, CR, LF, NEL, U+2028, and U+2029, and do
not execute Markdown, page variables, JavaScript callbacks, or locale-dependent
formatting.

The complete render-tested small-data cookbook is in
[`doc/examples/chart-v0.1/`](../../examples/chart-v0.1/README.md). It includes
all ten native kinds, all four presets, Spark/Progress/Bullet compositions,
Combo and same-kind dual axes, both stack modes, all three Projection shapes,
reference annotations, explicit gradients and patterns, monochrome output,
and Donut centre text composed with a coordinate Container.

Top-level fields:

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `type` | `"chart"` | **Yes** | Element discriminator. |
| `kind` | enum | **Yes** | `bar`, `line`, `area`, `combo`, `pie`, `donut`, `waterfall`, `scatter`, `range`, or `heatmap`. |
| `layout` | `ElementLayout` | Scope-dependent | Uses canonical `layout.left/top/right/bottom/anchor/flow/gap_after/z_index`. Legacy top-level placement fields are rejected. |
| `width`, `height` | number | **Yes** | Final frame size in mm; each must be finite and `> 0`. |
| `padding` | `Insets` | No | Physical sides with explicit `mm` or `pt` lengths that must leave a positive content area. |
| `data` | `ChartData` or Projection reference | **Yes** | Kind-specific inline shape or `{ "projection": "name" }`. |
| `options` | `ChartOptions` | No | Kind-specific geometry and curve options. |
| `theme` | `ChartTheme` | No | Preset plus palette/background/text overrides. |
| `axes` | `ChartAxes` | Non-radial | Category/value axes, Scatter X/value axes, or Heatmap category/row axes. |
| `grid` | `ChartGrid` | Non-radial | Kind-specific category/value or X/Y grid directions. |
| `legend` | `ChartLegend` | No | Visibility, side, alignment, and text style. |
| `data_labels` | `ChartDataLabels` | No | Visibility, content, placement, overflow policy, and format. |
| `annotations` | `ChartAnnotations` | Bar/Line/Area/Combo/Waterfall/Scatter/Range | Up to 32 reference lines and bands in total. |
| `alt` | string | PDF/UA: **Yes** | Describe the trend, comparison, or composition. |
| `link` | `LinkSpec` | No | One link over the complete chart frame. Point-level links are not supported. |
| `comment` | string | No | Retained but not drawn. |

`width` and `height` remain authoritative in a Container. The chart is not
stretched to its parent slot, split across pages, or rotated.

##### Data and series

Most kinds use `data.labels` as a shared category or Heatmap-column sequence.
Scatter omits labels and uses `series[].points`. Range adds aligned lower and
upper arrays; Waterfall adds optional point semantics. Category-shaped values
may be finite numbers or `null`; radial values must be finite, non-negative,
non-null, and have a total greater than zero.

| Data field | Type | Rules |
|------------|------|-------|
| `labels` | string[] | Non-empty except Scatter, which omits it or uses `[]`. Pie/Donut and Heatmap labels are non-empty and unique after trimming. |
| `series` | `ChartSeries[]` | Non-empty. Pie/Donut and Waterfall require exactly one series. |

| Series field | Applies to | Rules |
|--------------|------------|-------|
| `name` | All | Trimmed value must be non-empty and unique within the chart. |
| `values` | Category/radial/Heatmap | Array length equals `labels.length`; optional centre values on Range. |
| `points` | Scatter | Non-empty `{ x, y?, size? }[]`; x is required and finite, y may be omitted or null for a missing value, and optional size is positive. |
| `lower_values`, `upper_values` | Range | Required aligned bounds; each finite lower value is no greater than its upper value. |
| `point_kinds` | Waterfall | Optional aligned `delta/subtotal/total` sequence; subtotal values are null. |
| `kind` | Cartesian | `bar`, `line`, or `area`; required for every Combo series. Other Cartesian kinds use the resolved-kind mapping below when the field is supplied. |
| `value_axis` | Cartesian | `primary` (default) or `secondary`. |
| `options` | Cartesian | Series-kind option overrides. |
| `stack` | Bar/Area | `{ "group": string, "mode": "normal" | "percent" }`. |
| `format` | Cartesian | Deterministic number formatting for labels. |
| `color` | All | `#RRGGBB` colour. On Pie/Donut it intentionally makes all slices the same colour. |
| `colors` | Pie/Donut, Waterfall, Heatmap | Slice colours; 3-4 Waterfall semantic colours; or 2-9 global Heatmap scale colours. |
| `stroke` | All | Chart stroke; `compound` is not supported. |
| `fill` | Bar/Area (including Combo), Waterfall/Range/Scatter/Pie/Donut | Solid, linear-gradient, or pattern fill; ordinary Line and Heatmap reject it. |
| `marker` | Line/Area/Scatter/Range | `show`, `shape`, and positive `size` in mm. |

Pie and Donut require exactly one series and unique, non-empty labels after
trimming. They reject Cartesian-only series fields, `axes`, and `grid`.

`marker.shape` accepts `circle`, `square`, `diamond`, or `triangle`.
The resolved series kind is `bar` for Bar and Waterfall, `line` for Line and Scatter, and `area` for Area and Range. Combo declares the kind on each series. Waterfall and Range still reject `stack`; a mapping does not enable every option of the corresponding base kind.

Series-level `options` depend on the resolved series kind:

| Series kind | Supported `series[].options` |
|-------------|--------------------------------|
| Bar / Waterfall | `corner_radius` (finite mm value `>= 0`) |
| Line / Scatter | `curve`, `connect_nulls` |
| Area / Range | `curve`, `connect_nulls`, `area_fill` |

Scatter series additionally accept `show_line`; the default is marker-only.
When a Scatter line uses `curve: "monotone"`, X values must be strictly
increasing within each connected run.

##### Waterfall, Scatter/Bubble, Range, and Heatmap data

Waterfall uses exactly one series. `point_kinds` defaults to `delta`; `delta`
and `total` require finite values, while `subtotal` requires `null` and is
computed from the current cumulative value. `options.show_connectors` controls
the bridge lines.

Scatter uses no category labels. Each series supplies `points` with continuous
`x`, nullable `y`, and optional positive `size`. Any size value enables Bubble
encoding. `bubble_min_size` and `bubble_max_size` bound the rendered marker
diameter and are valid only when at least one point supplies size. The X axis is
`axes.x`; Y uses `axes.values.primary/secondary`.

Range requires aligned `lower_values` and `upper_values`. Optional `values`
draw the centre line and must stay inside the finite bounds. Null bounds occur
as a pair, and the centre is also null at that position. At least one finite
range is required.

Heatmap treats labels as columns and each named series as one row. Every row
has the same value count and the same optional format/heat-scale colours.
`options.cell_gap` is a non-negative millimetre gap. It uses
`axes.category`, `axes.row`, and X/Y grid directions; value axes and reference
annotations are not supported.

Spark Line/Area, Progress Circle, and Bullet are compositions rather than new
kind values. Spark hides axes/grid/legend/labels, Progress Circle combines a
Donut with centre Text in a coordinate Container, and Bullet combines a
horizontal Bar with reference bands and a target line.

`NumberFormat` is available on a Cartesian series, a value axis, and data
labels. A more specific declared field overrides the corresponding axis or
series field; omitted fields continue to inherit.

| Number-format field | Type / rules |
|---------------------|--------------|
| `notation` | `standard` or `compact`. |
| `decimals` | Integer from `0` through `6`. |
| `grouping` | Boolean; `true` inserts deterministic comma-separated groups of three digits. |
| `prefix`, `suffix` | Plain strings placed before or after the formatted number. |

##### Combo and dual value axes

Use `series[].kind` to mix Bar, Line, and Area. Use
`series[].value_axis` for the scale. This same axis binding also enables a
same-kind dual-axis Bar, Line, or Area chart.

```json
{
  "type": "chart",
  "kind": "combo",
  "layout": { "left": 15, "top": 35 },
  "width": 180,
  "height": 92,
  "theme": { "preset": "business_light" },
  "data": {
    "labels": ["Jan", "Feb", "Mar"],
    "series": [
      {
        "name": "Revenue",
        "kind": "bar",
        "value_axis": "primary",
        "values": [42000, 58000, 51000]
      },
      {
        "name": "Margin",
        "kind": "line",
        "value_axis": "secondary",
        "values": [26.2, 24.1, 25.5],
        "marker": { "show": true, "shape": "circle", "size": 1.6 },
        "options": { "curve": "monotone", "connect_nulls": false }
      }
    ]
  },
  "axes": {
    "values": {
      "primary": {
        "title": "Amount",
        "position": "left",
        "include_zero": "always",
        "format": { "notation": "compact", "prefix": "$" }
      },
      "secondary": {
        "title": "Margin",
        "position": "right",
        "include_zero": "never",
        "format": { "decimals": 1, "suffix": "%" }
      }
    },
    "zero_alignment": "off"
  },
  "grid": { "show": true, "axis": "value", "value_axis": "primary" },
  "alt": "Revenue is shown as bars and margin as a secondary-axis line."
}
```

The primary axis is implicit but must bind at least one series and at least one
finite value. A series may reference `secondary` only when
`axes.values.secondary` is declared; a declared secondary axis must also be
referenced. With two value axes, both titles must be non-empty and their
positions must differ.

Value-axis fields:

| Field | Values / rules |
|-------|----------------|
| `show` | Boolean; defaults to visible. |
| `title` | Plain string; required and non-empty on both axes when dual-axis mode is active. |
| `position` | `left/right` for vertical charts; `top/bottom` for horizontal charts. |
| `scale` | `linear` in v0.1. |
| `include_zero` | `auto`, `always`, or `never`. Bar-bound axes must retain zero. |
| `min`, `max` | Optional finite bounds; `min < max`. |
| `tick_count` | Nice-tick target from `2` through `32`; the resolved numeric interval may produce a nearby count. Explicit requests must fit without label hiding. |
| `format` | `NumberFormat`. |
| `text` | Chart-supported `TextStyle` subset. |

`axes.zero_alignment` accepts `auto`, `on`, or `off`, and is valid only with
two value axes. It can expand automatic domains to align zero positions but
does not clip data or change a percent axis from its fixed `0..100` domain.
When `tick_count` is omitted, automatic value-axis labels may be sparsified by
a deterministic collision pass while preserving endpoints and zero. When it is explicit, all
resolved nice-tick labels are mandatory; overlap or an undersized axis frame is
a layout error rather than an implicit hide or font-size reduction.

The optional `axes.category` object has these fields:

| Category-axis field | Values / rules |
|---------------------|----------------|
| `show` | Boolean. |
| `title` | Plain string. |
| `position` | `top/bottom` for vertical charts; `left/right` for horizontal charts. |
| `label_overflow` | `auto`, `ellipsis`, `rotate`, `hide`, or `error`. |
| `label_rotation` | Finite degrees from `-90` through `90`; required with `label_overflow: "rotate"`. |
| `text` | Chart-supported `TextStyle` subset. |

##### Stacking

Bar and Area series can declare a stack. Members of one group must use the
same final series kind, value axis, and mode. Normal stacking accumulates
positive and negative values independently around zero. Percent stacking
requires non-negative, non-null values and a positive total for every category;
its axis is exclusively percent-stacked and has the fixed domain `0..100`.

```json
{
  "type": "chart",
  "kind": "bar",
  "layout": { "left": 15, "top": 35 },
  "width": 180,
  "height": 88,
  "data": {
    "labels": ["North", "South", "West"],
    "series": [
      {
        "name": "Online",
        "values": [60, 45, 70],
        "stack": { "group": "channel", "mode": "percent" }
      },
      {
        "name": "Retail",
        "values": [40, 55, 30],
        "stack": { "group": "channel", "mode": "percent" }
      }
    ]
  },
  "axes": {
    "values": {
      "primary": {
        "title": "Share",
        "min": 0,
        "max": 100,
        "format": { "suffix": "%" }
      }
    }
  },
  "data_labels": {
    "show": true,
    "mode": "percent",
    "position": "inside",
    "overflow": "hide"
  }
}
```

For Pie/Donut, percent labels use `slice / total`. For Cartesian charts,
`percent` and `label_and_percent` are valid only for percent stacks and use
`current value / current category's total for the same stack group`.

##### Options, themes, and labels

Kind-specific top-level options:

| Kind | Supported `options` |
|------|---------------------|
| Bar | `orientation`, `category_gap_ratio`, `bar_gap_ratio`, `corner_radius` |
| Line | `orientation`, `curve`, `connect_nulls` |
| Area | `orientation`, `curve`, `connect_nulls`, `area_fill` |
| Combo | `orientation`, `category_gap_ratio`, `bar_gap_ratio`; series-specific Bar/Line/Area options belong in `series[].options` |
| Pie | `start_angle`, `pad_angle` |
| Donut | `start_angle`, `pad_angle`, `inner_radius` |
| Waterfall | `orientation`, `category_gap_ratio`, `corner_radius`, `show_connectors` |
| Scatter | `curve`, `connect_nulls`, `bubble_min_size`, `bubble_max_size` |
| Range | `curve`, `connect_nulls`, `area_fill` |
| Heatmap | `cell_gap` |

Option value rules:

| Option | Values / rules |
|--------|----------------|
| `orientation` | `vertical` or `horizontal`. |
| `category_gap_ratio` | Finite number in `0..1` (exclusive upper bound). |
| `bar_gap_ratio` | Finite number in `0..=1`. |
| `corner_radius` | Finite mm value `>= 0`. |
| `curve` | `linear`, `monotone`, or `step`. |
| `connect_nulls` | Boolean. |
| `area_fill` | `gradient`, `solid`, `pattern`, or `none`. |
| `start_angle` | Any finite degree value. |
| `pad_angle` | Finite degrees in `0..360` (exclusive upper bound); multiplied by `labels.length`, it must remain `< 360`. |
| `inner_radius` | Donut ratio in `0..1` (both bounds exclusive). |
| `bubble_min_size`, `bubble_max_size` | Positive marker sizes in mm; minimum must not exceed maximum. |
| `cell_gap` | Finite Heatmap cell gap in mm, `>= 0`. |
| `show_connectors` | Boolean Waterfall connector visibility. |

Chart presets:

| Preset | Default palette | Intended use |
|--------|-----------------|--------------|
| `dashboard_light` | `#3B82F6`, `#10B981`, `#8B5CF6`, `#F59E0B`, `#6B7280`, `#06B6D4`, `#EC4899`, `#84CC16`, `#D946EF` | Default light analytical style with a transparent background and subtle Area gradients. |
| `business_light` | `#E8573A`, `#2563EB`, `#0F766E`, `#A16207`, `#4B5563` | Warm business reports with a `#FAFAF7` background. |
| `executive_dark` | `#E8573A`, `#D4A853`, `#7483A3`, `#5F9B8D`, `#A3A39E` | Dark `#0E0E0F` executive summaries with high-contrast labels and visible markers. |
| `print_mono` | `#111111`, `#555555`, `#888888`, `#B5B5B5` | Monochrome output with pattern, dash, and marker redundancy. |

| Theme field | Type / rules |
|-------------|--------------|
| `preset` | One of the four preset names above; defaults to `dashboard_light`. |
| `palette` | Non-empty array of `#RRGGBB` colours, used in order and repeated as needed. |
| `background` | Chart-owned `ChartFillStyle` override. |
| `text` | Chart-supported `TextStyle` override. |

Chart text styles accept only
`font_family`, `font_size`, `font_weight`, `font_style`, `font_mode`, `color`,
`opacity`, `letter_spacing`, and `direction`; text-owned layout and wrapping
fields are rejected because the chart compiler owns label placement.
The preset default uses `NotoSans-Regular` with `font_mode: "prefer"`. If a
Chart text override declares its own `font_family` without `font_mode`, that
explicit family is strict and does not inherit the preset's `prefer` mode.

| Grid field | Type / rules |
|------------|--------------|
| `show` | Boolean. |
| `axis` | Category-shaped charts use `category/value/both`; Scatter and Heatmap use `x/y/both`. |
| `value_axis` | `primary` or `secondary`; invalid with `axis: "category"`, and `secondary` must exist. A chart draws at most one value-grid scale. |
| `stroke` | Chart-supported `StrokeStyle`; `compound` is not supported. |

| Legend field | Type / rules |
|--------------|--------------|
| `show` | Boolean. |
| `position` | `top`, `right`, `bottom`, or `left`. |
| `align` | `start`, `center`, or `end`. |
| `text` | Chart-supported `TextStyle` subset. |

| Data-label field | Type / rules |
|------------------|--------------|
| `show` | Boolean. |
| `mode` | `value`, `percent`, `label`, `label_and_value`, or `label_and_percent`. |
| `position` | `inside/outside/end` for Bar; `above/below/end` for Line and Area; `inside/outside` for Pie and Donut. |
| `overflow` | `hide` omits labels that do not fit; `error` returns a layout error. Neither silently reduces the requested font size. |
| `format` | `NumberFormat`. |

Pie and Donut charts must identify every category through the legend or direct
labels. If `legend.show` is explicitly `false`, `data_labels.show` must be
`true`, `mode` must include the category label (`label`, `label_and_value`, or
`label_and_percent`), and `overflow` must be `error`.

One Combo `data_labels` object applies to every series. Its position must be
valid for every resolved series kind; when Bar is mixed with Line or Area,
`end` is the shared supported position.

Reference annotations use `annotations.lines[]` and `annotations.bands[]`.
Each line has `value` plus optional `axis`, `label`, and `stroke`; each band has
`from`, `to`, optional `axis`, `label`, `fill`, and `stroke`, with `from < to`.
The axis defaults to `primary`; `secondary` requires a declared and bound
secondary value axis, while `x` is Scatter-only. Lines and bands share a
combined maximum of 32. Pie, Donut, and Heatmap reject annotations.

##### ChartFillStyle gradients and patterns

Chart `series[].fill` and `theme.background` use the Chart-owned
`ChartFillStyle`, which accepts one paint source: `color`, `gradient`, or
`pattern`. Declaring more than one is invalid. Other element fill fields keep
the existing `FillStyle` contract and do not gain gradient or pattern paint.

Linear gradient:

```json
{
  "gradient": {
    "kind": "linear",
    "angle": 90,
    "stops": [
      { "offset": 0, "color": "#3B82F6", "opacity": 0.22 },
      { "offset": 1, "color": "#3B82F6", "opacity": 0 }
    ]
  },
  "opacity": 1
}
```

The angle is finite and clockwise in page coordinates (`0` is left-to-right;
`90` is top-to-bottom). A gradient has 2 through 16 stops. Offsets are in
`0..=1` and non-decreasing. The optional stop `opacity` and outer
`ChartFillStyle.opacity` are in `0..=1` and each default to `1`; effective stop
opacity is their product.

Pattern:

```json
{
  "pattern": {
    "kind": "diagonal",
    "color": "#111111",
    "background": { "color": "#FFFFFF" },
    "spacing": 2,
    "stroke": { "width": 0.25 }
  }
}
```

Pattern kinds are `diagonal`, `crosshatch`, and `dots`. `spacing` and
`stroke.width` are positive mm values. `pattern.color` is the foreground, so
`pattern.stroke.color` is forbidden; compound pattern strokes are also
unsupported. Every kind accepts `stroke.width` and `stroke.opacity`.
`diagonal` and `crosshatch` additionally accept `stroke.cap` and `stroke.dash`
but reject `stroke.join` and `stroke.miter_limit`, because their tile paths are
independent subpaths. `dots` rejects `cap`, `join`, `miter_limit`, and `dash`;
fields that cannot change the rendered tile never parse as no-ops. Gradient
stops and all Chart-scoped colours use `#RRGGBB` values.

##### Capacity and PDF resource limits

Chart v0.1 applies deterministic request-shape and resource safety bounds:

| Limit | Maximum | Counting scope |
|-------|--------:|----------------|
| Labels | `512` | One Chart. |
| Series | `16` | One Chart. |
| Total points | `4096` | Values, Scatter points, or Heatmap cells in one Chart. |
| Reference annotations | `32` | Lines plus bands in one Chart. |
| Gradient stops | `16` | One linear Gradient. |
| Transparent Gradient fill primitives | `512` | One compiled `PreparedChart`. |
| Resolved text runs | `16384` | All shaped text runs in one compiled `PreparedChart`. |
| Resolved text run UTF-8 bytes | `1048576` | Run text plus resolved manifest font names in one compiled `PreparedChart`. |
| Distinct Pattern resources | `32` | Pattern cache misses in one PDF document. |

The transparent-Gradient bound counts `FillRect`, `FillPath`, and `Marker`
commands whose Gradient has at least one stop with `opacity < 1`. An outer
`ChartFillStyle.opacity < 1` without a transparent stop does not consume this
limit. A continuous Area fill is normally one `FillPath`; Bars, radial slices,
and filled markers count as their rendered primitives.

Pattern resources are keyed by the normalized Pattern definition. Reusing the
same Pattern in one document is a cache hit and does not consume another slot.
Exceeding a bound fails deterministically; the renderer does not truncate data,
drop a series, or silently replace a Gradient or Pattern with a solid fill.

Chart Gradient or Pattern paint that uses transparency is incompatible with
`settings.profile: "pdfa-1b"`. Rendering fails deterministically with `API-502`;
the renderer does not flatten transparency, remove opacity, or substitute a
solid fill. Use opaque fills or the `print_mono` preset for PDF/A-1 output.

`PreparedChart` is an internal Preview/LayoutPlan resource, not another Chart
request shape. New schema-version-1 text commands may include additive
`resolved_runs`. Each visually ordered run records `source_start` and
`source_len` as UTF-8 byte offsets within the command string, the exact source
text, resolved manifest font face (for text runs), chart-local `x`,
`baseline_y`, and `advance_width` in millimeters, `font_size` in points, and
the shaping direction. Source ranges must cover the command string exactly
once. PDF replay validates the runs against the active font catalog before
drawing; older resources without `resolved_runs` retain the legacy
raw-`TextStyle` replay path.

These values are accepted-input safety limits, not one latency promise shared
by native, Worker, WASIX, and browser WASM deployments. A compatible release
may raise a limit but must not lower it without declaring a breaking contract
change.

##### Validation and accessibility

- All Chart-owned objects reject unknown fields. Existing legacy
  `FillStyle`/`StrokeStyle` parsing compatibility is unchanged.
- Labels, series names, axis titles, number-format prefix/suffix, and `alt` are
  single-line plain strings. They reject form feed, CR, LF, NEL, U+2028, and
  U+2029.
- `NumberFormat` supports `standard` or deterministic `compact` notation,
  `0..=6` decimals, fixed three-digit grouping, and string prefix/suffix.
  Compact units are `K`, `M`, `B`, and `T`; no locale runtime is used.
- PDF/UA requires a non-empty `alt` for every Chart. A linked Chart also
  requires non-empty `link.alt`; the Link structure associates the annotation
  with the grouped Figure instead of exposing internal vector commands as
  unrelated structure nodes.
- Unknown Chart fields fail JSON deserialisation with `API-001`. Recognised but
  incompatible options fail validation with `API-002`. Layout that cannot
  satisfy an explicit `error` overflow policy returns a layout/render error.

#### 4.9.10 Named Dataset and Projection

Named Dataset/Projection is the typed data-input layer for sharing one small
set of raw values between one or more Tables and Charts. It is additive: all
existing inline `table.rows` and inline `chart.data` payloads remain valid.

Availability is capability-gated. Use this contract only when the target
runtime's `/health` response includes `data.projection`; a projected Chart also
requires `layout.chart`. Source documentation or an OpenAPI file alone is not
proof that a Test or Production runtime has been deployed. The `pdn-label`
shipping-label profile intentionally advertises neither capability and rejects
Dataset/Projection input.

The server uses a two-stage contract:

```text
SourceDocumentRequest
  -> validate typed datasets and projections
  -> materialize references before layout
  -> MaterializedDocumentRequest
  -> existing Table/Chart validation, layout, Preview, and PDF rendering
```

Datasets and projections have no geometry. The consuming Table or Chart still
owns `layout`, `width`, `height`, axes, legend, headers, grid, pagination, and
accessibility text. A Chart never reads a sibling Table or a paginated Table
fragment.

##### Root registries and names

`datasets` and `projections` are optional root objects. Their property names
are case-sensitive and must match `^[a-z][a-z0-9_]{0,63}$`. Names are unique
within each registry. Runtime validation does not trim or normalize a name.

| Root field | Value | Current Test cap |
|------------|-------|--------:|
| `datasets` | Map from registry name to `DatasetDefinition` | `32` definitions per document |
| `projections` | Map from registry name to a tagged Table or Chart Projection | `64` definitions per document |

Declaring either registry, even without a consumer reference, uses the
`data.projection` capability. Projection definitions may reference Datasets,
but they do not reference other Projections; there is no recursive graph,
join, filter, aggregate, expression, remote URL, or database lookup.

##### DatasetDefinition

```json
{
  "fields": [
    { "key": "month", "type": "string" },
    { "key": "revenue", "type": "number", "unit": "currency_usd" },
    { "key": "margin", "type": "number", "unit": "percent", "nullable": true }
  ],
  "rows": [
    { "month": "Jan", "revenue": 42000, "margin": 26.2 },
    { "month": "Feb", "revenue": 58000, "margin": null }
  ]
}
```

`fields` is non-empty and order-stable. The current Test parser accepts at
most `128` fields and `1024` rows per Dataset. Field keys must match
`^[A-Za-z_][A-Za-z0-9_]{0,63}$` and be unique within the Dataset.

| Dataset field | Type | Required | Rules |
|---------------|------|----------|-------|
| `key` | string | Yes | Declares the only row property that may carry this value. |
| `type` | enum | Yes | `string`, `number`, or `boolean`. |
| `nullable` | boolean | No | Defaults to `false`. When true, the row property may be omitted or `null`; omission materializes as `null`. |
| `unit` | string | No | Number fields only. Safe opaque ASCII token; no conversion or display behaviour. |

Every row is a flat object. It may contain only declared field keys, and each
non-null value must match the declared scalar type. Objects and arrays are not
Dataset scalar values. A non-nullable field must be present and non-null. No
string-to-number, number-to-string, Boolean, date, locale, or unit coercion is
performed.

`unit` is an opaque token with `1..=64` ASCII bytes. Its first character is an
ASCII letter; remaining characters may be ASCII letters, digits, `_`, `.`,
`:`, `/`, or `-`. Tokens such as `percent`, `currency_usd`,
`requests/second`, and `acme:weighted-score` are legal. gPdf does not maintain
a unit registry, correct spelling, convert values, or create axis titles from
the token. Projected series on one value axis must all use exactly the same
unit, or all omit it; primary and secondary axes may use different units.

Keep Dataset numbers raw. Do not reuse formatted Table strings such as
`"$42,000"`, `"26.2%"`, or `"21 ms"` as Chart values, and do not expect the
runtime to infer numbers from them. Convert legacy display strings to typed
numbers explicitly before building the Dataset; use Projection `format`,
Table headers, series names, and axis titles for visible formatting.

##### TableProjection

```json
{
  "type": "table",
  "dataset": "monthly_results",
  "columns": [
    { "field": "month" },
    {
      "field": "revenue",
      "key": "revenue_display",
      "format": { "notation": "compact", "prefix": "$" }
    }
  ]
}
```

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| `type` | `"table"` | Yes | Projection discriminator. |
| `dataset` | string | Yes | Existing Dataset registry name. |
| `columns` | `TableProjectionColumn[]` | Yes | Non-empty, stable order; at most `128` columns in the current Test runtime. |

Each column requires `field`, which references a declared Dataset field.
Optional `key` changes the output Table row key; it defaults to `field` and
must be non-empty. Output keys must be unique. Optional `format` uses the
existing deterministic `NumberFormat` and is valid only for number fields.
Formatting a number produces a Table text cell; an unformatted number remains
a numeric cell. String, Boolean, and null values retain their scalar meaning.

The consumer Table keeps its own `columns` definitions. The set of consumer
`columns[].key` values must exactly equal the Projection output-key set. Table
column order, headers, widths, cell styles, grids, and pagination remain on the
Table element.

##### ChartProjection

```json
{
  "type": "chart",
  "dataset": "monthly_results",
  "shape": "category",
  "category": { "field": "month" },
  "series": [
    {
      "field": "revenue",
      "name": "Revenue",
      "kind": "bar",
      "value_axis": "primary",
      "format": { "notation": "compact", "prefix": "$" },
      "fill": {
        "gradient": {
          "kind": "linear",
          "angle": 90,
          "stops": [
            { "offset": 0, "color": "#3B82F6", "opacity": 0.72 },
            { "offset": 1, "color": "#3B82F6", "opacity": 0.18 }
          ]
        },
        "opacity": 0.9
      }
    },
    {
      "field": "margin",
      "name": "Margin",
      "kind": "line",
      "value_axis": "secondary",
      "format": { "decimals": 1, "suffix": "%" },
      "stroke": { "color": "#F59E0B", "width": 0.7, "opacity": 0.85 },
      "marker": { "show": true, "shape": "circle", "size": 1.6 }
    }
  ]
}
```

Chart Projection supports three shapes:

| `shape` | Required coordinates | Series auxiliary fields |
|---------|----------------------|-------------------------|
| `category` (default) | `category` bound to a non-null string field | Optional paired `lower`/`upper` number fields and optional non-null string `point_kind` |
| `xy` | `x` bound to a number field | Optional positive number `size` field; the main series field supplies Y |
| `heatmap` | non-null string `x` and `y` fields | Exactly one value series; no auxiliary fields |

Each non-empty series item requires a number `field` and non-empty `name`.
Nullable number fields materialize null values; the consuming kind may reject
them. Category `point_kind` accepts only `delta`, `subtotal`, or `total`.
Heatmap builds ordered unique X/Y dimensions from first occurrence and rejects
a duplicate X/Y cell. A Chart Projection accepts at most `16` series.

A Chart Projection series owns the complete existing Chart-series presentation
contract except inline values/points/bounds: optional `kind`, `value_axis`,
kind-specific `options`, `stack`, `format`, `color`, `colors`, `stroke`,
`fill`, and `marker`. There is no second style-override DSL. Auxiliary field
references materialize to the ordinary specialized ChartSeries fields before
validation and layout.

The consuming Chart still declares its top-level `kind`. Every Combo series
declares `kind: "bar" | "line" | "area"`; same-kind Bar, Line, or Area Charts
may use `value_axis` without changing their top-level kind. Scatter, Range,
Waterfall, and Heatmap consumers require the matching Projection shape and
specialized fields. A secondary-bound series requires
`axes.values.secondary` on the consumer Chart. Existing Chart validation
remains authoritative for all kind-specific and resource rules.

##### Inline/reference unions

Table rows accept exactly one of these shapes:

```text
array<TableRow> | { "projection": "table_projection_name" }
```

Chart data accept exactly one of these shapes:

```text
{ "labels": [...], "series": [...] } | { "projection": "chart_projection_name" }
```

A Projection reference object requires `projection` and rejects every other
field. Inline and reference fields cannot be mixed. A Table may reference only
`type: "table"`; a Chart may reference only `type: "chart"`. Unknown names,
wrong Projection kinds, or mixed union fields fail deterministically.

The runtime parses both unions with dedicated single-pass visitors, not a
retrying untagged parser. This keeps duplicate/unknown-field errors stable and
avoids parsing large rows or series arrays twice.

##### Complete small-data Table plus Combo Chart

This complete source request defines four typed rows once and consumes them as
both a formatted Table and a dual-axis Combo Chart. The translucent Gradient
and stroke are regular Chart series styles; remove transparency or use
`print_mono` when `settings.profile` is `pdfa-1b`.

```json
{
  "datasets": {
    "monthly_results": {
      "fields": [
        { "key": "month", "type": "string" },
        { "key": "revenue", "type": "number", "unit": "currency_usd" },
        { "key": "margin", "type": "number", "unit": "percent" }
      ],
      "rows": [
        { "month": "Jan", "revenue": 42000, "margin": 26.2 },
        { "month": "Feb", "revenue": 58000, "margin": 24.1 },
        { "month": "Mar", "revenue": 51000, "margin": 25.5 },
        { "month": "Apr", "revenue": 67000, "margin": 28.3 }
      ]
    }
  },
  "projections": {
    "monthly_table": {
      "type": "table",
      "dataset": "monthly_results",
      "columns": [
        { "field": "month" },
        { "field": "revenue", "format": { "grouping": true, "prefix": "$" } },
        { "field": "margin", "format": { "decimals": 1, "suffix": "%" } }
      ]
    },
    "monthly_combo": {
      "type": "chart",
      "dataset": "monthly_results",
      "category": { "field": "month" },
      "series": [
        {
          "field": "revenue",
          "name": "Revenue",
          "kind": "bar",
          "value_axis": "primary",
          "format": { "notation": "compact", "prefix": "$" },
          "fill": {
            "gradient": {
              "kind": "linear",
              "angle": 90,
              "stops": [
                { "offset": 0, "color": "#3B82F6", "opacity": 0.72 },
                { "offset": 1, "color": "#3B82F6", "opacity": 0.18 }
              ]
            },
            "opacity": 0.9
          }
        },
        {
          "field": "margin",
          "name": "Margin",
          "kind": "line",
          "value_axis": "secondary",
          "format": { "decimals": 1, "suffix": "%" },
          "stroke": { "color": "#F59E0B", "width": 0.7, "opacity": 0.85 },
          "marker": { "show": true, "shape": "circle", "size": 1.6 }
        }
      ]
    }
  },
  "pages": [
    {
      "size": "a4",
      "elements": [
        {
          "type": "chart",
          "kind": "combo",
          "layout": { "left": 15, "top": 20 },
          "width": 180,
          "height": 92,
          "data": { "projection": "monthly_combo" },
          "theme": { "preset": "business_light" },
          "axes": {
            "values": {
              "primary": {
                "title": "Revenue",
                "position": "left",
                "format": { "notation": "compact", "prefix": "$" }
              },
              "secondary": {
                "title": "Margin",
                "position": "right",
                "format": { "decimals": 1, "suffix": "%" }
              }
            }
          },
          "alt": "Revenue bars rise overall while the margin line ends at its highest value."
        },
        {
          "type": "table",
          "layout": { "left": 15, "top": 126 },
          "width": 180,
          "columns": [
            { "key": "month", "header": "Month", "width": { "mode": "fixed", "value": 45 } },
            { "key": "revenue", "header": "Revenue", "width": { "mode": "fixed", "value": 75 } },
            { "key": "margin", "header": "Margin", "width": { "mode": "fixed", "value": 60 } }
          ],
          "rows": { "projection": "monthly_table" },
          "header": { "show": true },
          "pagination": { "row_min_height": 8, "header_min_height": 9 }
        }
      ]
    }
  ]
}
```

##### Resource limits and errors

Named data applies all existing request, Table, Chart, font, PDF, and token
policy limits. The following values are enforced by the Test runtime. They are
not frozen Production cross-runtime safety promises; Production still requires
Worker, WASIX, and browser capacity evidence. Compatible releases may raise them:

| Limit | Maximum | Counting scope |
|-------|--------:|----------------|
| Datasets | `32` | Root definitions; summed again when Template batch items merge. |
| Projections | `64` | Root definitions; summed again when Template batch items merge. |
| Fields | `128` | Per Dataset. |
| Rows | `1024` | Per Dataset. |
| Table Projection columns | `128` | Per Table Projection. |
| Chart Projection series | `16` | Per Chart Projection; the materialized Chart has the same series cap. |
| Dataset scalar cells | `16384` | Sum of `fields.length × rows.length` across Datasets. |
| Dataset string UTF-8 bytes | `1048576` | String scalar payload across Datasets. |
| Projection references | `256` | Every Table/Chart consumer reference, including repeated references. |
| Materialized output cells | `65536` | Per-reference fan-out across Table cells plus Chart labels/points. |
| Materialized output string UTF-8 bytes | `2097152` | Conservative per-reference output estimate. |
| Materialized estimated heap bytes | `5242880` (`5 MiB`) | Deterministic materializer estimate, not measured process RSS. |

Do not design a Production payload to sit on these Test boundaries until the
target runtime publishes and verifies them. The same Projection output may be cached for reuse, but all request-wide
fan-out budgets still count each consumer reference. Chart Projection output
must also fit the Chart limits of `512` labels, `16` series, and `4096` points;
the runtime never samples or truncates Dataset rows to fit.

Malformed union shapes, unknown fixed-object fields, and parser safety bounds
fail direct JSON decoding with HTTP `400` / `API-001`. Those parser bounds are
more than `64` entries in either root registry, more than `128` Dataset fields
or fields in one Dataset row, more than `1024` Dataset rows, more than `128`
Table Projection columns, or more than `16` Chart Projection series. Dataset
schema, scalar type/nullability, unit/axis, unknown name/field, wrong Projection
kind, consumer Table key mismatch, `33..64` Dataset definitions, and aggregate
source/materialized budgets fail validation with HTTP `400` / `API-002`.
Template-bound output decoding and semantic failures are also `API-002` and
include `data[index]` context. In short, direct JSON parser-bound violations
return `API-001`; aggregate source/materialized budgets return `API-002`. Error
messages use source paths such as `datasets.monthly_results.rows[1].revenue`,
`projections.monthly_combo.series[0].field`,
`pages[0].elements[0].data.projection`, or
`pages[0].elements[1].rows.projection`. The renderer does not silently coerce,
drop, rotate, aggregate, or infer data to repair an invalid request.

### 4.10 Table

A table renders tabular data with optional grouped headers, row headers, cell
merging, alternate row fills, configurable grids, and page-break behaviour.

```json
{
  "type": "table",
  "layout": { "left": 12, "top": 24 },
  "width": 180,
  "pagination": {
    "row_min_height": 10,
    "header_min_height": 12
  },
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
| `layout.left` | `number` | Conditional | Horizontal placement (mm). May be omitted with explicit `layout.flow: true`, which defaults it to `0`. |
| `layout.top` | `number` | Conditional | Required for coordinate placement; forbidden when effective body flow is true, including inherited flow. |
| `layout.flow` | `boolean` | No | Body flow override; inherits document flow when omitted. |
| `layout.gap_after` | `number` | Flow only | Gap after the table (mm); forbidden when effective flow is false. |
| `width` | `number` | No | Total table width. Required when any column uses `percent`, `auto`, or `fit_content`. |
| `columns` | `TableColumn[]` | **Yes** | Column definitions. At least 1. |
| `rows` | `TableRow[]` | **Yes** | Row data. May be empty. |
| `cell` | `TableCellStyle` | No | Default cell style for the whole table. |
| `header` | `TableHeaderConfig` | No | Column header configuration. |
| `row_header` | `TableZoneConfig` | No | Row-header zone configuration. |
| `body` | `TableBodyConfig` | No | Body-zone configuration. |
| `grid` | `TableGridConfig` | No | Grid lines. |
| `pagination` | `TablePaginationConfig` | No | Page-break behaviour. |
| `layout.z_index`, `comment` | | No | Common element fields. |

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
| `width` | `TableColumnWidth` | **Yes** | One of `fixed | percent | auto | fit_content`. |
| `role` | `string` | No | `data` (default) or `row_header`. |
| `cell` | `TableCellStyle` | No | Per-column body cell style. |
| `header_cell` | `TableCellStyle` | No | Per-column header cell style. |

`TableColumnWidth`:

```json
[
  { "mode": "fixed",   "value": 30 },
  { "mode": "percent", "value": 25 },
  { "mode": "auto" },
  { "mode": "fit_content" }
]
```

Rules:

- `columns[].key` must be unique.
- `role = "row_header"` columns must be **contiguous on the left**. You may have multiple row-header columns.
- `fit_content` measures header and body content before flexible width
  allocation. Text, images, and barcodes participate in that measurement.
- `auto` is also content-aware, but it expands to absorb remaining table width
  after fixed, percent, and fit-content allocation.
- Column width and text wrapping are separate decisions. Use
  `wrap_policy = "no_wrap"` in the relevant `TextStyle` when text should stay
  on one line; do not invent a column width mode named `no_wrap`.

#### 4.10.2 Row and cell

`rows` is an array of objects keyed by `columns[].key`.

Shorthand cell (scalar value):

```json
{ "name": "Apple", "qty": 2, "enabled": true, "note": null }
```

The shorthand form accepts only `string`, `number`, `boolean`, or `null`.
Rich block text, spans, images, barcodes, merges, per-cell style, and cell links
must use the complex-cell envelope under that row key.

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

Rich text cell:

```json
{
  "description": {
    "content": {
      "blocks": [
        {
          "type": "paragraph",
          "content": [
            { "type": "text", "text": "Implementation services" }
          ]
        }
      ]
    }
  }
}
```

Image and barcode cells use the same complex-cell envelope:

```json
{
  "logo": {
    "image": {
      "width": 24,
      "height": 9,
      "asset": "brand-logo",
      "format": "png"
    },
    "style": { "text": { "text_align": "center" } }
  },
  "tracking_code": {
    "barcode": {
      "format": "code128",
      "content": "GPDF-0001",
      "width": 40,
      "height": 12,
      "barcode_text": {
        "enabled": true,
        "position": "bottom",
        "offset": 1,
        "style": { "font_size": 6, "text_align": "center" }
      }
    }
  }
}
```

Complex cell fields:

| Field | Type | Notes |
|-------|------|-------|
| `content` | `string \| number \| boolean \| null \| BlockTextContent` | Cell content. Scalars, or block text under the table profile (§4.6.4). |
| `image` | `TableCellImage` | Cell image content. Body rows only. Mutually exclusive with `content` and `barcode`. |
| `barcode` | `TableCellBarcode` | Cell barcode content. Body rows only. Mutually exclusive with `content` and `image`. |
| `row_span` | `integer` | `>= 1`. Merge downward; must not extend past the last row. |
| `col_span` | `integer` | `>= 1`. Merge rightward; must not extend past the last column. |
| `style` | `TableCellStyle` | Per-cell override. |
| `link` | `LinkSpec` | Cell-level link. Only on complex cells. |

Rules:

- `null` renders as an empty string. `boolean` renders as `"true"` / `"false"`.
- Rich block text is valid in body rows only as `cell.content`. Do not put a
  bare `{ "blocks": [...] }` object directly as a `rows[]` cell value.
- A complex cell may set at most one of `content`, `image`, or `barcode`.
- `image` uses the same source contract as §4.8, but omits element placement
  `layout`, `comment`, and `link`; `width` and `height` are required.
  Table-cell image `rotation` must be `0`.
- `barcode` uses the same `format`, `style`, `options`, and `barcode_text`
  contract as §4.7, but omits element placement `layout`, `comment`, and
  `link`; `format`, `content`, `width`, and `height` are required.
  Table-cell barcode `rotation` must be `0`.
- Image and barcode cells participate in row-height measurement and table pagination. `barcode_text` height and offset are included in the measured row height.
- `style.text.text_align` can align table-cell images and barcodes horizontally. Use `content_offset_x` / `content_offset_y` for small manual adjustments.
- `columns[].header` and `header.rows[].cells[]` remain text/rich-text only; table-cell media is supported only in body `rows`.
- Spans cannot exceed the table boundary or overlap each other. Violations return `API-002`.
- Rows containing a key not declared in `columns[].key` return `API-002` (no silent ignore).

#### 4.10.3 Cell style

```json
{
  "padding": {
    "space": "physical",
    "left": {
      "value": 1,
      "unit": "mm"
    },
    "right": {
      "value": 1,
      "unit": "mm"
    },
    "top": {
      "value": 1,
      "unit": "mm"
    },
    "bottom": {
      "value": 1,
      "unit": "mm"
    }
  },
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

Do not put frame-like `TextStyle` fields such as `width`, `height`,
`vertical_align`, `text_overflow`, `shrink_to_fit`, or `min_font_size` inside
`table.cell.text`, `header.cell.text`, `columns[].cell.text`,
`columns[].header_cell.text`, or `cell.style.text`. Table cell text uses the
table text profile, which rejects frame-like fields. Use table width, column
width, physical `Insets` padding, `content_offset_x`, `content_offset_y`, and `text_align`
instead.

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
- Each grouped header row accepts `cells` and an optional row-wide `cell` style.
  Each grouped header cell requires `content` and accepts `col_span`, `row_span`,
  and `style`; it does not accept `link` or unknown fields. Header content and
  `columns[].header` use `TableTextContent`, not body-cell image/barcode media.
- The top-left corner cell is the leftmost row-header column's own `header` value.
- `header.show = false` ignores `header.rows`, `columns[].header`, and `header_cell`.
- Header text renders with middle vertical alignment by default. Do not set
  `text.vertical_align` in `header.cell`, `columns[].header_cell`, grouped header
  cell styles, or body cell styles; table text uses `table_text_profile`, which
  rejects frame-like text style fields. Use `pagination.header_min_height`,
  `padding.top` / `padding.bottom`, or `content_offset_y` for visual adjustment.

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
| `row_min_height` | `number > 0` | — | Minimum body row height (mm). |
| `header_min_height` | `number > 0` | — | Minimum header row height (mm). |

When a body table effectively has `layout.flow: false`, `pagination.row_min_height`
is required. If that table's header is visible, `pagination.header_min_height`
is also required. Whenever either minimum is present, it must be finite and
greater than zero.

Fixed Canvas body table pagination uses the measured page fragment height. Without margins,
the fragment limit is `page.height - footer.layout.height` and body offset is
`0`. With `settings.layout.page_margin` or `pages[].layout.page_margin`, body
offset is the effective top margin and the fragment limit is the smaller of
`page.height - effective_bottom_margin` and `page.height - footer.layout.height`.
Flow body tables instead use the Page Master body region defined in
[§4.2.1](#421-flow-document-contract). In either source mode, the visible table
header consumes its measured height on each fragment. Each
body row fits only when `used_height + measured_row_height <= available_height`;
measured row height includes text, padding, images, barcodes, barcode text, and
minimum heights. For strict documents, render-verify pagination instead of
assuming a fixed row count per page.

The public table contract supports minimum row heights, not fixed per-row
heights. There is no public `row_height` field and rows do not accept a
per-row height override. Approximate visual height with `padding`, font size,
`content_offset_y`, `row_min_height`, and `header_min_height`; use positioned
elements or a purpose-built template when strict fixed-row geometry is required.

#### 4.10.7 Width and style precedence

Width:

- `columns.length >= 1`.
- All `columns[].width` use `fixed`, `percent`, `auto`, or `fit_content`.
- If `table.width` is set:
  - `fixed` columns reserve their mm.
  - `percent` columns take a share of `table.width`.
  - `fit_content` columns measure header/body content first, then take their
    preferred width when it fits; if preferred content width exceeds the
    remaining space, fit-content columns are scaled to fit.
  - `auto` columns absorb the remainder, sized by content measurement. When
    both `fit_content` and `auto` are present and their preferred widths exceed
    the remaining space, both flexible groups are scaled proportionally.
  - With no `auto` or `fit_content` column, the resolved widths must fill
    `table.width` exactly (otherwise `API-002`).
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

### 4.11 Table followed by container

Use sibling body elements for invoice and statement totals: a `table` followed
by one or more `container` elements. Enable flow on those siblings so the
container starts after the measured table height, including table pagination.

```json
{
  "type": "table",
  "width": 84,
  "columns": [
    { "key": "description", "header": "Description", "width": { "mode": "auto" } },
    { "key": "amount", "header": "Amount", "width": { "mode": "fixed", "value": 28 },
      "cell": { "text": { "text_align": "right" } } }
  ],
  "rows": [
    { "description": "Cloud compute", "amount": "$823.10" },
    { "description": "Replica storage", "amount": "$214.50" }
  ],
  "layout": { "left": 8, "flow": true, "gap_after": 6 }
}
```

```json
{
  "type": "container",
  "width": 84,
  "height": 16,
  "layout": {
    "left": 8,
    "flow": true,
    "children": { "mode": "coordinate" }
  },
  "elements": [
    { "type": "text", "content": "Subtotal", "layout": { "left": 44, "top": 0 } },
    { "type": "text", "content": "$1,037.60", "style": { "width": 28, "text_align": "right" },
      "layout": { "right": 0, "top": 0 } },
    { "type": "line", "x1": 44, "y1": 7, "x2": 84, "y2": 7 }
  ]
}
```

Rules:

- Put both elements directly in `pages[].elements`.
- Use `table.layout.flow: true` and `table.layout.gap_after` for the vertical
  gap between the table and the trailing container.
- Use `container.layout.flow: true` on the following container so it is placed
  after the table's measured rendered height.
- The trailing container uses normal container-local coordinates. Use
  `layout.right` for right-aligned amounts inside the container.
- `container.elements` cannot contain `table`; put additional tables as sibling
  body elements.
- Table pagination remains owned by the table. Following flow containers start
  only after the table has fully rendered.

### 4.12 Header and footer

In Fixed Canvas, root `header` and `footer` are page-global decorations applied
to every page. This section describes their coordinate rules; Flow Document
uses Page Master repeated regions as defined in [§4.2.1](#421-flow-document-contract).

```json
{
  "header": {
    "layout": { "height": 14 },
    "elements": [
      {
        "type": "text",
        "content": "Monthly Report",
        "style": {
          "font_family": "NotoSans-Regular",
          "font_mode": "prefer",
          "font_size": 10,
          "font_weight": "bold",
          "color": "#111827",
          "width": 80
        },
        "layout": {
          "left": 12,
          "top": 8
        }
      }
    ]
  },
  "footer": {
    "layout": { "height": 12 },
    "elements": [
      {
        "type": "text",
        "content": "Page 1 / 12",
        "style": {
          "font_family": "NotoSans-Regular",
          "font_mode": "prefer",
          "font_size": 8,
          "color": "#6B7280",
          "width": 38,
          "text_align": "right"
        },
        "layout": {
          "left": 54,
          "top": 6
        }
      }
    ]
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `layout.height` | `number` | **Yes** | Region height (mm). |
| `elements` | `Element[]` | No | Region elements. May be empty. |

`header.elements` and `footer.elements` may include `table` for repeated
tabular page furniture. Keep body line-item tables, totals, notes, and
signatures in `pages[].elements` so flow and table-followed-by-container layout
remain body-owned.

Coordinate semantics:

- `header.elements` use section-local vertical coordinates. By convention place them inside `layout.top ∈ [0, header.layout.height]`.
- For top-level positioned `header` / `footer` elements with a `layout.left` field, horizontal placement is margin-relative when `settings.layout.page_margin` or `pages[].layout.page_margin` is active: rendered left = effective left margin + `layout.left`. Negative section `layout.left` values may move content toward the paper edge, but they must not be less than `-effective_left_margin`; without margins, negative section `layout.left` values are invalid. `layout.anchor` keeps normal anchor semantics.
- `footer.elements` are auto-shifted by `page.height - footer.layout.height` at render time. Write them as if `layout.top = 0` is the top of the footer region.
- Header height does **not** push body elements down; body coordinates are controlled by `settings.layout.page_margin` / `pages[].layout.page_margin` as described in §4.3.
- Footer height reduces the body pagination region. Splittable Text/Table/Container fragments and whole-fit Chart blocks use that limit. Fixed Canvas ordinary geometry can draw a partial frame past the limit when its top still lies inside the body; it moves whole only when its top reaches the limit. Footer reservation is not automatic clipping.

Recommended:

- Set `header.layout.height` and `footer.layout.height` close to actual content height. Oversized regions waste body space.
- For header/footer divider rules, use `line`. Its endpoints are section-local
  geometry and its visual bounds include the resolved stroke and marker
  extents.
- For per-page-different headers/footers, do not use these global regions; embed the content in `pages[].elements` instead.

Text inside `header` / `footer` follows the **section profile** (§4.6.4): no
`list`, no `page_break`, no `frame.overflow = paginate`, no multi-column.

### 4.13 Layers (background, watermark, stamp)

`layers` are document-level decorative layers. They do not participate in
body pagination and do not consume `header` / `footer` height. Use them
for diagonal "DRAFT" / "PRIVATE COPY" watermarks, page background colour
or paper textures, and "PAID" / approval stamps.

```json
{
  "layers": {
    "background": {
      "repeat": "all_pages",
      "fill": { "color": "#FFFBEB" },
      "elements": []
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
          "cx": 74, "cy": 120, "r": 12,
          "stroke": { "color": "#B91C1C", "width": 0.9 }
        },
        {
          "type": "text",
          "layout": { "left": 62, "top": 116 },
          "content": "PAID",
          "style": {
            "font_size": 10,
            "font_weight": "bold",
            "color": "#B91C1C",
            "width": 24,
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
| `background` | Bottom-most | `repeat`, `fill`, `elements[]` | Page colour, paper backgrounds, decorative frames. |
| `watermark` | Above body | Algorithmic spec: `template` + `style` + `layout` + `opacity` | Diagonal-tiled "DRAFT" / "PRIVATE COPY" / brand text. |
| `stamp` | Top-most | `repeat`, `elements[]` | "PAID" / approval marks, often on the last page. |

Common fields:

- `repeat`: `all_pages` (default), `first_page`, `last_page`.

`background` rules:

- `fill`: optional `FillStyle` painted across the exact PDF page MediaBox
  before every background element selected by `repeat`.
- `elements`: optional absolutely positioned decorative elements in physical
  page coordinates.

`background` / `stamp` element rules:

- Allowed types: `text`, `image`, `rect`, `line`, `circle`, `ellipse`, `polygon`, `path`, `link`.
- Forbidden types: `table`.
- `flow` and `gap_after` are forbidden recursively.
- Every `rect` requires explicit, finite `width` and `height` greater than
  zero. A zero-sized `rect` is never expanded to the page. Use
  `layers.background.fill` for a page-wide fill.

`watermark` rules:

- `template.type` is currently only `text`.
- `layout.preset`: `center`, `tile`, `diagonal_tile`, `arc_outside`, `arc_inside`, `wave`.
- A single `watermark` selects one `layout.preset` behavior. Standard placement (`center`, `tile`, `diagonal_tile`) and path text placement (`arc_outside`, `arc_inside`, `wave`) are mutually exclusive.
- `opacity` is in `[0, 1]`.
- For `center`, `tile`, and `diagonal_tile`, `layout.angle` is text rotation in degrees.
- For `arc_outside` and `arc_inside`, text is placed along a circular baseline. This is path text placement, not glyph-outline distortion.
  - `center_x` / `center_y`: optional circle centre in mm. Defaults to page centre.
  - `radius`: optional circle radius in mm. Defaults to `min(page.width, page.height) * 0.28`.
  - `angle`: optional arc anchor angle in degrees. Defaults to `90` for `arc_outside` and `270` for `arc_inside`.
- For `wave`, text is placed along a sine-wave baseline.
  - `start_x` / `start_y`: optional wave start point in mm. Defaults to centred horizontally and page centre vertically.
  - `amplitude`: optional wave amplitude in mm. Default `6`.
  - `wavelength`: optional wave length in mm. Default `42`.
- Path watermark limits are enforced by validation:
  - Single-line text only.
  - At most 32 grapheme clusters.
  - `style.width`, `style.height`, `style.text_overflow`, `style.shrink_to_fit`, `style.background`, `style.decoration`, `style.link_style`, and `style.wrap_policy` are not supported for path watermarks.
  - `gap_x`, `gap_y`, and `stagger_x` are only valid for tiled presets.

Path watermark example:

```json
{
  "template": {
    "type": "text",
    "content": "PRIVATE COPY"
  },
  "style": {
    "font_size": 16,
    "font_weight": "bold",
    "color": "#B91C1C"
  },
  "opacity": 0.14,
  "layout": {
    "preset": "arc_outside",
    "center_x": 105,
    "center_y": 148,
    "radius": 58,
    "angle": 90
  }
}
```

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
      "file_name": "archive-report-20260310.pdf"
    },
    "profile": "pdfa-2b",
    "layout": {
      "flow": true,
      "gap_after": 4,
      "pagination": {
        "continuation_top_gap": 8,
        "continuation_top_gap_with_header": 5
      },
      "page_margin": { "top": 10, "right": 12, "bottom": 10, "left": 12 }
    }
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `defaults` | `Defaults` | Global default styles. See §4.14.1. |
| `metadata` | `Metadata` | PDF metadata. See §4.14.2. |
| `output` | `OutputSettings` | Response shape. See §4.14.3. |
| `profile` | `string` | PDF/A profile. See §6.4. |
| `layout` | `DocumentLayout` | Global layout controls: `flow`, `gap_after`, `start_top`, `page_margin`, and `pagination`. |
| `e_invoice` | `EInvoiceSettings` | **Only valid on `POST /api/v1/e-invoice/render`.** Sending it to JSON Render returns `API-002`. See §5. |
| `security` | `SecuritySettings` | PDF password + permission protection. See §4.15. |

`DocumentPaginationSettings` (Fixed Canvas only):

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `continuation_top_gap` | `number` | `8` | Top gap in mm for generated continuation pages when no positive-height `header` is present and no `settings.layout.page_margin` / `pages[].layout.page_margin` is configured. |
| `continuation_top_gap_with_header` | `number` | `5` | Gap in mm after `header.layout.height` for generated continuation pages when `header.layout.height > 0` and no `settings.layout.page_margin` / `pages[].layout.page_margin` is configured. |

These fields affect generated continuation content in both coordinate placement and effective flow. Each explicit source page uses its own coordinate placement or `start_top`; generated continuations use the continuation origin instead of resetting `start_top`. When `settings.layout.page_margin` or `pages[].layout.page_margin` is configured, continuation content starts at the next page's content-box top instead of using these gaps.

Flow Document does not use these gaps; every continuation page uses its Page
Master body region as defined in [§4.2.1](#421-flow-document-contract).

`settings.layout.flow`:

- Default is disabled; omitted `flow` preserves explicit `layout.left/top` body layout.
- `settings.layout.flow: true` affects only body `pages[].elements`, not `header`,
  `footer`, `layers`, or watermark content.
- On each explicit source page, flow starts at `settings.layout.start_top`, or
  `0` when omitted, in the body content-box coordinate system. Generated
  continuation pages use the continuation origin described above, for both
  whole deferred elements and resumed fragments.
- Flow planning respects each explicit source page. It does not flow elements from one explicit
  `pages[]` entry into the next; plan each page's `pages[].elements` array
  independently.
- Element-level `layout.flow` overrides the global setting:
  `element.layout.flow ?? settings.layout.flow ?? false`.
- Gap-after precedence is `element.layout.gap_after ?? settings.layout.gap_after ?? 0`.
- Effective flow uses one rule whether enabled globally or on the element:
  `layout.top` and `layout.bottom` are forbidden, and source order plus
  `gap_after` fully owns vertical placement.
- A direct body element may set `layout.flow: false` to opt out of global flow.
  It then uses Coordinate Placement, does not consume cursor space,
  and cannot declare `gap_after`.
- Planning follows the JSON source array order, not `layout.z_index`; `layout.z_index` only
  controls drawing order.
- Geometry elements such as `line`, `circle`, `ellipse`, and `polygon` are
  translated as a whole so their visual top aligns with the cursor. Resolved
  stroke and marker extents are part of their flow bounds.
- Flow never infers grouping from overlap, a rectangle, source coordinates,
  typography, or a rewound Y value. Put same-row or background-backed content
  in an explicit `container`.
- With `settings.layout.page_margin` or `pages[].layout.page_margin`, Fixed Canvas Sequential Flow uses the same body content
  box coordinate system as ordinary body elements.
- Tables advance following flow elements by their measured rendered height.
  Text advances by measured height, unless explicit `frame.height` or
  `style.height` controls the box height.
- Containers advance by their fixed `height`, or by their measured auto height
  when `height` is omitted. Auto height is clamped by `min_height` when set.
- For table totals or notes, put a `container` after the `table` and enable
  `layout.flow: true` on both siblings. The table's `layout.gap_after` controls
  the gap before the trailing container.
- A paginated container follows the direct-body or Flow block scope in
  §4.9.8; ContainerItem descendants and non-flow body containers cannot paginate.
- Fixed Canvas pagination is element-specific: Text/Table and supported
  Containers can split; Chart uses whole-fit placement; ordinary geometry
  retains partial-frame drawing and can extend past the body bottom. Use the
  Flow Document mode for content-first structural pagination.

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
| `text` | `TextStyle` | Default text style. If `font_family` is set here without `font_mode`, `strict` is used. Use `font_mode = "prefer"` with the same `font_family` when this default should allow fallback, for example mixed Latin + CJK text. |
| `stroke` | `StrokeStyle` | Default stroke for shapes and table grids. |
| `fill` | `FillStyle` | Default fill. Default opacity is `0` (transparent) when omitted. |
| `shape` | `ShapeDefaults` | `corner_radius` (mm) for default rounded rectangles. |

Do not set `settings.defaults.fill` just to make elements white. Prefer
element-specific `fill` on the rectangle, table header, cell, or container that
needs a visible background.

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

- `binary` (or omitted): `Content-Disposition` starts with `inline`; browsers preview inline.
- `file`: `Content-Disposition` starts with `attachment`; browsers download.
- Both modes emit a safe ASCII `filename="..."` fallback. When the requested name contains Unicode or characters that are unsafe in the quoted fallback, the header also emits RFC 5987 `filename*=UTF-8''...`; clients should prefer `filename*` when present. Path separators and control characters are replaced before either parameter is generated.
- Any value other than `binary` / `file` returns `API-002`.

### 4.15 Security: PDF password and document permissions

`settings.security` turns on PDF standard-security-handler encryption
on the output PDF — AES-128 or AES-256, optional open password, optional
owner password, and eight per-action permission flags (print / copy /
modify / annotate / fill_forms / extract_accessibility / assemble /
print_high_quality). **Only valid on `POST /api/v1/pdf/render`.** Mutually
exclusive with `settings.profile` (PDF/A) and `settings.e_invoice` —
combining either returns `API-002`.

```json
{
  "settings": {
    "security": {
      "algorithm": "aes_128",
      "open_password": "reader-demo",
      "owner_password": "owner-demo",
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

> The example uses `aes_128` to match the default and the lowest tier that ships encryption (Pro). For AES-256 — PDF 2.0 standard-security-handler revision 6 — switch `algorithm` to `aes_256`; that path is Enterprise-only (see tier policy below). Both algorithms accept the same field shape.

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

### 4.16 Default value precedence

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

All gPdf errors share the JSON envelope shown in [§1.6](#11-request-and-response-basics).
Codes fall into four families: client (`API-0xx`), authentication (`API-1xx`),
billing/entitlement (`API-2xx`), and rendering/system (`API-5xx` / `API-9xx`).

| Code | HTTP | Family | Trigger | Typical `message` | What to do |
|------|------|--------|---------|-------------------|------------|
| `API-001` | `400` | Client | JSON syntax or request-shape decoding failed. | `Invalid JSON payload`, `unknown field`, parser safety bound | Validate syntax and correct the reported field, enum, union shape, or parser bound. |
| `API-002` | `400` | Client | Body decoded but failed semantic or business validation. | `<field> must be >= 0`, `Missing required field <name>`, aggregate resource bound | Read `message`. The offending path or relationship is included. |
| `API-004` | `400` | Client | Total page count exceeds the per-request limit (token policy or platform max, whichever is smaller). | `page count exceeds max_pages_per_request` | Split the request into smaller batches, or request a higher policy limit. |
| `API-007` | `400` | Client | An image exceeds the renderer's per-image byte limit (`32 KiB` raster, `256 KiB` SVG) or decoded-size limit: an inline (`base64`) image, or an asset whose stored candidates all exceed it. A token policy's `max_image_bytes` bounds only the base64 payload of an inline image, and answers `API-002`. | `Image payload too large for png: <n> bytes (max 32768 bytes)`, `Image asset <key>: Image payload too large for ...`, `Image asset <key>: Image payload too large: <n> bytes (max 262144 bytes for any image format)` | Re-encode or downscale the image below the limit. An asset has the same renderer limit, so replace an oversized asset rather than switching to one. |
| `API-008` | `400` | Client | Request body exceeds the platform body limit (default `16 MiB`; some deployments differ). | `Request body too large` | Reduce inline payload (especially base64 images). Consider splitting the document. |
| `API-101` | `401` | Auth | `Authorization` header is missing or not in `Bearer YOUR_TOKEN` form. | `Missing or malformed Authorization header` | Add the header. The format is exactly `Bearer ` followed by the token. |
| `API-102` | `403` | Auth | Authentication failed (for example, rejected or inactive token, or token without PDF-render authorization). | `Authentication failed` (redacted) | Verify the token is active and valid for the public API. |
| `API-103` | `403` | Auth | Token is blacklisted (revoked, suspended, or otherwise invalidated). | `Authentication failed` (redacted) | Rotate the token via the Console, or contact support. |
| `API-201` | `402` | Billing | No active subscription entitlement for this token. | `No active subscription` | Activate or renew a plan in the Console. |
| `API-202` | `402` | Billing | Subscription expired. | `Subscription expired` | Renew in the Console. |
| `API-203` | `429` | Billing | Subscription quota exceeded. | `Quota exceeded` | Wait for the next billing cycle, top up, or upgrade the plan. |
| `API-204` | `402` | Billing | Wallet balance insufficient for overage. | `Insufficient wallet balance` | Top up via the Console. |
| `API-501` | `500` | Render | PDF generation failed during rendering. | Detailed message describing the cause | Inspect `message`. Often points to invalid font, asset, or coordinate. |
| `API-502` | `500` | Render | PDF/A compliance check failed after rendering. | `PDF/A compliance check failed: <reason>` | Adjust the offending field (commonly fonts not in the embedded set, or non-PDF/A images). |
| `API-503` to `API-507` | `500` | Render | Specific rendering subsystem failures (font, asset resolution, layout). | Detailed message | Inspect `message`. These preserve actionable detail intentionally. |
| `API-504` | `500` | Render | Resource loading failed: auto / `prefer` font fallback found no covering font, or the storage of a `strict` font or an image asset did not answer (a transient failure). | `Font fallback failed for: <run>`, `Font temporarily unavailable: <face> (storage or network failure; retry the request)`, `Image temporarily unavailable: <key> (...)` | For a `temporarily unavailable` message, retry the same request with backoff. Otherwise declare a `font_family` that covers the script, or check the font or image name. Custom fonts are added by the operator on request. |
| `API-506` | `500` | Render | An image asset is not stored under its key. A runtime instance remembers the absence for up to 10 minutes. | `Image not found: <key>` | Check the asset key and `format`. An asset added after a miss can take up to 10 minutes to be found everywhere. |
| `API-900` | `500` | System | Internal system error. | Redacted message | Retry once. If it persists, contact support with the `req_id`. |
| `API-999` | `500` | System | Unknown internal error. | Redacted message | Same as `API-900`. |

Notes on validation errors (`API-002`):

- `API-002` is the most common error. Common triggers include:
  - More than one of `left`, `right`, and `anchor` provided on the same element.
  - Custom `page.width` / `page.height` outside the supported `10..=2000 mm` range.
  - `font_mode` provided without a same-level `font_family`.
  - Explicit font in `strict` mode that does not cover the submitted text.
  - Invalid `link` (unsupported URL scheme, page index out of bounds, malformed `padding` / `border`).
  - Invalid `chart` (data lengths differ, kind-specific fields conflict, a secondary axis is unbound, stack groups are incompatible, or a radial value is null/negative).
  - Invalid Named Dataset/Projection input (schema/type/nullability mismatch, unsafe unit token, unknown Dataset/Projection/field, wrong Projection kind, mixed inline/reference shape, Table output-key mismatch, or a data/materialization safety bound exceeded).
  - Invalid `table` (unknown column key, `table.width` cannot allocate a positive width to undeclared columns, invalid span).
  - Invalid `profile` value.
  - Invalid `settings.security` (password >32 UTF-8 bytes, policy doesn't permit algorithm, combined with `settings.profile` or `settings.e_invoice`, or `permissions` provided without `owner_password`). Unrecognised `algorithm` value is rejected at JSON deserialisation with `API-001`.
  - A `data.projection` or `layout.flow_document` request to a deployment that does not serve the capability. The body's optional `capability` field names the id; see [§1.6](#16-request-and-response-basics).

Notes on resource failures:

- A font named by a `strict` style is loaded before rendering. If its storage does not answer (a transient storage or network failure), the request returns `API-504` with a `Font temporarily unavailable` message; retry it with backoff. If the font is not stored, the font coverage check answers `API-002` as for any strict miss.
- A font tried in `prefer` or auto mode does not fail the request when its storage does not hold it or does not answer: rendering falls back through the bundled families (`API-504` only if nothing covers the text).
- An image asset answers `API-506` when it is not stored, `API-007` when every stored candidate exceeds the per-image byte limit (other candidates of the key are tried first), and `API-504` with an `Image temporarily unavailable` message when its storage does not answer. Only `API-506` is remembered; the other two are retried by the next request.

Notes on redaction:

- `API-102`, `API-103`, and `API-9xx` deliberately use generic messages and never carry `capability`. They will not tell you whether a token exists or why authentication failed. This is by design — to defeat token enumeration.
- `API-201` to `API-204` (billing) preserve actionable text so end users know whether to renew, top up, or wait for the cycle.
- `API-501` to `API-507` (render) preserve actionable text so engineering can debug.

### 6.2 Limits

Three kinds of limits apply to every request:

1. **Platform limits** — fixed across all tenants, set per deployment.
2. **Policy limits** — bound to the token, set when the plan is provisioned.
3. **Request-shape limits** — encoded in the request schema itself.

| Limit | Default | Scope | Override path | Triggered error |
|-------|---------|-------|---------------|-----------------|
| Pages per request | `50` pages, or a lower token policy limit | Platform plus token policy | Plan or per-token policy in the Console. | `API-004` |
| Request body size | `16 MiB` transport limit | Platform | Per-deployment env var; some private deployments raise it. This is not a complexity or render-time guarantee. | `API-008` |
| Image bytes per element | Raster images: `32 KiB`; SVG: `256 KiB`, inline or asset; a token policy's `max_image_bytes` may bound an inline base64 payload further | Renderer plus token policy | Re-encode the image, use a smaller asset, or adjust the token policy if your plan allows it. | `API-007`; the token policy bound `API-002` |
| SVG images | Filters: `1048576` pixels per region, `64` primitives per filter, `numOctaves` up to `8`, `1024` kernel taps, `16777216` bytes of raster memory per filtered group and `16777216` estimated work units per image, shared by its filtered groups; limits cover content nested inside filtered groups; the raster resolution is lowered first when a region, the raster memory or the work would exceed a limit; embedded images follow the decoded size limits; `svgz` rejected | Renderer | Not configurable; simplify filters or rasterize the artwork. | `API-002` |
| Decoded PNG and WebP size | `8192` pixels per side, `4194304` pixels, `16777216` decoded bytes per image | Renderer | Not configurable; downscale the image before embedding it. | `API-007` |
| Template batch size (`data` array) | `10` items | Platform | Not configurable. Split into multiple requests if you need more. | `API-002` (`Template render data item count ... exceeds max 10`) |
| URL TTL for e-invoice artifacts | `300` seconds | Request | `delivery.url_ttl_seconds`, range `1..900` | `API-002` if out of range |
| Retention for e-invoice artifacts | `23` hours | Request | `retention.ttl_hours`, range `1..23` | `API-002` if out of range |
| `xml.content` (e-invoice) | `2 MiB` | Platform | Not configurable. | `API-002` |
| Named Datasets (Test candidate) | `32` definitions, `128` fields/Dataset, `1024` rows/Dataset, `16384` total scalar cells, `1048576` source string UTF-8 bytes | Request shape | Not configurable. Test enforces these values; Production remains gated on cross-runtime evidence. Compatible releases may raise them. | Direct JSON: `API-001` for parser bounds (`65+` registry entries, `129+` fields, `1025+` rows or `129+` fields in one row); `API-002` for `33..64` Dataset definitions and aggregate cells/string bytes. Template output: `API-002` with `data[index]` context. |
| Named Projections (Test candidate) | `64` definitions, `128` columns/Table Projection, `16` series/Chart Projection, `256` consumer references | Request shape | Not configurable; reduce or split the source request. | Direct JSON: `API-001` for `65+` definitions, `129+` Table columns, or `17+` Chart series; `API-002` for aggregate consumer references. Template output: `API-002` with `data[index]` context. |
| Materialized Projection output (Test candidate) | `65536` cells, `2097152` string UTF-8 bytes, `5242880` estimated heap bytes | Request-wide materialization | Not configurable; reduce rows, projected fields/series, or repeated consumers. | `API-002` |
| Barcode inputs | `4096` content UTF-8 bytes; `64` bytes per format or shape name; `bearer_bar.thickness` `0..=64` modules when bearer bars are drawn; `1048576` encoded modules, or `65536` with `module_shape: "circle"` (not MaxiCode) | Request shape | Not configurable; shorten content or use a 2D format for long payloads. | `API-002` |
| Table grid cells | `65536` per table, counted as (body rows + shown header rows) × columns, where a shown header counts its grouped rows plus the column-label row; `262144` summed over all tables in one request or Template batch | Request shape | Not configurable; reduce columns or split long tables across requests. | `API-002` (`table grid cells exceeds current safety limit 65536`, `request table grid cells exceeds current safety limit 262144`) |

How to pre-check on the client side:

- Check page count first. Split requests before they reach the active per-request page limit; page count is the most reliable public boundary for render work.
- Sum your request body bytes and reject above ~`14 MiB` before sending. This leaves headroom for proxies and avoids the transport-level `API-008`.
- If you embed images via `source.kind = "base64"`, the base64 form is ~33% larger than the raw bytes. A 5 MiB raw JPEG becomes ~6.7 MiB in JSON.
- If you batch templates, never put more than `10` items in `data`. Use multiple HTTP calls for larger batches.
- If you use Named Dataset/Projection, pre-check the published data limits and the existing per-Chart `512` labels / `16` series / `4096` points limits. Do not assume a `1024`-row Dataset can be projected into one Chart without reducing it before submission.

### 6.3 Response headers

| Header | Direction | Always present | Notes |
|--------|-----------|----------------|-------|
| `Content-Type` | Response | Yes | `application/pdf` on success, `application/json` on error. E-invoice job and artifact endpoints also return `application/json`. |
| `Content-Disposition` | Response | Yes (PDF responses only) | `inline` by default; `attachment` when `output.mode = "file"`. Includes a safe ASCII `filename` fallback and, for Unicode or legacy-unsafe names, RFC 5987 `filename*=UTF-8''...`. |
| `X-Request-Id` | Both | Yes | Echoed from the request if the client supplied it; otherwise generated. |
| `X-GPDF-Build-Profile` | Response | Native and Wasmer server `POST /health` only | Name of the server's build profile, for diagnostics. |
| `X-GPDF-Enabled-Capabilities` | Response | Native and Wasmer server `POST /health` only | Comma-separated capability ids the server serves, a set. See [Capability discovery](#capability-discovery). |
| `X-GPDF-Catalog-Version` | Response | Native server `POST /health` only; the Wasmer package from its next release | Version of the capability id vocabulary; the hosted API reports it as `catalog_version` in the `/health` JSON. |

Headers gPdf does **not** currently emit (and the absence is intentional in v1):

- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- `Retry-After`
- `Idempotency-Key` echo
- `X-Render-Time-Ms`
- `X-Render-Engine-Version`

Do not depend on these. If a future version adds any of them, this document
will list them and clients can opt in.

### 6.4 Fonts and PDF/A profiles

Font resolution is controlled by `font_family` and `font_mode`:

- **Auto**: when no `font_family` is declared anywhere in the inheritance chain, the renderer chooses fonts that cover each text run from the bundled font set. `font_mode = "auto"` is not a public input value.
- **`prefer`**: when `font_family` and `font_mode = "prefer"` are declared in the same style object, the renderer tries the declared family first and falls back through bundled families for glyphs it cannot cover.
- **`strict`**: when `font_family` is declared with `font_mode = "strict"`, or declared without `font_mode`, the renderer must use that family. If the family cannot cover the text, the request fails with `API-002`.

Practical CJK guidance:

- If you do not declare `font_family` anywhere, auto mode can select bundled CJK fonts.
- If you declare a Latin/default family such as `NotoSans-Regular` or `RobotoMono-Regular` and the text may contain Chinese, Japanese, or Korean characters, set `font_mode = "prefer"` in the same style object.
- If you declare a family without `font_mode = "prefer"`, the request is strict. Mixed CJK text that the declared family cannot cover returns `API-002`.

Failure modes:

- Strict coverage miss → `API-002`. Adjust the text or pick a font that covers it, or switch the same style object to `font_mode = "prefer"`.
- Strict font whose storage did not answer → `API-504` (`Font temporarily unavailable: ...`). Retry the request with backoff.
- Auto / prefer total fallback miss → `API-504`. The bundled set could not cover the text in any family.

Bundled CJK fallback maps Hangul to `NotoSansKR-Regular`, Japanese kana to
`NotoSansJP-Regular`, and CJK ideographs / fullwidth punctuation to
`NotoSansSC-Regular`. The bundled set also includes Latin, Greek, Cyrillic,
Arabic, Hebrew, Bengali, Tamil, Thai, Vietnamese, Osage, and monospace fonts.
This is not full Unicode coverage; unsupported scripts or symbols may still
fail in auto / prefer mode with `API-504`. Custom fonts are added by the
operator on request, after which `font_family` can reference them; there is no
self-service font upload.

Common built-in font family names that can be declared explicitly when needed:
`RobotoMono-Regular`, `RobotoMono-Bold`, `NotoSans-Regular`, `NotoSans-Bold`,
`NotoSansSC-Regular`, `NotoSansSC-Bold`, `NotoSansJP-Regular`,
`NotoSansJP-Bold`, `NotoSansKR-Regular`, `NotoSansKR-Bold`,
`NotoSansArabic-Regular`, `NotoSansArabic-Bold`, `NotoSansThai-Regular`,
`NotoSansHebrew-Regular`, `NotoSansBengali-Regular`,
`NotoSansTamil-Regular`, `NotoSans_ru-RU`, `NotoSans_vi-VN`,
`NotoSansOsage-Regular`, and `Mansalva-Regular`. When unsure, omit
`font_family` and let auto fallback choose.

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
- Horizontal axis: rightward. Vertical axis: downward.
- `rotation` is in degrees, clockwise. `text` and `image` accept arbitrary angles. Barcodes and their attached `barcode_text` should use only `0/90/180/270`; the current PDF renderer treats other integer barcode angles as `0`.

Runnable coordinate map:

```json
{
  "settings": {
    "layout": {
      "page_margin": { "top": 8, "right": 8, "bottom": 8, "left": 8 }
    }
  },
  "pages": [
    {
      "size": "label_100_150",
      "elements": [
        {
          "type": "rect",
          "layout": { "left": 0, "top": 0 },
          "width": 84,
          "height": 134,
          "fill": { "color": "#F8FAFC" },
          "stroke": { "color": "#CBD5E1", "width": 0.4 }
        },
        {
          "type": "rect",
          "layout": { "left": 0, "top": 0 },
          "width": 2,
          "height": 2,
          "fill": { "color": "#2563EB" }
        },
        {
          "type": "line",
          "x1": 2,
          "y1": 1,
          "x2": 32,
          "y2": 1,
          "stroke": { "color": "#2563EB", "width": 0.5 }
        },
        {
          "type": "line",
          "x1": 1,
          "y1": 2,
          "x2": 1,
          "y2": 32,
          "stroke": { "color": "#2563EB", "width": 0.5 }
        },
        {
          "type": "text",
          "layout": { "left": 3, "top": 3 },
          "content": "content origin (0,0)",
          "style": { "font_size": 6.5, "color": "#1E3A8A", "width": 50 }
        },
        {
          "type": "text",
          "layout": { "left": 20, "top": 25 },
          "content": "left=20, top=25",
          "style": { "font_size": 7, "color": "#111827", "width": 35 }
        },
        {
          "type": "rect",
          "layout": { "left": 20, "top": 34 },
          "width": 36,
          "height": 20,
          "fill": { "color": "#DBEAFE" },
          "stroke": { "color": "#1D4ED8", "width": 0.6 }
        },
        {
          "type": "text",
          "layout": { "left": 21.5, "top": 41 },
          "content": "36 x 20 mm",
          "style": { "font_size": 7, "color": "#1E40AF", "width": 33, "text_align": "center" }
        },
        {
          "type": "text",
          "layout": { "left": 28, "top": 76 },
          "rotation": 30,
          "content": "30 degrees clockwise",
          "style": { "font_size": 7, "color": "#B45309", "width": 38 }
        }
      ]
    }
  ]
}
```

In this example the physical page is `100 × 150 mm`. The `8 mm` page margin
makes the body content box `84 × 134 mm`; every body element above uses
coordinates relative to that content-box origin.

---

## 7. Changelog

This document tracks the public API contract. Internal changes that do not
affect callers are not listed here.

### 2026-10-26

- Resource failures answer by their cause. A transient storage or network
  failure while loading a font that a `strict` style names returns `API-504`
  (`500`, `Font temporarily unavailable: <face> ...`) instead of the strict
  coverage answer `API-002` (`400`); retry it. The same failure for an image
  asset returns `API-504` instead of `API-506` "Image not found". An image
  asset whose stored candidates all exceed the per-image byte limit returns
  `API-007` (`400`), as an inline image does, instead of `API-506` (`500`),
  after the key's other candidates are tried; it is no longer remembered as
  missing. Fonts in `prefer` and auto mode still fall back on any failure,
  and absent fonts and images keep `API-002` and `API-506`. See
  [§6.1](#61-error-codes).
- An extension-less image asset key resolves by its `format`: two requests
  that name the same key with different formats each get the stored image
  their format finds first, where before the first request's image could be
  served to both.
- When the storage of a higher-priority stored variant of a font or an image
  asset does not answer, the request answers as for a transient failure
  (`API-504` for a strict font or an image; a `prefer` or auto font falls
  back), instead of a lower-priority variant answering and staying cached.
- A template that has no active runtime when it is looked up is looked up
  again after 30 seconds instead of 15 minutes, so a template activated
  after a failed lookup is served within that time.
- On the native and Wasmer servers, a `font_family` that no font source can
  hold, such as one with a space, falls back in `prefer` mode and answers
  `API-002` in strict mode, as on the hosted API, instead of `API-505`; an
  image file that exists but cannot be read answers `API-504` instead of
  `API-506`.
- The `API-007` advice no longer suggests switching to an asset (assets have
  the same per-image limit), and custom fonts are described as added by the
  operator on request; there is no Console font upload.

### 2026-10-25

- A deployment that does not list `layout.flow_document` and reports
  `catalog_version` 2 or later does not decode a `flow_document` value. Every
  request that carries one gets `API-002` with `capability`
  `layout.flow_document`, including a Flow Document that is malformed, over a
  limit, `null` or sent with `pages`, and one that references projections
  only inside its content. A Template Render runtime with a Flow Document
  layout gets the Template Render form of that rejection whether or not it
  declares the capability. An older deployment without the capability may
  reject the root as an unknown field with `API-001`. Deployments that list
  `layout.flow_document` answer every request as before. See
  [§3.1](#31-source-root-fixed-canvas-or-flow-document).
- Capability catalog version 2 adds `render.svg`: SVG images, emoji and
  OpenType SVG glyphs in PDF output. Every deployed gPdf build that renders
  PDFs compiles it, so a deployment that reports `catalog_version` 2 lists
  `render.svg` in `enabled_capabilities` and `X-GPDF-Enabled-Capabilities`.
  Requests, SVG and emoji rendering, codes and messages are unchanged. See
  [Capability discovery](#capability-discovery).

### 2026-10-15

- Error bodies gain an optional `capability` field. It names the capability id
  when the request needs `data.projection` or `layout.flow_document` and the
  target build does not compile or enable it, including a Template Render
  runtime that requires one of them. Codes, messages and HTTP statuses are
  unchanged, and every other error body keeps exactly `error`, `code`,
  `message` and `req_id`. See [§1.6](#16-request-and-response-basics).
- `POST /health` reports `catalog_version`, the version of the capability id
  vocabulary, beside `enabled_capabilities`; Native servers send it as the
  `X-GPDF-Catalog-Version` header, and the Wasmer package does from its next
  release. See [Capability discovery](#capability-discovery).

### 2026-10-14

- Barcode elements, including container and layer barcodes, reject empty or
  whitespace-only `content` with `API-002`, the rule table-cell barcodes and
  GFL already applied. Previously some 2D formats rendered a blank symbol and
  other formats failed with `API-501`.
- Barcodes are encoded with xBarcode 2.0.0. Every documented `format` value and
  alias is unchanged.
  - Content over a format's own capacity returns `API-002`, as the barcode
    resource-limit rule states. Previously 2D over-capacity input (QR, Data
    Matrix, PDF417 content too large for the largest symbol) returned
    `API-501`. Some linear formats now cap their content: `code39` and
    `code93` at `256` characters, `codabar` at `256` data characters between
    its start and stop characters, `msi` and its variants at `256`, and
    Code 128 at `1536`.
  - Some symbols use fewer codewords for the same content and never a larger
    symbol; the encoded text is unchanged. QR content made only of QR
    alphanumeric characters (uppercase letters, digits, space and `$%*+-./:`)
    that contains a run of 7 or more digits, such as tracking numbers or GS1
    element strings, may get a different module pattern or a smaller version,
    in plain and GS1 mode. Data Matrix, PDF417, MicroPDF417 and Aztec text may
    encode more compactly, and `code128` text with runs of Latin-1 characters
    may be narrower.
  - Full-range `aztec` symbols (longer content; compact symbols are
    unchanged) now draw the centre row and column of the reference grid.
    Symbol size and encoded text are unchanged.
  - `microqr` content with characters U+0080 to U+00FF is encoded as
    ISO-8859-1, the Micro QR default character set, and Japanese text uses
    Kanji mode. These symbols change and may be smaller, and Latin-1 text now
    decodes correctly. ASCII content is unchanged.
  - Non-GS1 `code128`, and unrecognized names that fall back to it, encode
    `~1`..`~4`, `~~` and `{FNC1}`..`{FNC4}` as literal text; gPdf never
    documented them as function codes. `gs1128` and `options.gs1: true` keep
    their FNC1 handling, and `code128a`, `code128b` and `code128c` were already
    literal.
  - `qrcode`, `pdf417`, `datamatrix`, `aztec` and `maxicode` ignore
    `options.bearer_bar`. Other formats draw a recognized `bearer_bar.shape`
    with a thickness of at least `1` module when they support bearer bars,
    which now includes `gtin8`, `gtin12`, `gtin13`, `sscc` and `upus10`. A
    format without bearer bars still returns `API-501`, now with the message
    "options.bearer_bar is not supported for this barcode format".
  - Encoder error messages (`API-501`) use the xBarcode 2.0 wording, except
    "Unsupported barcode format" and the bearer-bar message above.
  - Newly rejected input: `upce` and `upcecomposite` number system `1`; GS1
    element strings with conflicting precision, or with a GS1 Syntax
    Dictionary exclusion pair such as (01) with (8006), in `gs1128`,
    `gs1qrcode`, `gs1datamatrix`, `options.gs1` and the UPC composites; raw
    UPC-composite 2D components with a leading, repeated, trailing or missing
    GS separator; `codabar` made only of start and stop characters, such as
    `AB`; and Micro QR input without a single-mode representation.
  - Newly accepted input: Data Matrix, PDF417, MicroPDF417 and Micro QR
    content that now fits; `upcacomposite` and `upcecomposite` with a `+2` or
    `+5` add-on; and 111 GS1 Application Identifiers added from the GS1
    Barcode Syntax Dictionary.

### 2026-10-07

- Table cells and grouped header cells reject `row_span` / `col_span` values
  that extend past the last row or column on every runtime, with `API-002`.
  Header spans are checked by the renderer as well as the validator.
- Added Table grid limits: `65536` cells per table and `262144` cells per
  request or Template batch, counted as (body rows + shown header rows) ×
  columns. Larger tables return `API-002` before layout. See [§6.2](#62-limits).
- Barcode elements and table-cell barcodes enforce resource limits on every
  runtime: `4096` content UTF-8 bytes, `64`-byte format and shape names,
  `bearer_bar.thickness` from `0` to `64` modules when bearer bars are drawn,
  and `1048576` encoded modules (`65536` with circle modules, except
  MaxiCode). Violations return `API-002` instead of exhausting memory or
  failing as a render error. Preview barcode commands report `module_shape`
  as `circle` or `square`, matching the PDF output.
- PNG and WebP images are checked against `8192` pixels per side, `4194304`
  pixels and `16777216` decoded bytes from their headers before decoding;
  larger images, and WebP chunks that extend past the image data, return
  `API-007`.
- SVG images resolve only embedded `data:` images (no file system access),
  reject `svgz`, and bound filters before rasterization: lower resolution when
  needed, then `API-002` beyond `1048576` pixels per region, `64` primitives,
  `numOctaves` `8` or `1024` kernel taps, including content nested inside
  filtered groups, or when the per-SVG work limit is exceeded at every
  resolution. Malformed convolution kernels no longer crash the renderer.
- Native and Wasmer servers accept Template Render `template_id` values made of
  ASCII letters, digits, `.`, `_` and `-` only, excluding `.` and `..`; other
  values return `API-002` before template lookup.

### 2026-08-27

- Added the capability-gated `flow_document` source root as the content-first
  alternative to Fixed Canvas `pages`. The roots are mutually exclusive and
  converge on the same resolved layout, Preview/LayoutPlan, and PDF replay
  pipeline.
- Added typed Page Master, Content Segment, logical destination, page/column
  break, pagination constraint, Columns, Stack, Grid, Placed Content, semantic
  reading-order, named style, and scoped selector contracts with bounded
  validation.
- Added `layout.flow_document` capability gating. The full Worker, Server,
  standalone, and Wasmer profiles may advertise it; the shipping-label profile
  remains excluded. Documentation or OpenAPI presence alone is not runtime
  deployment proof.

### 2026-08-26

- Added Chart P0-P2: Waterfall, continuous XY Scatter/Bubble, Range/Error Band, Heatmap, and reference lines/bands. Spark, Progress Circle, and Bullet remain documented compositions of stable public primitives.
- Extended Chart Projection with explicit `category`, `xy`, and `heatmap` shapes plus `size`, `lower`, `upper`, and `point_kind` field references. Existing category projections remain valid because `shape` defaults to `category`.
- Extended PreparedChart/Layout WASM with additive optional X/row axes while keeping PDF and Preview on the same compiled geometry.

### 2026-08-24

- Added the implementation-candidate Named Dataset/Projection contract: typed root `datasets`, typed Table/Chart `projections`, strict inline/reference unions, Combo and dual-axis series styles, opaque unit tokens, request-wide safety limits, and `data.projection` capability gating. This changelog entry does not itself prove Test or Production deployment.
- Added the native `chart` element with Bar, Line, Area, Combo, Pie, and Donut kinds. Combo and a maximum of two value axes are included in the first public contract through `series[].kind`, `series[].value_axis`, and `axes.values.primary/secondary`.
- Added normal and percent stacks, deterministic number formats, category/data-label overflow policies, four chart presets, and PDF/UA chart alternative-text requirements.
- Added Chart-owned `ChartFillStyle` with mutually exclusive linear-gradient and pattern paint sources while leaving the existing global `FillStyle` contract unchanged.
- Documented the already-enforced `16384` resolved-text-run and `1048576` resolved-run UTF-8 byte limits per compiled Chart, plus the deterministic `API-502` failure for transparent Chart Gradient or Pattern paint under PDF/A-1.

### 2026-05-31

- Docs-only canonical English rewrite (no API changes). The opening outline now follows the stronger localized structure: API surface, five-second start, common fallback rules, complete request skeleton, sandbox, response basics, and non-goals. §3 now explains the current `DocumentRequest` surfaces and the table / container layout decision model using behaviour verified against the current Rust schema and validators.
- Docs-only English reference enrichment (no API changes). The overview now includes the full public route map and endpoint boundary notes. Quick Start now explains the intentionally omitted fields in the minimum request, the three public `text.content` forms, and common default fallback behaviour so the English source can serve as the base for future localized versions.
- Docs-only coordinate example (no API changes). §6.5 now includes a runnable coordinate map that shows content-box origin, horizontal/vertical direction, millimetre dimensions, and clockwise rotation on a 100 × 150 mm label page.

### 2026-05-18

- Added constrained path-text watermark presets to `layers.watermark.layout.preset`: `arc_outside`, `arc_inside`, and `wave`. The feature places short single-line text along circular or sine-wave baselines; it does not distort glyph outlines. Validation caps path watermark text at 32 grapheme clusters and rejects layout/style fields that would make rendering expensive or ambiguous.

### 2026-05-17

- Docs-only outline reshuffle (no API changes). §4.13 renamed `Layers` → `Layers (background, watermark, stamp)` so the watermark feature is visible in the right-hand page outline. §4.14.4 `Security (password + permissions)` promoted to its own H3 §4.15 `Security: PDF password and document permissions` for the same reason. Old §4.15 `Default value precedence` renumbered to §4.16. Anchor URLs for §4.13, §4.15, and §4.16 changed; the §4.15 anchor that pointed at default-value-precedence now points at the security section, so update any external bookmarks accordingly. All field names (`settings.security`, `layers.watermark`, etc.) and behaviour are unchanged.

### 2026-05-16

- Added `settings.security` to JSON Render (`POST /api/v1/pdf/render`). Optional AES-128 or AES-256 PDF encryption with `open_password`, `owner_password` (Enterprise policy), and 8 permission bits (`print`, `modify`, `copy`, `annotate`, `fill_forms`, `extract_accessibility`, `assemble`, `print_high_quality`). Passwords capped at 32 UTF-8 bytes. Mutually exclusive with `settings.profile` (PDF/A) and `settings.e_invoice` — both combinations return `API-002`. Full field table, per-bit effect table, and tier matrix at §4.15 (was §4.14.4 at ship-time; renumbered 2026-05-17).
- Added Security section to the request reference (originally §4.14.4, now §4.15).
- §6.1 `API-002` trigger list extended with `settings.security` misuse cases (password >32 UTF-8 bytes, policy doesn't permit algorithm, combined with `settings.profile` / `settings.e_invoice`, or `permissions` provided without `owner_password`). Unrecognised `algorithm` value returns `API-001` at JSON deserialisation, not `API-002`.

### 2026-05-08

- First publication of the consolidated public API reference.
- Documented the e-invoice job and artifact endpoints (`GET /api/v1/e-invoice/jobs/{job_id}` and `.../artifacts/{artifact}`).
- Single consolidated error-code reference at §6.1.
