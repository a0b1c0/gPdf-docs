
# gPdf Template Render API

> Status: Public API surface. Authoritative source for the public docs site.
> Last updated: 2026-05-08
>
> This document defines the Template Render API. For the lower-level JSON
> Render API, see `api-reference.en.md`. For template authoring (designing,
> validating, and publishing templates), see the internal
> `template-authoring.en.md` — that document is **not** part of the public
> docs site.


## 1. When to use this API

Use the Template Render API when:

- You only want to send a `template_id` and business `data` and get back a PDF.
- You do not want to handle page sizes, coordinates, element trees, or pagination.
- Multiple systems (ERP, OMS, WMS, billing) need to coordinate around a single document contract.

Use the JSON Render API (`api-reference.en.md`) instead when:

- You need pixel control over layout, custom report shapes, or interactive designer output.
- The document does not match an existing template.

A simple test:

- "Our team agreed on a template name and the fields we'll send" → Template Render.
- "Our team is shipping a new layout that doesn't match anything" → JSON Render.


## 2. Endpoint

| Property | Value |
|----------|-------|
| Method | `POST` |
| Path | `/api/v1/template-render` |
| Auth | Required — `Authorization: Bearer <token>` |
| Request `Content-Type` | `application/json` |
| Success | `200`, `Content-Type: application/pdf` |
| Error | `4xx` / `5xx`, `Content-Type: application/json` |

```bash
curl -X POST "https://api.gpdf.com/api/v1/template-render" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: $(uuidgen)" \
  --data-binary @request.json \
  --output out.pdf
```

Both `https://api.gpdf.com` (production) and `https://api-test.gpdf.com`
(test) host this endpoint. Tokens, templates, and artifacts do not cross
between environments.

