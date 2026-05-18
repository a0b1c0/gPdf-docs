# gPdf Template Authoring Guide

> Audience: template maintainers, **not** API callers.
>
> If you call `POST /api/v1/template-render`, you do not need this document.
> You need `template-api.en.md`.
>
> This guide covers what template authors must know to design, validate,
> and publish templates that the public API will accept. It is intentionally
> kept out of the public docs site.
>
> Last updated: 2026-05-08.

---

## 1. Lifecycle

A template moves through three stages:

1. **Source template** — the JSON file the author writes. This is where layout, schema, and binding syntax live.
2. **Compile** — the source is linted, type-checked against schema declarations, and converted into a runtime artifact.
3. **Publish** — the runtime artifact is associated with a stable `template_id` and activated in an environment (test or production).

Callers only see the third stage. They send `{ template_id, data }`; the
runtime resolves the active artifact and binds the data.

---

## 2. Schema

Each template declares a schema describing the fields callers can send. The
schema controls:

- Which fields are recognised.
- Which fields are required.
- The expected JSON type for each field (`string`, `number`, `boolean`, `array`).
- For arrays, the structure of each item.

Schema constraints today:

- Field types are limited to `string`, `number`, `boolean`, `array`. Free-form objects must be modelled as a set of dotted-path fields.
- Dotted paths are supported (`shipment.number`).
- Numeric indices in paths (`items[0].qty`) are **not** supported.

Example schema fragment:

```json
{
  "fields": [
    { "name": "invoice_number",       "type": "string",  "required": true },
    { "name": "date_of_issue",        "type": "string",  "required": true },
    { "name": "issuer_name",          "type": "string",  "required": true },
    { "name": "issuer_tax_id",        "type": "string",  "required": false },
    { "name": "subtotal",             "type": "string",  "required": true },
    { "name": "items",                "type": "array",   "required": true,
      "items": [
        { "name": "description", "type": "string", "required": true },
        { "name": "qty",         "type": "number", "required": true },
        { "name": "unit_price",  "type": "string", "required": true },
        { "name": "amount",      "type": "string", "required": true }
      ]
    }
  ]
}
```

---

## 3. Binding syntax

The layout file references schema fields via three constructs.

### 3.1 `{{path}}`

Plain field substitution:

```text
"content": "{{invoice_number}}"
```

Mixed-string substitution:

```text
"content": "Invoice #{{invoice_number}}"
```

### 3.2 `_if`

Object-level conditional rendering:

```json
{
  "type": "text",
  "content": "{{issuer_tax_id}}",
  "_if": "{{issuer_tax_id}}"
}
```

The object renders only when the bound value is non-empty.

### 3.3 `{{#each items}}`

Expands an array into table rows:

```text
"rows": "{{#each items}}"
```

Given table columns:

- `description`, `qty`, `unit_price`, `amount`

A caller's `items` array becomes one row per element. Element field names
must match `columns[].key`.

Full example:

```json
{
  "type": "table",
  "x": 12,
  "y": 80,
  "width": 180,
  "columns": [
    { "key": "description", "header": "Description", "width": { "mode": "auto" } },
    { "key": "qty",         "header": "Qty",         "width": { "mode": "fixed", "value": 18 } },
    { "key": "unit_price",  "header": "Unit Price",  "width": { "mode": "fixed", "value": 30 } },
    { "key": "amount",      "header": "Amount",      "width": { "mode": "fixed", "value": 30 } }
  ],
  "rows": "{{#each items}}"
}
```

### 3.4 `pages[]._each`

Repeats a page per array element:

```json
{
  "pages": [
    {
      "_each": "labels",
      "size": "label_100_150",
      "elements": [
        { "type": "text", "x": 10, "y": 18, "content": "{{name}}" },
        { "type": "text", "x": 10, "y": 28, "content": "{{address}}" }
      ]
    }
  ]
}
```

Constraints:

- The object carrying `_each` must be the **only** object in its containing array (no static pages mixed with `_each` pages).
- `_each` must reference a schema field of type `array`.
- Inside the `_each` body, fields of the array element are referenced directly (`{{name}}`) or via the explicit `_item` namespace (`{{_item.name}}`).

