# gPdf E-Invoice Render API

`POST /api/v1/e-invoice/render` produces a PDF/A-3b document with a CII XML
attachment, packaged according to the Factur-X or ZUGFeRD standard.

There are four endpoints in this family:

| Endpoint | Auth | Purpose |
|----------|------|---------|
| `GET /api/v1/e-invoice/capabilities` | None | Static capability registry. |
| `POST /api/v1/e-invoice/render` | Required | Render. Returns either inline PDF or a job descriptor. |
| `GET /api/v1/e-invoice/jobs/{job_id}` | Required | Poll an object-mode job. |
| `GET /api/v1/e-invoice/jobs/{job_id}/artifacts/{artifact}` | Required | Download a job artifact. |

For the underlying `DocumentRequest` shape (pages, elements, fonts), see
[the JSON Render API reference](/docs/api-reference/). Token / quota / 4xx
codes shared across endpoints live in the same document's §6.

## 1. Capabilities

`GET /api/v1/e-invoice/capabilities` returns the static product capability
registry. **Authentication is not required** — this advertises what the
platform supports, not what your token is allowed to do. Per-token
authorisation is checked at render time.

| Property | Value |
|----------|-------|
| Method | `GET` |
| Path | `/api/v1/e-invoice/capabilities` |
| Auth | None |
| Response | `200`, `Content-Type: application/json` |

```bash
curl "https://api.gpdf.com/api/v1/e-invoice/capabilities"
```

Sample response:

```json
{
  "schema_version": 1,
  "package": "pdfa3_embedded_xml",
  "pdfa_profile": "pdfa-3b",
  "standards": [
    {
      "standard": "factur_x",
      "standard_id": "factur_x",
      "name": "Factur-X",
      "profile": "en16931",
      "document_type": "invoice",
      "xml_format": "cii",
      "embedded_file_name": "factur-x.xml"
    },
    {
      "standard": "zugferd",
      "standard_id": "zugferd",
      "name": "ZUGFeRD",
      "profile": "en16931",
      "document_type": "invoice",
      "xml_format": "cii",
      "embedded_file_name": "zugferd-invoice.xml"
    }
  ],
  "delivery_modes": ["inline_pdf", "object"],
  "data_residency_modes": ["auto", "eu", "global"]
}
```

## 2. Render

