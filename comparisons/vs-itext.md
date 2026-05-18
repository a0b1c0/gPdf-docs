# gPdf vs iText for logistics labels

**Compared:** [iText](https://itextpdf.com/) — PDF SDK (Java / .NET)  
**Published:** 2026-05-17  
**Author:** gPdf Engineering

> iText is the industry-standard PDF SDK; gPdf is a hosted PDF generation service. For 4×6 thermal labels at 100K → 10M pages/month, we compare usage cost, integration difficulty, maintenance effort, and edge deployment.

## Verdict

iText is a mature, well-licensed PDF SDK — paying for it is fair. The question logistics teams should ask is what they're paying for. iText sells you the SDK; you build, deploy, scale, and maintain the label-generation service around it. gPdf sells you the service: POST a label JSON, get a scannable thermal label PDF in ~4 ms at the edge, with no JVM, no warm pools, and no carrier-spec patch nights.

## Comparison table

| Axis | gPdf | iText | Advantage |
|---|---|---|---|
| First production-ready 4×6 thermal label | ~5 minutes — copy the JSON sample, curl it, scan the PDF on a Zebra printer. | 2–5 engineering days — add the Maven/NuGet dependency, write the Java class, configure Barcode128, tune fonts, test scan-rate, deploy. | ★ gPdf |
| Integration shape | HTTPS POST from any language (Node, Python, Go, .NET, Ruby, PHP, Java). | Java or .NET only; forces a JVM/CLR in your runtime stack. | ★ gPdf |
| Render p50 (1× 4×6 label) | ~4 ms at the nearest Cloudflare PoP (300+ globally). | ~2 ms steady-state in-JVM, plus 100–250 ms network if the JVM lives in a different region from the warehouse. | ★ gPdf |
| Monthly cost at 100K labels | $5 (Basic tier; per-page rate $0.050/1K). | iText commercial license + servers + on-call; license alone is typically mid-5-figure USD/year, prorated to month. | ★ gPdf |
| Monthly cost at 1M labels | $45 ($50 base × 10% volume discount; per-page rate $0.045/1K). | Same license + bigger HA footprint + more engineer-hours per month. | ★ gPdf |
| Monthly cost at 10M labels | $400 ($500 base × 20% volume discount; per-page rate $0.040/1K). | Multi-region HA + on-call rotation + carrier-spec maintenance grows with volume. | ★ gPdf |
| When a carrier changes spec (UPS SSCC, FedEx tracking, SF Express check digit) | Edit the JSON template; runtime untouched. Turnaround: hours. | Edit Java → unit test → integration test → build JAR → deploy staging → roll prod across regions. Turnaround: 2–10 engineering days. | ★ gPdf |
| GS1-128 with Application Identifiers | Single `barcode` element with `format: "gs1_128"` and the AI string in `content`. | Barcode128 primitive plus manual AI encoding and FNC1 wiring in Java. | ★ gPdf |
| Visual template editor | editor.gpdf.com — designs the same JSON that runs in production. Public, included. | iText DITO — part of the iText commercial product ecosystem. | — |
| Offline / air-gapped deployment | Available via enterprise private deployment (separate engagement). | Native — iText runs in any JVM, no network needed. | ★ iText |
| Deep PDF manipulation (signing, forms, splitting, editing) | Not in scope — gPdf renders new PDFs from JSON. | Strong — iText's home turf, where the SDK genuinely earns its license. | ★ iText |
| Maturity | Public since 2025. | Since 1998. | ★ iText |

**Notes:**

- *Render p50 (1× 4×6 label)* — iText's in-process render is fast; the cost is hosting the JVM. gPdf renders at the edge PoP nearest the warehouse, so the network hop is the smallest part of the budget.
- *Monthly cost at 10M labels* — Full TCO comparison (license, infra, engineer-time, maintenance) lives in the long-form analysis linked at the bottom.

## When to pick gPdf

- You generate logistics labels at any volume (5K/day to 5M/day) and PDF generation is infrastructure for your business, not the business itself.
- Your stack is multi-language (Node, Python, Go, .NET, Ruby) and you don't want to operate a Java service just to render labels.
- You want to absorb carrier-spec changes as JSON template updates, not JVM redeploys.
- Your warehouses are global and you don't want to operate label rendering in four AWS regions.
- You want predictable per-page pricing with a published volume-discount ladder, not an annual license renegotiation.

## When to pick iText

- You manipulate existing PDFs — signing, form filling, splitting, deep editing — not just rendering new ones.
- Your stack is already Java/.NET-first and adding an outbound HTTP dependency feels like a regression.
- You operate in air-gapped or strictly offline environments where outbound HTTP is forbidden.
- PDF tooling is core to your product (you are a PDF vendor, e-signature platform, or legal-tech archive) and owning the SDK is the correct level of control.
- You need niche PDF spec coverage (XFA forms, advanced digital-signature handlers, attestation profiles) that gPdf doesn't ship.

---

## Deep dive

## SDK or infrastructure?

iText is a brilliant PDF SDK. Buying it is fair — paying for mature software is fair. The question logistics teams should actually ask is **what they're paying for**.

With iText, you pay for the SDK. Around the SDK you also build:

- A label-rendering service (Java, hosted by you, on servers you scale).
- Font handling, barcode regression tests, PDF/A configuration.
- Deployment pipeline, monitoring, on-call rotation, redundancy.
- Carrier-spec change tracking — every UPS, FedEx, DHL, SF Express tweak becomes a Java diff plus a JVM redeploy.

With gPdf, you pay for **the label being generated**. The renderer, the edge distribution, the carrier-spec adaptability layer, the visual editor, and the regression testing are part of the service.

## First 4×6 label: 5 minutes vs 5 days

A typical "from zero to a thermal label that actually scans on a Zebra ZT411" measurement:

**iText path** — Java; simplified, real code adds the build setup, font registration, scan-rate test harness, and a CI pipeline that runs it:

```java
PdfWriter writer = new PdfWriter("label.pdf");
PdfDocument pdf = new PdfDocument(writer);
PageSize labelSize = new PageSize(288, 432);     // 4×6 in @ 72 dpi
Document doc = new Document(pdf, labelSize);
// Address block, sender block, carton ID, service code…
// (15–25 more lines positioning text and configuring Barcode128 with
// GS1 Application Identifiers, fonts, FNC1 framing, then a JUnit test
// that loads the PDF and validates the barcode renders at 203 dpi)
Barcode128 code = new Barcode128(pdf);
code.setCode("(01)00012345678905(21)SN12345");
code.setCodeType(Barcode128.CODE128);
// … position, sizing, human-readable interpretation line …
doc.close();
```

Typical first-success time (from `mvn init` to a label that scans cleanly): **2–5 engineering days**.

**gPdf path** — any language; the example below is curl:

```bash
curl -X POST https://api.gpdf.com/api/v1/pdf/render \
  -H "Authorization: Bearer $GPDF_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "pages": [{
      "size": "label_4_6_in",
      "elements": [
        { "type": "text", "x": 4, "y": 12,
          "content": "Acme Distribution Centre\n1200 Logistics Pkwy\nMemphis TN 38116" },
        { "type": "barcode", "format": "gs1_128",
          "content": "(01)00012345678905(21)SN12345",
          "x": 4, "y": 60, "width": 92, "height": 22,
          "barcode_text": { "enabled": true, "position": "bottom" }
        }
      ]
    }]
  }' -o label.pdf
```

Typical first-success time: **about 5 minutes**, including reading the JSON sample and printing the PDF on the same Zebra ZT411.

The gap isn't engineering talent. It's where the work is. With iText, the work is on your team. With gPdf, it's productised away.

## When a carrier changes its spec

Logistics is the rare domain where the document spec changes from outside your team. UPS revises an SSCC encoding rule. SF Express adds a check digit. FedEx publishes a new HAZMAT block layout. Whatever rendering stack you picked has to absorb the change.

**With iText**: a developer reads the carrier bulletin, modifies the Java code, runs unit + integration tests, builds the JAR, deploys to staging, deploys to production, rolls forward across regions. Typical turnaround: **2–10 engineering days**. During the rollout window, warehouses still print the old format.

**With gPdf**: edit the template JSON (in the visual editor at editor.gpdf.com, or in code) and push the template config. The renderer itself doesn't move; only the spec it's rendering does. Typical turnaround: **hours**. If the carrier change is in a barcode format gPdf already supports, you don't touch the renderer side at all.

## Usage cost, with the volume discount tiers public

Pricing transparency, list prices for the volumes most logistics teams ask about:

| Monthly volume       | gPdf list price                | Effective per-1K labels |
|---------------------:|-------------------------------:|------------------------:|
| 100K labels          | $5                             | $0.050                  |
| 1M labels            | $45 (10% volume discount)      | $0.045                  |
| 10M labels           | $400 (20% volume discount)     | $0.040                  |
| 100M+ labels         | Contact for enterprise pricing | —                       |

The list-price column is the easy part. The harder part is **everything else on the bill**: iText license amortisation, server/HA footprint, the engineer-hours you spend on the label service every month, and the maintenance overhead that grows with the number of carrier integrations.

That full TCO breakdown — including engineer-month estimates by volume tier, infrastructure cost ranges, and the curve of how iText cost rises while gPdf per-page cost falls as you scale — lives in the long-form analysis:

→ [Shipping label TCO: iText vs gPdf at 100K → 100M pages/month](/blog/itext-vs-gpdf-for-shipping-labels-tco-2026/)

## When iText is still the right answer

A comparison that pretends the competitor never wins is marketing fluff. iText remains the better pick when:

- **You manipulate PDFs, not just render them.** Signing, form filling, splitting, page-level edits — gPdf renders new PDFs from JSON and stays out of those workflows.
- **Your stack is Java/.NET first.** If the rest of your services run on the JVM and adding an outbound HTTP dependency feels like a regression, iText keeps everything in-process.
- **You run air-gapped or strictly offline.** Outbound HTTPS is the wrong shape for some warehouse-floor or government deployments. iText runs anywhere a JVM does.
- **PDF tooling is your product.** If you are a PDF vendor, an e-signature platform, or a legal-tech archive, owning the SDK is the right level of control. gPdf is built for teams whose product is logistics, invoicing, or commerce — not PDFs themselves.
- **You need niche PDF spec coverage** (XFA forms, advanced digital-signature handlers, attestation profiles) that gPdf does not ship.

For *"I need a scannable label on a parcel and I have a million parcels a month"*, gPdf is the lower-friction path. For *"I need to manipulate an existing legal PDF inside my Java backend"*, iText is.

## Migration shape

For teams moving label rendering from iText to gPdf, the diff is roughly:

```diff
- // Before: a Java label-rendering service
- PdfWriter writer = new PdfWriter(out);
- PdfDocument pdf = new PdfDocument(writer);
- Document doc = new Document(pdf, new PageSize(288, 432));
- // 20–40 lines wiring fonts, positions, Barcode128 with GS1 AIs
- doc.close();
+ // After: HTTPS POST the structured DocumentRequest from any language
+ const res = await fetch('https://api.gpdf.com/api/v1/pdf/render', {
+   method: 'POST',
+   headers: { Authorization: `Bearer ${KEY}`, 'Content-Type': 'application/json' },
+   body: JSON.stringify(labelDocumentRequest),
+ });
+ const pdf = Buffer.from(await res.arrayBuffer());
```

Once the cut is done, the Java label service collapses to a single fetch call from whatever language already orchestrates orders. The JVM disappears from the label path; carrier-spec changes stop being a deploy event; the on-call rotation stops getting paged for label-rendering OOMs.

## See also

- [Shipping label TCO: iText vs gPdf at 100K → 100M pages/month](/blog/itext-vs-gpdf-for-shipping-labels-tco-2026/) — long-form cost math, engineer-months, infra ranges.
- [Shipping labels use case](/use-cases/shipping-labels/) — sample payloads, p99 numbers, Black Friday math.
- [JSON Render API reference](/docs/api-reference/) — endpoints, request shape, security model.
- [GS1-128 barcodes at 0.1 mm precision in JSON](/blog/gs1-128-barcodes-at-01mm-precision/) — barcode geometry deep-dive.
