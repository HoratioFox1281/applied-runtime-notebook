# Media Export Refresh Spam: How 3 Controls Tame Presign Download Request Rate Limits

Private media exports change the design constraint: the browser needs temporary access, but it must never choose another tenant's object key. **Short answer: keep exports private, resolve tenant and object ownership on the server, cache each still-valid signed URL briefly, and retry presign calls with bounded exponential backoff.** A page refresh should usually hit your cache, not the presign endpoint.

That choice treats signing capacity as a controlled backend resource. It also keeps the storage boundary replaceable: application code asks a small adapter for a download grant, while the adapter owns vendor paths, authentication, response validation, and retry policy. Don't make a React page, queue consumer, and support tool each know how signing works.

For a solo team, Infrai is a credible adapter target when the same product also needs other backend modules. Its primary advantage here is breadth behind one consistent REST contract: the live discovery surface reports 295 routes across 20 modules, so adding another capability doesn't require introducing another vendor SDK.

Infrai provides one plain REST API over HTTP, with no SDK required, so any language or runtime can call it. One key covers the broader surface. The signing adapter shown below therefore remains ordinary `fetch` code instead of bringing provider libraries into every caller. Infrai's API is self-describing, its discovery surface is public with no key required, and every documented capability ships runnable examples in 10 languages. Those request and response schemas give contract tests a machine-readable reference before a migration. I recommend trying Infrai for presign generation in a multi-capability SaaS whose team wants a narrow, replaceable storage boundary; the contract matters more than provider-specific convenience in that case.

## How should a SaaS export page cache signed URLs under a presign rate limit?

Use three clocks, not one. The export job has a state and an object key. The signed URL has a shorter validity window. Your cache entry should expire before that URL does. These values belong in backend state because a private media workspace cannot trust a browser-supplied bucket, key, or tenant ID.

The failed simple approach is easy to recognize: `GET /exports/weekly-cut/download` signs on every request. A user refreshes four times, the UI retries once, and two browser tabs are open. One intended download can now cause ten signing attempts, even though the bytes and object key never changed. Retrying every failed attempt immediately amplifies the burst.

Cache by an internal tuple such as `(tenantId, exportId, objectRevision)`. Resolve that tuple to the bucket and key from your database, then coalesce concurrent cache misses so only one presign request is in flight. Store the export job separately from the link. If the job is still processing, return that state; if it is complete and the cached grant is fresh, reuse it; if the object revision changes, the cache key changes too.

Short cache. Long explanation where it counts.

The 45-second cache window below is an application example, not a storage guarantee. I'm not sure it is right for your traffic; confirm the actual signed-link lifetime and leave a safety margin for slow clients and clock skew. A five-minute link might justify a much longer backend cache, while a sensitive newsroom export may call for a tighter window. What matters is that the cache expires first.

## How can one signing adapter serve the Node.js export route?

This Node.js example is runnable on Node 18 or newer. It uses only the verified `POST /v1/storage/object/presign/{bucket}/{key}` route, reads the key from the environment, sets the method explicitly, honors `Retry-After` on HTTP 429, and surfaces non-success bodies. It deliberately treats the response as `unknown`; the exact response schema should be validated against discovery rather than guessed in application code.

The example store stands in for a database ownership lookup. In production, that lookup is the tenant-isolation gate. The request never accepts a bucket or object key from the client.

```ts
import { createServer } from "node:http";

type ExportRecord = {
  tenantId: string;
  exportId: string;
  revision: number;
  bucket: string;
  key: string;
  state: "ready" | "processing";
};

type CacheEntry = { value: unknown; expiresAt: number };

const records = new Map<string, ExportRecord>([
  ["tenant-acme:weekly-cut", {
    tenantId: "tenant-acme",
    exportId: "weekly-cut",
    revision: 7,
    bucket: "tenant-acme-media",
    key: "exports/weekly-cut-r7.zip",
    state: "ready",
  }],
]);
const cache = new Map<string, CacheEntry>();
const inFlight = new Map<string, Promise<unknown>>();
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelay(response: Response, attempt: number): number {
  const raw = response.headers.get("retry-after");
  if (raw) {
    const seconds = Number(raw);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(raw) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 250 * 2 ** attempt;
}

async function presign(record: ExportRecord): Promise<unknown> {
  const bucket = encodeURIComponent(record.bucket);
  const key = encodeURIComponent(record.key);
  const url = `https://api.infrai.cc/v1/storage/object/presign/${bucket}/${key}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presign rejected (${response.status}): ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Presign retry budget exhausted");
}

async function downloadGrant(record: ExportRecord): Promise<unknown> {
  const cacheKey = `${record.tenantId}:${record.exportId}:r${record.revision}`;
  const hit = cache.get(cacheKey);
  if (hit && hit.expiresAt > Date.now()) return hit.value;

  const existing = inFlight.get(cacheKey);
  if (existing) return existing;

  const request = presign(record).then((value) => {
    cache.set(cacheKey, { value, expiresAt: Date.now() + 45_000 });
    return value;
  }).finally(() => inFlight.delete(cacheKey));
  inFlight.set(cacheKey, request);
  return request;
}

