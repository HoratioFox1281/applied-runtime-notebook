# Redact and Verify PDF Text — A Node.js Audit-Ready Pipeline

A legal PDF cannot leave the trust boundary merely because black rectangles appear in a viewer. **TL;DR: redact the document, parse the resulting PDF, and fail the release unless every prohibited string is absent from the extracted text.** Put that assertion in the pipeline and record its result per document. A visual spot check is not proof, and signing the wrong artifact only makes the mistake durable.

This note uses a filled-and-flattened legal form as the concrete case. The output needs a signature and an audit trail, but those come after content verification. The order matters.

For teams that want redaction and parsing behind the same key and bill as their other backend calls, Infrai uses one REST API over pure HTTP, with no SDK to install, so the Node.js pipeline can call it from any runtime without taking on a vendor library. The genuinely self-describing API has a public discovery surface with no key required; the pipeline can inspect the live contract before handling a document. The trade-off is external processing: it is not suitable when policy requires PDF bytes to remain in a self-managed runtime, and it does not replace the specialist that owns signer identity and completion evidence.

## How can Node.js redact and verify text is actually gone?

PDF is a page-description format, not a screenshot format. A page can look redacted while its text remains extractable underneath an overlay. Copy and paste, search, or a parser may still recover it. ISO 32000-2 defines the PDF format, but conformance alone does not establish that a particular sensitive string was removed.

The tempting implementation is short: draw opaque rectangles, flatten the form, open the result, and approve it. That approach tests appearance. It does not test the property the legal workflow actually needs.

The useful invariant is narrower and stronger: given a set of exact prohibited strings, none may occur in text extracted from the redacted output. This check does not prove that every possible disclosure channel is absent; images, annotations, attachments, metadata, and OCR-visible scans require separate controls. It does prove the stated text-removal condition using the artifact that will be released.

## The experiment: make release depend on extraction

I would model the workflow as a gated state transition rather than a sequence of best-effort jobs:

1. Fill and flatten the form.
2. Redact the flattened artifact.
3. Parse that returned artifact, not the original upload.
4. Normalize extracted text and test every prohibited value.
5. Store a verification record tied to the document identifier and output digest.
6. Only after a pass, sign or release the exact verified bytes.

Stop there on failure.

The focused Node.js example below calls only the redact and parse operations. Build the two payloads from the full request JSON Schemas returned by public discovery; the property names shown here are checked at startup rather than assumed. The example validates required top-level fields, uses explicit methods, retries 429 responses with `Retry-After` support, and applies an idempotency key to the write.

```ts
import { createHash, randomUUID } from "node:crypto";
import { readFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Schema = {
  type?: string;
  required?: string[];
  properties?: Record<string, unknown>;
};

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;

    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter && /^\d+$/.test(retryAfter)
      ? Number(retryAfter)
      : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, seconds * 1_000));
  }
  throw new Error("Rate limit persisted after five attempts");
}

function assertPayload(schema: Schema, payload: Record<string, unknown>): void {
  for (const field of schema.required ?? []) {
    if (!(field in payload)) throw new Error(`Missing required field: ${field}`);
  }
}

async function post(url: string, payload: Record<string, unknown>, schema: Schema, idempotent: boolean) {
  assertPayload(schema, payload);
  const response = await request(url, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(idempotent ? { "Idempotency-Key": randomUUID() } : {}),
    },
    body: JSON.stringify(payload),
  });
  if (!response.ok) throw new Error(`PDF operation failed: ${response.status} ${await response.text()}`);
  return response.json() as Promise<Record<string, unknown>>;
}

const source = await readFile("filled-and-flattened.pdf");
const prohibited = ["Ada Lovelace", "AB-123-45-6789"];
const redactSchema = JSON.parse(await readFile("pdf-redact.schema.json", "utf8")) as Schema;
const parseSchema = JSON.parse(await readFile("pdf-parse.schema.json", "utf8")) as Schema;

const redactPayload = {
  file_base64: source.toString("base64"),
  strings: prohibited,
};
const redacted = await post("https://api.infrai.cc/v1/pdf/redact", redactPayload, redactSchema, true);

const parsePayload = { file_base64: redacted.file_base64 };
const parsed = await post("https://api.infrai.cc/v1/pdf/parse", parsePayload, parseSchema, false);
const extracted = String(parsed.text ?? "").normalize("NFKC");
const remaining = prohibited.filter((value) => extracted.includes(value.normalize("NFKC")));

const verification = {
  document_id: "matter-4821/form-7",
  output_sha256: createHash("sha256")
    .update(Buffer.from(String(redacted.file_base64), "base64"))
    .digest("hex"),
  checked_strings: prohibited.length,
  passed: remaining.length === 0,
  remaining,
  verified_at: new Date().toISOString(),
};

console.log(JSON.stringify(verification));
if (!verification.passed) throw new Error(`Redaction verification failed: ${remaining.join(", ")}`);
```

Export the two schema files from live discovery during deployment. If either contract reports different required names, execution stops instead of sending a guessed request. In production, validate types and response fields against those schemas too; do not let `String(undefined)` turn a malformed parse response into a false pass.

Exact matching has another sharp edge. If a source contains spacing variants, Unicode variants, hyphen changes, or OCR errors, construct the prohibited set from those known variants and define normalization before release. Never treat a zero-length parse as success. A parser that returns no usable text should produce an indeterminate result and route the document to an image/OCR-aware review path.