Authentication, request ID, error envelope, rate-limit guidance, and limits
are identical to the JSON Render API. See [api-reference.en.md §2](./api-reference.en.md#2-authentication-and-environments)
and [§6](./api-reference.en.md#6-reference) for the shared contract.


## 3. Quick Start

The smallest valid request:

```json
{
  "template_id": "shipping_label",
  "data": [
    {
      "recipient_name": "John Doe",
      "recipient_address": "123 Main St, Los Angeles, CA",
      "sender_name": "Acme Warehouse",
      "sender_address": "88 Harbor Rd, Long Beach, CA",
      "tracking_number": "TRK1234567890"
    }
  ]
}
```

```bash
curl -X POST "https://api.gpdf.com/api/v1/template-render" \
  -H "Authorization: Bearer $GPDF_TOKEN" \
  -H "Content-Type: application/json" \
  --data-binary @label.json \
  --output label.pdf
```

If you get a PDF back, your token, the route, and the template are all
working. Move on to the field reference below.


## 4. Request fields

```json
{
  "template_id": "f8m2zd",
  "data": [
    { },
    { }
  ],
  "output": {
    "mode": "file",
    "file_name": "invoice-001.pdf"
  }
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `template_id` | `string` | **Yes** | Stable template identifier. |
| `data` | `object[]` | **Yes** | One or more data items. Each item is rendered with the same template; the resulting pages are concatenated into a single PDF. |
| `output` | `object` | No | Output mode and file name. |

### 4.1 `template_id`

Stable per-template identifier issued by gPdf. Treat it as opaque.

- `template_id` is what your callers depend on. It does **not** change when the template's display name changes.
- Display names live in the Console, not in the API.
- Some `template_id` values look like random short codes (`k4n82q`, `f8m2zd`, `0q7xmp`); others look like semantic slugs (`invoice`, `shipping_label`). Both are valid; do not parse them.

### 4.2 `data`

Rules:

- Must be a non-empty array.
- Each item must be an object.
- The platform limit on items per request is `10`. Exceeding it returns `API-002`. Split larger batches into multiple requests.
- Each item is rendered independently against the template; the page outputs are concatenated in order into one PDF. Useful for batch printing labels, invoices, or statements.
- The template defines which fields are required, which are typed, and how arrays inside an item are structured. The caller's job is to match that schema.
- Unknown fields are tolerated today (silently ignored). **Do not rely on this.** A future schema may turn unknown fields into validation errors.
- For `required = true` `string` fields, an empty string or whitespace-only string is treated as missing.
- All items in one request must produce the same global structure (`settings`, `layers`, `header`, `footer`). Per-item differences in those globals are rejected with `API-002`. Per-item differences inside the rendered body are fine — that is the whole point.

Multi-item example (3 invoices in one PDF):

```json
{
  "template_id": "invoice",
  "data": [
    {
      "invoice_number": "INV-001",
      "date_of_issue": "2026-03-11",
      "issuer_name": "Acme Cloud Inc.",
      "bill_to_name": "Receiver A",
      "subtotal": "$100.00", "total": "$100.00", "amount_due": "$100.00",
      "items": [
        { "description": "Service A", "qty": 1, "unit_price": "$100.00", "amount": "$100.00" }
      ]
    },
    {
      "invoice_number": "INV-002",
      "date_of_issue": "2026-03-11",
      "issuer_name": "Acme Cloud Inc.",
      "bill_to_name": "Receiver B",
      "subtotal": "$240.00", "total": "$240.00", "amount_due": "$240.00",
      "items": [
        { "description": "Service B", "qty": 2, "unit_price": "$120.00", "amount": "$240.00" }
      ]
    },
    {
      "invoice_number": "INV-003",
      "date_of_issue": "2026-03-11",
      "issuer_name": "Acme Cloud Inc.",
      "bill_to_name": "Receiver C",
      "subtotal": "$60.00", "total": "$60.00", "amount_due": "$60.00",
      "items": [
        { "description": "Service C", "qty": 1, "unit_price": "$60.00", "amount": "$60.00" }
      ]
    }
  ]
}
```

### 4.3 `output`

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `mode` | `"binary" \| "file"` | `binary` | `binary` = `Content-Disposition: inline; filename="..."`. `file` = `Content-Disposition: attachment; filename="..."`. Both return identical PDF bytes. |
| `file_name` | `string` | Auto: `gPdf-MMDDHHmmssSSS.pdf` | Sanitised; `.pdf` is appended automatically. |

Any value other than `binary` / `file` for `mode` returns `API-002`.

```json
{
  "template_id": "invoice",
  "data": [ { "invoice_number": "INV-001" /* ... */ } ],
  "output": {
    "mode": "file",
    "file_name": "INV-001-2026-03-11.pdf"
  }
}
```

### 4.4 Fields the caller does not send

Two field families exist in some internal contexts but **must not** be sent
by callers:

- `revision_id` — gPdf manages template versioning internally. The runtime always resolves the active version for the calling environment. `revision_id` is not part of the public contract.
- Any field starting with `_` (e.g. `_each`, `_item`) — reserved for the template engine itself.

Sending either returns `API-002`.


## 5. Templates and environments

A template's `template_id` is stable across environments, but availability is
environment-scoped:

- Test and production maintain independent active templates. A template available in test is not automatically available in production.
- A template can exist but be unavailable to your token if it is disabled in the current environment, or scoped to other clients.

Errors you may see during integration:

| Surfaced message | Cause |
|------------------|-------|
| `Template is not activated in this environment` | The template exists but no version has been published in the current environment. |
| `Template is disabled in this environment` | The template was explicitly disabled. |
| `Template authentication required` | The template requires authenticated access and the request is anonymous. |
| `Client is not allowed to use this template` | The template is scoped to specific clients and your token is not on the list. |
| `Template runtime artifact not found` | The active version's artifact is missing. Re-publish the template via the Console. |

These surface as `API-002` (or `API-101` / `API-102` for the auth-related
ones) under the standard error envelope.


## 6. Discovering available templates

There is **no public `GET /api/v1/templates` endpoint** at this time. The
list of templates available to your environment is managed in the gPdf
Console and coordinated out-of-band:

- The team that publishes templates owns the `template_id` and field-schema decisions.
- Callers receive the contract (template ID + field list + types) through internal documentation, the Console UI, or a shared design system.

This is a deliberate choice for v1. A discovery endpoint may be added later
and will be listed in the [api-reference.en.md changelog](./api-reference.en.md#7-changelog)
when it is.


## 7. Built-in example templates

The repository ships three example templates. Whether they are **callable in
your environment** depends on whether they have been published there.
Confirm with the team that owns the Console.

### 7.1 `invoice`

Top-level fields:

- `invoice_number`, `date_of_issue`, `date_due`, `billing_period`
- `issuer_name`, `issuer_address`, `issuer_phone`, `issuer_email`, `issuer_tax_id`
- `bill_to_name`, `bill_to_address`, `bill_to_email`
- `ship_to_name`, `ship_to_address`
- `amount_due_summary`, `payment_note`
- `subtotal`, `total`, `amount_due`
- `items` (array)

`items[]` fields:

- `description`, `qty`, `unit_price`, `amount`

Minimum example:

```json
{
  "template_id": "invoice",
  "data": [
    {
      "invoice_number": "INV-001",
      "date_of_issue": "2026-03-11",
      "date_due": "2026-04-10",
      "issuer_name": "Acme Cloud Inc.",
      "issuer_address": "88 Harbor Rd, Long Beach, CA",
      "bill_to_name": "Receiver Inc.",
      "bill_to_address": "123 Main St, Los Angeles, CA",
      "amount_due_summary": "Amount due on receipt",
      "subtotal": "$100.00",
      "total": "$100.00",
      "amount_due": "$100.00",
      "items": [
        {
          "description": "Service A",
          "qty": 1,
          "unit_price": "$100.00",
          "amount": "$100.00"
        }
      ]
    }
  ]
}
```

### 7.2 `packing_list`

Top-level fields (note the dotted paths):

- `shipment.number`, `shipment.date`, `shipment.vessel`, `shipment.port_of_loading`, `shipment.port_of_discharge`
- `shipper.name`, `shipper.address`, `shipper.phone`
- `consignee.name`, `consignee.address`, `consignee.phone`
- `notes`
- `items` (array)

`items[]` fields:

- `item_no`, `description`, `quantity`, `unit`, `gross_weight`, `net_weight`

Minimum example:

```json
{
  "template_id": "packing_list",
  "data": [
    {
      "shipment": {
        "number": "PL-001",
        "date": "2026-03-11"
      },
      "shipper": {
        "name": "Acme Logistics",
        "address": "Shenzhen"
      },
      "consignee": {
        "name": "Receiver Inc.",
        "address": "Los Angeles"
      },
      "items": [
        {
          "item_no": "1",
          "description": "Box A",
          "quantity": 10,
          "unit": "CTN",
          "gross_weight": 20.0,
          "net_weight": 18.0
        }
      ]
    }
  ]
}
```

### 7.3 `shipping_label`

Top-level fields:

- `recipient_name`, `recipient_address`, `recipient_company`, `recipient_phone`
- `sender_name`, `sender_address`
- `tracking_number`, `weight`, `pieces`, `special_requirements`

Minimum example:

```json
{
  "template_id": "shipping_label",
  "data": [
    {
      "recipient_name": "John Doe",
      "recipient_address": "123 Main St, Los Angeles, CA",
      "sender_name": "Acme Warehouse",
      "sender_address": "88 Harbor Rd, Long Beach, CA",
      "tracking_number": "TRK1234567890"
    }
  ]
}
```


## 8. Validation rules

Templates run client data through these checks:

### 8.1 Required fields

A field declared `required = true` returns `API-002` if missing. For `string`
fields, empty or whitespace-only is treated as missing.

### 8.2 Field types

The supported field types are:

- `string`
- `number`
- `boolean`
- `array`

Other JSON shapes (e.g. nested objects without dotted-path schema
declarations) are not allowed unless the template declares them.

### 8.3 Array items

If a field is `array` and the template declares an `items` schema, every
array element is checked against that schema.

For example:

- `invoice.items[].qty` must be a `number`.
- `packing_list.items[].gross_weight` and `net_weight` must be `number`.

### 8.4 Dotted paths

Templates can read nested object paths:

- `shipment.number`
- `shipper.name`
- `consignee.address`

What templates do **not** support today:

- Numeric array indices in path syntax (e.g. `items[0].qty`).
- Expressions in paths.

### 8.5 Unknown fields

Currently, unknown fields in `data` items do not raise errors. They are
silently dropped.

This is intentional for v1 (forward compatibility), but **callers should not
rely on it**:

- A misspelled field name will silently render an empty value.
- If you set a field but the rendered PDF is unchanged, the most likely cause is a typo or a wrong path.


## 9. Errors

The error envelope is identical to the JSON Render API. See
[api-reference.en.md §6.1](./api-reference.en.md#61-error-codes) for the full
table. Common Template-Render-specific triggers:

| Trigger | Code | Typical message |
|---------|------|-----------------|
| `data` is not an array | `API-001` | `invalid type` |
| `data` is an empty array | `API-002` | `Template render data must contain at least one item` |
| `data` exceeds `10` items | `API-002` | `Template render data item count ... exceeds max 10` |
| `data[n]` is not an object | `API-002` | `Template render data[n] must be an object` |
| Required field missing | `API-002` | `Missing required field <name>` |
| Field type mismatch | `API-002` | `Field type mismatch` |
| Per-item global differs | `API-002` | `produced different document globals` |
| Page count exceeds policy | `API-004` | `page count` / `max_pages_per_request` |
| Template not active or unauthorised | `API-002` / `API-101` / `API-102` | See §5. |

Sample error response:

```json
{
  "error": true,
  "code": "API-002",
  "message": "Missing required field invoice_number",
  "req_id": "7f7d2f5a-4cb0-4c4e-b6ef-8f6d3e0b1fd8"
}
```


## 10. Integration playbook

When integrating a new template, follow this sequence:

1. **Smoke-test with a tiny template first.** `shipping_label` has only a handful of required fields. If even that fails, the issue is auth or environment, not the template.
2. **Add output controls next.** Verify `output.mode = "file"` and a custom `file_name` produce the expected `Content-Disposition`.
3. **Then integrate templates with arrays.** `invoice.items[]` and `packing_list.items[]` exercise per-item type checks.
4. **Cross-environment.** Once it works in test, repeat the smoke test in production. Tokens, templates, and quotas are all different.
5. **Lock the contract.** Pin three things in your team's design doc:
   - The exact `template_id` you use in each environment.
   - The list of fields and their types.
   - Your `output.mode` policy and file-name pattern.

These three lines are the entire contract between systems. Everything else is
implementation detail on either side.
