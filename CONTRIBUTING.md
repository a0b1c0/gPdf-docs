# Contributing

Thanks for helping make the gPdf docs better.

## Filing issues

- **API behaviour / wrong response / missing detail** → Issues in **this** repo.
- **Website (gpdf.com) bugs, blog typos, marketing-site issues** → Contact us via [the contact form on gpdf.com](https://gpdf.com/contact/) (marketing-site repo is private; we triage internally).
- **Render-engine bugs (wrong PDF output for valid input)** → Contact us via [the contact form on gpdf.com](https://gpdf.com/contact/) (the engine source is private; we triage internally).

When filing an API issue, please include:

1. The endpoint you called (`/api/v1/pdf/render` / `/api/v1/template-render` / `/api/v1/e-invoice/render`).
2. The `req_id` header from the response (gives us a direct lookup into our logs).
3. A minimal `DocumentRequest` that reproduces — strip business data, keep schema shape.
4. What you expected vs what you got.

## Pull requests

Docs PRs welcome for typos, clarifications, and missing examples. Please keep the **schema descriptions accurate** — every change is reviewed against the live engine's contract.

The `examples/*.json` files are **executable** — every payload here renders successfully against `api.gpdf.com`. If you propose a new example, please verify it runs first.

## Source of truth

The API reference markdown in this repo is auto-synced from a private upstream (the marketing-site source) by a GitHub Action — typically within ~30 seconds of an upstream merge. PRs filed against the rendered files in this repo are welcome for typo fixes, broken-link repairs and additional examples; substantive API-behaviour changes should be raised as an issue first so we can route them to the upstream owners.
