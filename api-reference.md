> [!NOTE]
> This file is generated from `doc/contracts/source/gpdf-api-contract.md`. Do not edit it directly.

# gPdf API

> Status: Public API contract
> Last updated: 2026-05-30

This is the single human-maintained public HTTP API contract for gPdf. Fields, endpoints, internal workflows, or debug capabilities not listed here are not part of the public contract. OpenAPI, website docs, and other publication copies are derived from this document.

## 1. Common Contract

Production base URL:

```text
https://api.gpdf.com
```

Public endpoints:

| Capability | Method and path | Purpose |
| --- | --- | --- |
| JSON Render | `POST /api/v1/pdf/render` | Render a complete `DocumentRequest` to PDF. |
| E-Invoice Render | `POST /api/v1/e-invoice/render` | Generate PDF/A-3b e-invoice output with CII XML. |
| E-Invoice Validate | `POST /api/v1/e-invoice/validate` | Validate e-invoice settings and XML inputs. |
| E-Invoice Capabilities | `GET /api/v1/e-invoice/capabilities` | Return public e-invoice capabilities and limits. |
| E-Invoice Artifacts | `GET /api/v1/e-invoice/artifacts/{job_id}/{artifact}` | Download asynchronous e-invoice artifacts. |
| Template Render | `POST /api/v1/template-render` | Render a published template with business data. |

Authentication uses Bearer tokens:

```http
Authorization: Bearer <token>
Content-Type: application/json
```

Errors return a JSON envelope:

```json
{
  "error": {
    "code": "API-002",
    "message": "Validation failed",
    "details": {
      "field": "pages[0].elements[0].text"
    }
  },
  "request_id": "req_..."
}
```

The service returns or propagates `X-Request-Id`. Callers may provide it for diagnostics, but it is not an idempotency key. The public API does not currently promise idempotency-key semantics, retry windows, or fixed rate-limit headers.

Default limits:

| Item | Public limit |
| --- | --- |
| JSON body | 16 MiB by default. |
| Inline raster image | 32 KiB by default. |
| Inline SVG image | 256 KiB by default. |
| Template Render batch size | 10 records by default. |
| E-Invoice XML input | 2 MiB by default. |
| E-Invoice synchronous wait | `wait_for_result_timeout_ms` from 1 to 900 ms. |
| E-Invoice retention | `retention_hours` from 1 to 23 hours. |

Coordinates and dimensions use millimeters by default. The page origin is the top-left corner. If an element uses `x: 0`, it starts at the left edge of the page; production layouts should use `page_margins.left` or an explicit `x` value when a visible margin is required.

`page_margins` affect automatic pagination and flow layout. Absolutely positioned elements still use their own `x/y`. Header and footer areas do not rewrite the absolute coordinates of first-page body elements.

Public PDF/A output only promises:

- `pdfa-1b`
- `pdfa-2b`
- `pdfa-3b`

E-Invoice output uses `pdfa-3b`. Other internal or historical profiles are not public contract.

Style precedence:

1. Element-level style.
2. `settings.defaults`.
3. Service defaults.

Fill, stroke, font, and size use that order. Shape elements default to no fill when no fill is resolved. Font mode can be `strict`, `prefer`, or `auto`: `strict` returns a validation error on missing glyphs; `prefer` and `auto` try fallback, but do not guarantee coverage for every requested glyph or bold/italic face.

`z_index` is an optional integer and defaults to `0`. If an element array is already sorted by `z_index`, or every element uses the default `0`, rendering follows the original JSON order. The renderer only allocates a temporary ordered list when it detects a later element with a smaller `z_index` than the previous element. Callers should set `z_index` only when overlapping elements need an explicit paint order.

## 2. JSON Render API

`POST /api/v1/pdf/render` accepts a `DocumentRequest`.

Top-level fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `settings` | object | No | Page, unit, defaults, output, pagination, and security settings. |
| `pages` | array | Yes | Page array. |
| `metadata` | object | No | PDF metadata. |
| `security` | object | No | PDF permission and encryption settings. |

Layer support:

| Layer | Order | Allowed elements | Limits |
| --- | --- | --- | --- |
| `background` | Before page content | `text`, `barcode`, `image`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `link` | `table` and `stack` are not supported. Background `rect` may omit `x/y/width/height`. |
| `watermark` | After page content and before stamp | Same capability as regular elements, subject to validation | For watermark content. |
| `stamp` | Last | `text`, `barcode`, `image`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `link` | `table` and `stack` are not supported. |

A page background color should be a `rect` in `layers.background.elements`:

```json
{
  "type": "rect",
  "fill": { "color": "#FFFBEB" }
}
```