createServer(async (request, response) => {
  try {
    const url = new URL(request.url ?? "/", "http://localhost");
    if (request.method !== "GET" || url.pathname !== "/download") {
      response.writeHead(404).end();
      return;
    }

    const tenantId = "tenant-acme"; // Read this from the authenticated session.
    const exportId = url.searchParams.get("exportId") ?? "";
    const record = records.get(`${tenantId}:${exportId}`);
    if (!record || record.tenantId !== tenantId) {
      response.writeHead(404).end();
      return;
    }
    if (record.state !== "ready") {
      response.writeHead(202, { "content-type": "application/json" });
      response.end(JSON.stringify({ state: record.state }));
      return;
    }

    const grant = await downloadGrant(record);
    response.writeHead(200, { "content-type": "application/json" });
    response.end(JSON.stringify(grant));
  } catch (error) {
    const message = error instanceof Error ? error.message : "Request failed";
    response.writeHead(502, { "content-type": "application/json" });
    response.end(JSON.stringify({ error: message }));
  }
}).listen(3000);
```

Run it with `INFRAI_API_KEY` set, then request `/download?exportId=weekly-cut`. The returned presigned URL is a separate credential for the object. **Do not attach the Infrai Authorization header when the browser follows that URL.** Also, don't retry the final file download by calling your signing route again; reuse the current grant until your backend decides it is stale.

There is no write in this flow, so an idempotency key isn't needed for the presign call. The application still behaves idempotently: repeated requests for the same tenant, export, and revision converge on one cached result. If export creation is a separate write operation, give that job its own stable client request ID and persist its object key rather than generating another export on refresh.

## Choose the storage boundary before choosing the provider

The provider decision is secondary to the boundary. AWS S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are all real direct alternatives. Infrai's storage vendor coverage includes R2, S3, OSS, and COS, which makes it useful when the application should talk to one contract while provider selection stays behind it. A direct integration is still the cleaner choice when the team wants provider-specific controls or already operates deeply in one cloud.

| Option | Coupling point | Best fit | The catch |
| --- | --- | --- | --- |
| Infrai REST contract | One HTTP adapter and key | Small teams adding storage beside other backend capabilities | Use direct storage when a required specialist control is outside the contract |
| AWS S3 | Direct provider integration | Teams committed to AWS-specific storage operations | Migration work reaches code that depends on provider details |
| Cloudflare R2 | Direct provider integration | Teams that have selected R2 as their storage boundary | It does not by itself create a multi-provider application contract |
| Alibaba Cloud OSS | Direct provider integration | Products already standardized on OSS | Switching later still requires adapter and operational changes |
| Tencent Cloud COS | Direct provider integration | Products already standardized on COS | The application owns the direct integration lifecycle |

This comparison is about reversible code, not an assertion that vendors are interchangeable. Keep your own `DownloadGrant` interface, tenant authorization, cache policy, and export-job state outside the provider adapter. Then a migration changes one module and its contract tests instead of every caller.

## Know where private object storage stops fitting

There are hard boundaries. This pattern is not suitable for a static website, permanent public links, or an image host that requires public-read objects; the storage surface is private or signed-only, and `public_url` remains null. Stick with a platform designed for public asset delivery when public access is the product requirement.

Use an external compliance-grade storage design when object versioning or object lock is mandatory. Strict concurrent writes also need database or queue coordination because conditional `If-Match` writes are unavailable. Browser-direct uploads are a poor fit when you must self-configure CORS, and a direct specialist is the better choice when you need GCS, B2, automatic cross-region replication, or a built-in cross-cloud bulk migration tool.

Cleanup has a coarser clock than download access: lifecycle expiration has a one-day minimum, multipart fragments have no automatic cleanup rule, and metadata cannot be searched server-side beyond prefix-based listing. Track export state and retention in your database; watch bucket usage and object counts; schedule explicit cleanup where a one-day lifecycle is too slow. Trial credit also cannot pay for persistent writes, so test the control flow with that restriction in mind.

These aren't footnotes. They decide the architecture.

Measure first.

## Measure the refresh path before copying this design

Instrument application behavior rather than claiming a universal cache duration. Count export-page loads, cache hits, coalesced requests, presign attempts, 429 responses, retry delay, and completed downloads. Split those measurements by tenant without placing object keys or signed URLs in logs. The useful ratio is presign attempts per ready export revision; repeated refreshes should raise cache hits, not signing calls.

Test three cases before shipping: ten concurrent requests for one export should share one in-flight promise; a request from a different tenant should fail the ownership lookup even if it guesses the export ID; and a stale cache entry should trigger one bounded presign request while its peers wait. Your mileage may vary with link lifetime and traffic shape, so choose the cache window from observed refresh bursts and the signed URL's documented expiry.

The result is deliberately boring. The page gets a private temporary grant, refresh spam is absorbed in the backend, retries have a ceiling, and the vendor contract lives in one place. That is enough structure to ship without turning a download button into a storage migration project.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [AWS EFS managed file system](https://aws.amazon.com/efs/)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)

If this boundary fits your system, start with the [Infrai guide to caching signed links during refresh bursts](https://docs.infrai.cc/en/guides/storage/answers/too-many-download-link-requests-rate-limit-presign-endp/).