---

## 4. Compile-time lint

The compiler runs lint checks before producing a runtime artifact. Two
severities:

### 4.1 Errors that block compile

These prevent the template from publishing:

- Empty, illegal, duplicate, or reserved schema field names.
- Path conflicts at the same schema layer (e.g. defining both `shipment` and `shipment.number`).
- Non-`array` fields declaring an `items` schema.
- `default_value` whose JSON type does not match the declared field type.
- `{{path}}`, `_if`, or `{{#each path}}` referencing a path not declared in the schema.
- `{{#each path}}` referencing a non-`array` field.
- A table with `rows: "{{#each items}}"` whose schema does not declare `items.items[]`, or where any `columns[].key` is not in the array element schema.
- Numeric or boolean position fields bound to wrong-typed schema fields (e.g. `x: "{{name}}"` where `name` is `string`).
- `_each` placed somewhere other than the unique template object inside an array, or `_each` whose value is not a string path.

### 4.2 Warnings (do not block, but should be fixed)

- Schema fields declared but never referenced by the layout.
- Array-typed fields used as plain placeholders.
- Mustache constructs the runtime does not execute (e.g. `{{#if ...}}` / `{{/if}}`).
- Excessively deep field paths that increase the chance of caller typos.
- Arrays nested inside array items (the binder does not recursively validate inner array schemas today).
- A page using `_each` while also containing a table with `rows: "{{#each ...}}"` (the runtime does not recursively expand both axes).

---

## 5. Validation endpoint

`POST /api/v1/template/validate` validates a source template before publish.

On success it returns:

| Field | Notes |
|-------|-------|
| `runtime` | The compiled runtime artifact. |
| `warnings` | Non-blocking lint warnings. |
| `preview_data` | Synthetic data the validator generated to test bindings. |
| `preview_document` | Rendered preview using the synthetic data. |

If lint errors are present, compilation fails outright. If only warnings
exist, the template can compile and preview, but a human reviewer should
look at the warnings before activating it in production.

This endpoint is for the template authoring pipeline. It is not advertised
to API callers.

---

## 6. Currently unsupported features

These are intentionally **not** supported in the v1 template engine. Layout
files using them will fail compile:

- `if / else` branches.
- Comparison expressions (`==`, `>`, `<`).
- Arithmetic expressions.
- Aggregation helpers (e.g. `sum(items.amount)`).
- Field formatters (date, number, currency formatting).
- Computed fields.
- Custom filter functions.
- Designer-specific schema extensions.
- Domain-specific block DSLs.

These are tracked for future template-engine versions. Until they are
released, do not rely on them in source templates.

---

## 7. Publishing checklist

Before activating a template in production:

- [ ] Source template compiles with zero errors.
- [ ] All warnings have been reviewed and either resolved or explicitly accepted.
- [ ] The schema field list is documented for callers (matching what `template-api.en.md §7` would show).
- [ ] At least one round-trip test has been done in the test environment using the same `template_id`.
- [ ] The runtime artifact has been published and activated in production.
- [ ] Caller teams have been notified of the new `template_id` and field schema.

---

## 8. Style guidance for caller-facing field names

When you choose schema field names, remember that callers will see them in
their integration code forever. A few conventions keep things consistent
across templates:

- **snake_case** for field names. `invoice_number`, not `invoiceNumber` or `InvoiceNumber`.
- **Singular for scalars, plural for arrays.** `item` is a single item; `items` is the array.
- **Prefix related fields by domain.** `issuer_name`, `issuer_address`, `issuer_email` — not a free-form `issuer` object.
- **Avoid abbreviations the caller has to memorise.** `tracking_number` reads better than `trk_no`.
- **Currency and date formats are caller responsibility.** Templates render strings as-is. If you want `$1,000.00`, the caller sends `"$1,000.00"`. Do not invent format directives in the schema.
- **Arrays of scalars are uncommon.** Most arrays should be arrays of objects with a clear `items` schema.

These conventions are not enforced by the compiler. They are how the
existing built-in templates (`invoice`, `packing_list`, `shipping_label`)
read, and following them keeps the caller-facing surface coherent across
the template catalog.