For a background-layer `rect`, omitted `x`, `y`, `width`, `height`, or `width/height: 0` default to full-page coverage. Omitted `x/y` are equivalent to `0`. Explicit positive `x/y/width/height` values are honored as a partial background. This default only applies to background-layer `rect`; body `rect` elements still require valid dimensions.

Header and footer are drawn with absolute coordinates. `stack` is not allowed in header/footer. `table` currently passes generic element validation there, but it does not participate in body pagination; public integrations should prefer simple elements in header/footer.

Automatic continuation gaps:

| Field | Default | Description |
| --- | --- | --- |
| `settings.pagination.continuation_top_gap` | `8` mm | Continuation-page top gap when no header is present. |
| `settings.pagination.continuation_top_gap_with_header` | `5` mm | Continuation-page top gap after a header area. |

If `page_margins.top` is declared, it affects continuation layout. Otherwise the continuation start uses header height plus the relevant gap. First-page absolutely positioned table or element placement still uses its own `x/y`.

Element types: `text`, `barcode`, `image`, `line`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `link`, `table`, `stack`.

`x_anchor` is supported for `text`, `image`, `barcode`, `rect`, `circle`, `ellipse`, `polygon`, `path`, `link`, and `line`.

`path` does not accept raw SVG `d` strings. It supports structured commands equivalent to `M`, `L`, `Q`, `C`, and `Z`: `move_to`, `line_to`, `quad_to`, `cubic_to`, and `close`. It does not support `A/H/V/S/T`, lowercase relative commands, full `d` strings, or SVG DOM input. A path must start with `move_to`, contain at least one drawing command, and stay inside its `view_box`. Handwritten signatures can be represented after the caller converts sampled points into these structured commands.

Tables support rows, columns, borders, backgrounds, text styles, repeated headers, cell spans, and pagination. Media content inside cells is supported for body rows; header cells should use text.

`stack.children[0]` must be a `table`. Later children must be `block`. `block.elements` supports `text`, `barcode`, `rect`, `line`, `circle`, `ellipse`, `polygon`, `path`, `link`, and `image`. `block.elements` is specific to stack layout and is not the same as text `content.blocks`.

## 3. E-Invoice API

Public e-invoice endpoints:

| Endpoint | Description |
| --- | --- |
| `POST /api/v1/e-invoice/render` | Generate an e-invoice PDF or asynchronous job. |
| `POST /api/v1/e-invoice/validate` | Validate without PDF generation. |
| `GET /api/v1/e-invoice/capabilities` | Return public formats, profiles, limits, and defaults. |
| `GET /api/v1/e-invoice/artifacts/{job_id}/{artifact}` | Download asynchronous job artifacts. |

`/api/v1/e-invoice/render` requires `settings.e_invoice`. Regular `/api/v1/pdf/render` rejects `settings.e_invoice`.

Important fields:

| Field | Description |
| --- | --- |
| `standard` | `factur-x` or `zugferd`. |
| `profile` | Public profile. |
| `xml` | CII XML content or reference. |
| `attachments` | Attachment array. |
| `delivery` | `sync` or `async`. |
| `validation` | Preflight, schema, business rules, and failure policy. |
| `retention_hours` | Artifact retention window. |
| `data_residency` | Data residency requirement. |

E-Invoice output is for PDF/A-3b archival workflows. Callers are responsible for making the XML and visible invoice content consistent.

## 4. Template Render API

The only public template rendering endpoint is:

```http
POST /api/v1/template-render
```

Callers provide a published template identifier and business data. Template creation, compilation, validation, AI parsing, publishing, and unpublishing are internal or control-plane capabilities, not public business APIs.

Request fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `template_id` | string | Yes | Published template ID. |
| `version` | string | No | Specific template version. |
| `data` | object | No | Single-render data. |
| `batch` | array | No | Batch-render data, 10 records by default. |
| `options` | object | No | Output, filename, metadata, and similar options. |

Use either `data` or `batch`. Batch records should keep a consistent data shape. Template variables, rich-text variables, image variables, and barcode variables resolve according to the binding plan stored with the published template.

## 5. Error Codes

| Code | HTTP | Meaning |
| --- | --- | --- |
| `API-001` | `400` | Invalid JSON structure or type. |
| `API-002` | `400` | Schema or business validation failure. |
| `API-003` | `413` | Request body too large. |
| `API-004` | `415` | Unsupported content type. |
| `API-101` | `401` | Missing or invalid authentication. |
| `API-102` | `403` | Resource access is forbidden. |
| `API-201` | `404` | Resource not found. |
| `API-301` | `429` | Too many requests. |
| `API-401` | `500` | Render or internal service error. |
| `API-504` | `422` | Resource, font, image, or attachment unavailable. |

Callers should rely on `error.code` and HTTP status, not natural-language `message` text.