| Property | Value |
|----------|-------|
| Method | `POST` |
| Path | `/api/v1/e-invoice/render` |
| Auth | Required — `Authorization: Bearer <token>` |
| Request `Content-Type` | `application/json` |
| Success (`inline_pdf`) | `200`, `Content-Type: application/pdf` |
| Success (`object`) | `200`, `Content-Type: application/json` (job descriptor; see [§5](#5-job-lookup)) |

The request body is a `DocumentRequest` (same shape as JSON Render) **plus**
a required `settings.e_invoice` block. The PDF/A profile is fixed:

- If `settings.profile` is omitted or empty, gPdf uses `pdfa-3b`.
- If `settings.profile` is set, it must be `pdfa-3b`. Anything else returns `API-002`.
- Sending `settings.e_invoice` to `POST /api/v1/pdf/render` returns `API-002` — the field is route-specific.

Minimum request:

```json
{
  "settings": {
    "profile": "pdfa-3b",
    "e_invoice": {
      "standard": "factur_x",
      "profile": "en16931",
      "document_type": "invoice",
      "xml": {
        "format": "cii",
        "encoding": "utf8",
        "content": "<?xml version=\"1.0\" encoding=\"UTF-8\"?><rsm:CrossIndustryInvoice>...</rsm:CrossIndustryInvoice>"
      }
    }
  },
  "pages": [
    { "size": "a4", "elements": [] }
  ]
}
```

```bash
curl -X POST "https://api.gpdf.com/api/v1/e-invoice/render" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @einvoice.json \
  --output invoice.pdf
```

## 3. `settings.e_invoice` fields

```json
{
  "standard": "factur_x",
  "profile": "en16931",
  "document_type": "invoice",
  "xml": {
    "format": "cii",
    "encoding": "utf8",
    "content": "<?xml version=\"1.0\" ... ?>"
  },
  "validation": {
    "level": "basic",
    "execution": "sync",
    "fail_on_warnings": false
  },
  "delivery": {
    "mode": "inline_pdf",
    "url_ttl_seconds": 900
  },
  "report": {
    "enabled": false
  },
  "retention": {
    "ttl_hours": 23,
    "store_request": false
  },
  "data_residency": {
    "mode": "auto",
    "country": "DE",
    "strict": false
  },
  "job_id": "0190f0e8-2b8a-7ad4-9b3e-1234567890ab"
}
```

Top-level fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `standard` | `"factur_x" \| "zugferd"` | **Yes** | Envelope standard. |
| `profile` | `"en16931"` | **Yes** | Semantic profile. Currently `en16931` only. |
| `document_type` | `"invoice"` | **Yes** | Currently invoice only. |
| `xml` | `EInvoiceXmlPayload` | **Yes** | The CII XML payload. |
| `validation` | `EInvoiceValidationSettings` | No | Validation policy. Default basic + sync. |
| `delivery` | `EInvoiceDeliverySettings` | No | Response shape. Default `inline_pdf`. |
| `report` | `EInvoiceReportSettings` | No | Whether to produce a validation report artifact. |
| `retention` | `EInvoiceRetentionSettings` | No | Retention for object-mode artifacts. |
| `data_residency` | `EInvoiceDataResidencySettings` | No | Storage residency profile. |
| `job_id` | `string` | No | Idempotency ID for object delivery. UUID v7 recommended. |

### 3.1 `EInvoiceXmlPayload`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `format` | `"cii"` | **Yes** | Currently CII only. |
| `encoding` | `"utf8"` | No | Default `utf8`. |
| `content` | `string` | **Yes** | UTF-8 XML content. Non-empty. Max `2 MiB`. |

`xml.content` is checked for basic structural sanity. Full EN 16931
Schematron is not run on the inline path; request strict validation via
`validation.level = "strict"` to schedule full checking.

### 3.2 `EInvoiceValidationSettings`

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `level` | `"basic" \| "strict"` | `basic` | `basic` = lightweight request-level checks. `strict` = full third-party validation. |
| `execution` | `"sync" \| "async"` | `sync` | `async` requires `level = strict` **and** `delivery.mode = object`. |
| `fail_on_warnings` | `boolean` | `false` | When `true`, any warning in the strict report fails the job. |

Rules:

- `level = strict` requires `delivery.mode = object`. Strict validation does not return inline.
- `execution = async` requires `level = strict` **and** `delivery.mode = object`. The job is queued and updated when validation completes.

### 3.3 `EInvoiceDeliverySettings`

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `mode` | `"inline_pdf" \| "object"` | `inline_pdf` | `inline_pdf` returns PDF bytes inline. `object` returns JSON + stores artifacts retrievable via the job API. |
| `url_ttl_seconds` | `integer` | `900` | Range `1..900`. TTL of artifact download URLs. |

When `mode = inline_pdf`:

- The render endpoint returns `application/pdf` immediately.
- No job is created.
- No artifacts are stored.

When `mode = object`:

- The render endpoint returns `application/json` describing the job (see [§5](#5-job-lookup)).
- Artifacts are accessible via the artifact API (see [§6](#6-artifact-download)) until they expire.

### 3.4 `EInvoiceReportSettings`

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `enabled` | `boolean` | `false` | When `true`, produces a `report.json` artifact alongside the PDF/XML. Requires `delivery.mode = object`. |

The report is produced asynchronously by the validator. It is added to the
job's `artifacts` once available. Until then `artifacts.report.available =
false`.

### 3.5 `EInvoiceRetentionSettings`

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `ttl_hours` | `integer` | `23` | Range `1..23`. Retention for object-mode artifacts and job metadata. |
| `store_request` | `boolean` | `false` | When `true`, the original request body is also stored as an artifact. |

After `ttl_hours` elapses, artifact downloads return `404` and the job
lookup returns metadata only (no usable artifact URLs).

### 3.6 `EInvoiceDataResidencySettings`

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `mode` | `"auto" \| "eu" \| "global"` | `auto` | Storage residency profile for artifacts and job metadata. |
| `country` | `string` | — | ISO 3166-1 alpha-2 country code. Used by `auto` and `strict` mode. |
| `strict` | `boolean` | `false` | When `true` and the country cannot be resolved to a residency profile, returns `API-002`. |

Behaviour:

- `auto` resolves residency from `country` (e.g. `DE` → `eu`, `US` → `global`).
- `eu` forces EU residency regardless of country.
- `global` forces global residency.
- `strict = true` makes ambiguous or missing country a hard error rather than a fallback.

## 4. Standard mapping

| `standard` | Embedded file | XMP namespace | XMP version | XMP conformance | XMP document type | AFRelationship |
|------------|---------------|---------------|-------------|-----------------|-------------------|----------------|
| `factur_x` | `factur-x.xml` | `urn:factur-x:pdfa:CrossIndustryDocument:invoice:1p0#` | `1.0` | `EN 16931` | `INVOICE` | `Alternative` |
| `zugferd` | `zugferd-invoice.xml` | `urn:zugferd:pdfa:CrossIndustryDocument:invoice:2p0#` | `2p0` | `EN 16931` | `INVOICE` | `Alternative` |

Both standards embed CII XML conforming to EN 16931 and use the
`Alternative` AFRelationship per the Factur-X / ZUGFeRD baseline.

## 5. Job lookup

`GET /api/v1/e-invoice/jobs/{job_id}` returns the current state of an
object-mode job, including artifact URLs.

| Property | Value |
|----------|-------|
| Method | `GET` |
| Path | `/api/v1/e-invoice/jobs/{job_id}` |
| Auth | Required — same token that created the job. |
| Response | `200`, `Content-Type: application/json` |

```bash
curl "https://api.gpdf.com/api/v1/e-invoice/jobs/$JOB_ID" \
  -H "Authorization: Bearer $GPDF_TOKEN"
```

Sample response:

```json
{
  "job_id": "0190f0e8-2b8a-7ad4-9b3e-1234567890ab",
  "status": "completed",
  "delivery": "object",
  "data_residency": "eu",
  "created_at_ms": "1715140800000",
  "expires_at_ms": "1715223600000",
  "artifacts": {
    "pdf": {
      "available": true,
      "content_type": "application/pdf",
      "bytes": 184231,
      "expires_at_ms": "1715141700000",
      "url": "https://api.gpdf.com/api/v1/e-invoice/jobs/0190f0.../artifacts/pdf"
    },
    "xml": {
      "available": true,
      "content_type": "application/xml",
      "bytes": 8421,
      "expires_at_ms": "1715141700000",
      "url": "https://api.gpdf.com/api/v1/e-invoice/jobs/0190f0.../artifacts/xml"
    },
    "report": {
      "available": false,
      "content_type": "application/json",
      "bytes": 0,
      "expires_at_ms": "1715141700000",
      "url": "https://api.gpdf.com/api/v1/e-invoice/jobs/0190f0.../artifacts/report"
    }
  },
  "validation": {
    "level": "strict",
    "execution": "async",
    "status": "pending",
    "report_requested": true
  }
}
```

| Field | Type | Notes |
|-------|------|-------|
| `job_id` | `string` | Echoes the request's `job_id` if supplied; otherwise generated. |
| `status` | `string` | `pending`, `completed`, `failed`. |
| `delivery` | `string` | `object` (this endpoint only exists for object jobs). |
| `data_residency` | `string` | `eu` or `global`. |
| `created_at_ms`, `expires_at_ms` | `string` | Unix milliseconds as decimal strings. |
| `artifacts` | object | Per-artifact descriptor. Each entry has `available`, `content_type`, `bytes`, `expires_at_ms`, `url`. |
| `artifacts.pdf` | object | The signed PDF/A-3b. Always present. |
| `artifacts.xml` | object | The embedded CII XML, also exposed as a separate artifact. Always present. |
| `artifacts.request` | object (optional) | Original request body. Present only when `retention.store_request = true`. |
| `artifacts.report` | object (optional) | Validation report. Present only when `report.enabled = true`. |
| `validation.status` | `string` | `pending`, `completed`, `failed`. `completed` means the report was produced — it does **not** mean the invoice is valid. Read `report.valid` inside the report artifact for that. |

Notes:

- Artifact URLs always point to the gPdf job API. Authenticate URL fetches with the same Bearer token.
- `validation.status = "failed"` indicates the validator pipeline failed to produce a clean report; the report artifact, if any, contains a failure envelope.
- `expires_at_ms` on the job and on each artifact may differ because each artifact URL has its own short TTL (`url_ttl_seconds`, default `900`).

## 6. Artifact download

`GET /api/v1/e-invoice/jobs/{job_id}/artifacts/{artifact}` returns the raw
artifact bytes.

| Property | Value |
|----------|-------|
| Method | `GET` |
| Path | `/api/v1/e-invoice/jobs/{job_id}/artifacts/{artifact}` |
| Auth | Required — same token that created the job. |
| Response | `200`, `Content-Type` matches the artifact (`application/pdf`, `application/xml`, `application/json`). |

`{artifact}` is one of:

- `pdf`
- `xml`
- `report` (only when `report.enabled = true`)
- `request` (only when `retention.store_request = true`)

```bash
curl "https://api.gpdf.com/api/v1/e-invoice/jobs/$JOB_ID/artifacts/pdf" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  --output invoice.pdf
```

If the artifact has expired or is not yet available, the response is `404`.

## 7. Validation flows

There are two distinct end-to-end flows depending on `validation.level` and
`delivery.mode`. Pick by what the caller needs to see at request time.

### 7.1 Inline flow — basic validation, sync, PDF returned immediately

Default. Use this when you only need request-level XML structure checks and
want the PDF in the same HTTP round-trip.

```text
       client                              gPdf
         │                                  │
    1    │  POST /api/v1/e-invoice/render   │
         │ ────────────────────────────────>│
         │                                  │ ─┐
         │                                  │  │ basic XML structure check
         │                                  │  │ embed CII as attachment
         │                                  │  │ render PDF/A-3b
         │                                  │ <─┘
    2    │  200 application/pdf             │
         │<──────────────────────────────── │
         │                                  │
```

Settings: `validation.level = "basic"`, `validation.execution = "sync"`,
`delivery.mode = "inline_pdf"` (all defaults).

Properties:

- One round-trip. Total time typically `200–800 ms`.
- No job is created. No artifacts are stored.
- `xml.content` only goes through structural sanity checks; full EN 16931
  Schematron is **not** run on this path.

### 7.2 Object-delivery flow — strict validation, async, polled job

Use this when you need full third-party validation (e.g. a verifiable
EN 16931 report alongside the PDF) and you can tolerate eventual delivery.

```text
       client                              gPdf
         │                                  │
    1    │  POST /api/v1/e-invoice/render   │
         │ ────────────────────────────────>│
         │                                  │ ─┐
         │                                  │  │ embed CII, render PDF/A-3b
         │                                  │  │ store pdf + xml artifacts
         │                                  │  │ schedule strict validator
         │                                  │ <─┘
    2    │  200 application/json (pending)  │
         │<──────────────────────────────── │
         │                                  │
         │           (background validator runs out-of-band)
         │                                  │
    3    │  GET /api/v1/e-invoice/jobs/{id} │   ←── poll
         │ ────────────────────────────────>│
         │  200 application/json (pending)  │
         │<──────────────────────────────── │
         │           ...                    │
         │  GET /api/v1/e-invoice/jobs/{id} │   ←── eventually
         │ ────────────────────────────────>│
         │  200 application/json (completed,│
         │     report.json now available)   │
         │<──────────────────────────────── │
         │                                  │
    4    │  GET .../artifacts/pdf           │
         │ ────────────────────────────────>│
         │  200 application/pdf             │
         │<──────────────────────────────── │
```

Settings: `validation.level = "strict"`, `validation.execution = "async"`,
`delivery.mode = "object"`, optional `report.enabled = true`.

Properties:

- Step 2 returns immediately (`< 1s`) with a job descriptor — see [§5](#5-job-lookup).
- `pdf` and `xml` artifacts are typically available **before** validation completes — you can download the PDF as soon as Step 2 returns; the report is what takes time.
- Step 3 returns `validation.status = "completed"` once the validator pipeline produces a report; that does **not** mean the invoice is valid — read `report.valid` inside the report artifact for that.
- `validation.status = "failed"` means the validator pipeline itself failed; the report artifact, if any, contains a failure envelope rather than a clean EN 16931 report.

### 7.3 Polling cadence

Recommended back-off schedule for Step 3 above:

| Poll | Wait since previous poll | Cumulative time |
|------|--------------------------|-----------------|
| 1st  | 5 s after Step 2         | 5 s             |
| 2nd  | 10 s                     | 15 s            |
| 3rd  | 20 s                     | 35 s            |
| 4th  | 30 s                     | 65 s            |
| 5th+ | 60 s                     | 125 s, 185 s, … |

Most strict jobs reach `completed` within `60 s`. A job still `pending` after
`5 minutes` is unusual — capture the `job_id` and report it.

## 8. Errors specific to e-invoice

In addition to the codes in [the JSON Render reference's §6.1](/docs/api-reference/),
the e-invoice endpoint surfaces:

| Trigger | Code | Note |
|---------|------|------|
| Token policy disallows `document.e_invoice` | `API-002` | Contact your account contact to add the entitlement. |
| Disallowed `standard`, `profile`, or `document_type` | `API-002` | Use only standards listed in `/api/v1/e-invoice/capabilities` and allowed by your token. |
| `xml.content` exceeds `2 MiB` | `API-002` | Reduce the XML or split the document. |
| `data_residency.strict = true` with unresolved country | `API-002` | Provide an explicit `mode` or a known `country`. |
| Artifact / job not found or expired | HTTP `404` with `API-002` envelope | The retention window has passed. |