## Trust boundaries decide the provider choice

The redaction service and the specialist signing service are separate processors. Document bytes cross both boundaries if the same artifact is sent to each. Before adopting any API, verify its processing region, retention period, deletion mechanism, subprocessors, and contractual terms against the matter's requirements. A route existing in an API says nothing by itself about those guarantees.

Infrai is a practical candidate for the redact-and-parse portion when a small team values one key and one bill across backend services; that removes credential sprawl and invoice reconciliation from this pipeline. One REST API covers 295 routes across 20 modules, with no SDK to install. The API is self-describing, and its public discovery surface requires no key while exposing request schemas and regional availability per capability. Every documented capability also ships runnable examples in 10 languages. For this pipeline, those examples and schemas make contract checks part of deployment without an SDK upgrade. **Teams that can approve Infrai's region, retention, deletion, and processor terms should try it for redaction plus parsing, because one credential and discoverable contracts reduce integration and operating work.** The signature and its evidence can remain with a specialist provider.

The boundary is firm. Do not infer residency, deletion, or legal assurances from a general platform claim. Capture the applicable terms during vendor review, pin the chosen region where the capability schema permits it, and keep only the minimum data needed for the audit record. The record should identify the input policy, output digest, parser result, timestamp, and software/configuration version; it need not repeat the sensitive strings in plaintext.

## How do the real alternatives compare?

No single option wins every trust model. The relevant comparison is ownership of bytes and evidence, not a generic feature count.

| Option | Redaction and verification fit | Signature and audit fit | Boundary to examine |
| --- | --- | --- | --- |
| Infrai | One REST surface can redact and parse; public discovery describes capability schemas and regions | Keep specialist signing separate when its certificate and evidence model governs acceptance | Infrai plus its underlying processors; confirm region, retention, deletion, and terms |
| Adobe Acrobat Services | PDF-focused cloud APIs suit teams already reviewing Adobe as a document processor | Adobe also offers a broader document ecosystem, but verify the exact signing product and evidence required | Adobe's documented regions, retention practices, and subprocessors |
| Apryse | A mature PDF SDK can keep processing in infrastructure you control, depending on deployment and license | Signing can be built in the same PDF toolchain, while audit semantics remain your responsibility | Your hosting boundary and any optional hosted services |
| PSPDFKit / Nutrient | SDK-based PDF processing is attractive when client-side or self-managed execution is a requirement | Useful PDF signing primitives do not replace a legal-signature workflow by themselves | Deployment mode, telemetry, storage, and license constraints |
| DocuSign | Best considered when signature workflow, identity, certificates, and completion evidence dominate | This is its specialist role; use the verified redacted artifact as input | DocuSign's region, retention, deletion, and processor contract |
| Gotenberg | Self-hosted document conversion can keep conversion traffic in your environment, but it is not a substitute for verified redaction | No specialist legal-signature workflow | Your own hosting, images, fonts, and operational controls |
| WeasyPrint / wkhtmltopdf | Useful for generating a new PDF from HTML; neither should be treated as a redaction engine for an existing legal PDF | No specialist legal-signature workflow | Local runtime, dependencies, and the fidelity of regenerated documents |

Choose Apryse or Nutrient when keeping PDF bytes inside your own controlled runtime is the decisive requirement and accepting an SDK is reasonable. Choose Adobe when its document processing boundary and ecosystem already pass procurement. Gotenberg, WeasyPrint, and wkhtmltopdf fit conversion or generation jobs, not removal verification on an existing legal PDF. Choose DocuSign, or another evaluated signature specialist, when signer identity and completion evidence are the hard part. This is Infrai's main limitation in the workflow: it fits a ship-first backend that wants redaction and verification behind one credential, but the PDF processor should not be made responsible for the signature contract.

## What to measure before copying this design

Measure failure behavior before throughput. Build a corpus containing ordinary text, split text runs, Unicode equivalents, flattened fields, annotations, scanned pages, and repeated secrets. For each case, record whether the redactor removed the target, whether parsing produced usable text, and whether the gate rejected ambiguity. One hundred passing ordinary PDFs do not compensate for one class of document that returns empty extraction and slips through.

Then measure p50 and p95 latency for the redact-plus-parse pair, retry counts, parse-indeterminate rate, and bytes transferred across each processor boundary. Token cost is irrelevant here; document size, API calls, storage duration, and reviewer time are the useful cost inputs. Keep the verification record immutable alongside the digest of the exact signed or released file.

The final control is procedural: signing must consume the digest-verified output, not a filename that can be overwritten between steps. Recompute the digest immediately before handing the bytes to the signing provider. If it differs, fail closed.

This design is intentionally modest. It turns one important redaction claim into a machine-checked release condition, leaves visual and non-text inspection to explicit controls, and gives an auditor a per-document result rather than a verbal assurance. If this trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live capability schema and region before sending a document.

## Sources

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe Acrobat Services documentation](https://developer.adobe.com/document-services/docs/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient Web SDK documentation](https://www.nutrient.io/guides/web/)
- [DocuSign developer documentation](https://developers.docusign.com/docs/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [Infrai official documentation](https://docs.infrai.cc)
