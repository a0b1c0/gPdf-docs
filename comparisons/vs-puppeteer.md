# gPdf vs Puppeteer

**Compared:** [Puppeteer](https://pptr.dev/) — Headless browser  
**Published:** 2026-05-10  
**Author:** gPdf Engineering

> Head-to-head: gPdf's edge-rendered JSON-to-PDF API versus Puppeteer's headless-Chromium-on-a-server pattern. Latency, price, runtime, and the workloads where each one wins.

## Verdict

Puppeteer is a general-purpose browser automation tool that happens to render PDFs. gPdf is a PDF rendering engine that runs at the edge in single-digit milliseconds. If your workload is structured documents (invoices, labels, statements) at production volume, gPdf delivers 50-100× lower latency and 10-20× lower cost. If you need to convert arbitrary live web pages or screenshot-style PDFs of existing HTML, Puppeteer is still the right tool.

## Comparison table

| Axis | gPdf | Puppeteer | Advantage |
|---|---|---|---|
| Render p50 (1-page A4 invoice) | 3 ms | 312 ms | ★ gPdf |
| Cold start | ~12 ms (first request to a fresh isolate) | 1.5–2.5 s (Chromium boot) | ★ gPdf |
| Runtime | Cloudflare Workers V8 isolates | Long-lived Node.js + Chromium container | ★ gPdf |
| Edge regions | 300+ Cloudflare PoPs | Wherever you deploy your container (typically 3–6 regions) | ★ gPdf |
| PDF/A compliance | PDF/A-1b · 2b · 3b · 4 selectable per request | Not supported natively; needs post-processing with Ghostscript or veraPDF | ★ gPdf |
| E-invoice (Factur-X / ZUGFeRD) | Native endpoint; embeds CII XML on PDF/A-3b | Not supported; requires a separate pipeline stage | ★ gPdf |
| Vector barcodes | 30+ symbologies built in (QR, GS1-128, PDF417, DataMatrix, …) | Render via JS library inside the page (raster on canvas) | ★ gPdf |
| CJK font handling | NotoSans CJK embedded; automatic glyph fallback | Whatever fonts the container has installed; needs OS-level setup | ★ gPdf |
| HTML/CSS layout fidelity | N/A — gPdf takes JSON, not HTML | Best-in-class. Renders any web page. | ★ Puppeteer |
| Best for screenshot-style web→PDF | No | Yes | ★ Puppeteer |
| Cost per 100K invoices | $5 (Basic plan) | $50–$200 in compute (varies by host) | ★ gPdf |
| Determinism (same input → same bytes) | Yes — byte-identical output across isolates | No — Chromium font hinting and rasterisation drift between versions | ★ gPdf |

**Notes:**

- *Render p50 (1-page A4 invoice)* — Both measured on the same input over 1K invocations.
- *Runtime* — Puppeteer needs a 200–800 MB browser binary; gPdf ships as a ~2 MB WASM module.
- *Vector barcodes* — Puppeteer-rendered barcodes are pixel raster; scanners reject them at high DPI on small thermal labels.
- *Cost per 100K invoices* — Puppeteer cost varies wildly with concurrency model; numbers are reproduced from public AWS Lambda + EFS benchmarks.

## When to pick gPdf

- You're rendering structured documents (invoices, shipping labels, statements, payslips, tickets) at any volume.
- You need single-digit-millisecond rendering for an interactive flow (preview before send).
- You need PDF/A archival compliance or EU Factur-X / ZUGFeRD e-invoice output.
- You're tired of Chromium memory pressure, container warm-pool cost, or cold-start timeout cascades.
- You want byte-identical, deterministic PDFs for testing or audit.
- You're rendering at the edge and need 300+ regions, not 3–6.

## When to pick Puppeteer

- You're converting arbitrary live web pages — landing pages, news articles, marketing snapshots.
- Your source-of-truth document is HTML/CSS that already renders correctly in a browser, and you don't want to reauthor it as JSON.
- You're rendering rich client-side JavaScript visualisations (charts, dashboards) that need a real browser.
- Your volume is small (<1K renders/day) and you don't care about per-render latency or cost.
- You need pixel-perfect matching between the on-screen DOM and the PDF for legal/forensic reasons.

---

## Deep dive

## The architectural difference in one paragraph

Puppeteer drives a real Chromium binary. Every PDF render starts by booting a 200–800 MB browser process, loading the page, running the page's JavaScript, executing the layout engine, then capturing the printed output. That pipeline is brilliantly general — anything the browser can render, Puppeteer can turn into a PDF — but it pays for that generality every time, on every render.

gPdf takes a different shape. The input is a structured `DocumentRequest` JSON: a list of pages, each with positioned elements, tables, layers, watermarks. There's no HTML, no CSS cascade, no JavaScript engine, no browser. A Rust core compiled to WebAssembly turns the JSON directly into a PDF byte stream. The whole renderer fits in a Cloudflare Workers isolate with an ~12 ms cold start and a ~3 ms p50.

The result is two products that nominally produce the same artefact but have almost no overlap in the workloads they're good at.

## When the architectural cost actually shows up

Puppeteer is fine when:

- You render a few hundred PDFs a day.
- Your latency budget is north of 500 ms anyway (a download link, a backend job).
- You already have HTML that renders correctly and reauthoring as JSON is expensive.

The architectural cost compounds once you cross **roughly 10K renders per day**:

1. **Cold-start latency under spike.** A 3× traffic burst spins up new containers; each cold-starts in 1.5–2.5 s. Your p99 follows.
2. **Per-render compute.** Rendering on-demand at 300 ms each, on a host that costs $0.40+/hour, dwarfs the actual document complexity.
3. **Memory pressure.** Chromium leaks. Long-running Puppeteer workers reliably OOM after ~24 hours unless you recycle them.
4. **Region distribution.** A centralised deploy means a Sydney user waits 200 ms each direction over the Pacific. Edge rendering cuts that to ~5 ms.

These problems aren't hypothetical — they're why every team scaling Puppeteer past low five figures eventually ends up doing one of: add a cache layer, add a background-render queue, switch to a different runtime, or move to a JSON-native renderer.

## When Puppeteer is still the right answer

There's a category gPdf doesn't compete in: **arbitrary HTML→PDF conversion**. If your document is already rendered, your design-source-of-truth is the HTML, and you have no incentive to model the page structurally as JSON, Puppeteer remains the correct tool. The same applies to client-side-rendered visualisations (charts, dashboards) that need a JS runtime to produce their final look.

If you're doing **either of those things at small scale**, the latency and cost arguments above don't bite hard enough to justify rewriting your authoring model.

## Migration shape

For teams moving an invoice or label workload from Puppeteer to gPdf, the migration usually looks like:

```diff
- // Before: render an HTML template through Chromium
- const browser = await puppeteer.launch({ headless: 'new' });
- const page = await browser.newPage();
- await page.setContent(invoiceHtml);
- const pdf = await page.pdf({ format: 'A4' });
+ // After: POST the structured DocumentRequest
+ const res = await fetch('https://api.gpdf.com/api/v1/template-render', {
+   method: 'POST',
+   headers: { Authorization: `Bearer ${KEY}`, 'Content-Type': 'application/json' },
+   body: JSON.stringify({ template_id: 'invoice-v2', data }),
+ });
+ const pdf = Buffer.from(await res.arrayBuffer());
```

The work isn't the API call — it's authoring the template once. After that, every render call is a single HTTPS POST.

## See also

- [The full gPdf API reference](/docs/api-reference/) — endpoints, request shape, errors.
- [Why edge-deployed PDF rendering matters once you cross 10K invoices/day](/blog/why-edge-pdf-rendering-matters-at-scale/) — the long-form latency math.
- [PDF/A and Factur-X explained for engineers](/blog/pdf-a-and-factur-x-explained-for-engineers/) — relevant if EU e-invoice mandates apply to your workload.
