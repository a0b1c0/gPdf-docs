> [!NOTE]
> This file is generated from `doc/contracts/source/gpdf-api-contract.md`. Do not edit it directly.

# E-Invoice API

This page remains for existing documentation links. The public E-Invoice API contract is consolidated into the “E-Invoice API” section of `api-reference.en.md`.

Public e-invoice endpoints:

| Endpoint | Description |
| --- | --- |
| `POST /api/v1/e-invoice/render` | Generate an e-invoice PDF or asynchronous job. |
| `POST /api/v1/e-invoice/validate` | Validate only. |
| `GET /api/v1/e-invoice/capabilities` | Return public capabilities and limits. |
| `GET /api/v1/e-invoice/artifacts/{job_id}/{artifact}` | Download asynchronous job artifacts. |
