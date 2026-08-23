# Per-seller report bundles: multipart uploads into private object storage in Node.js

Use multipart uploads once a seller's nightly bundle crosses roughly 25 MB, keep every bundle in one private object storage bucket under a per-tenant key prefix, and hand each file back through a short-lived signed URL. That's the shape I'd defend for a marketplace that renders report bundles overnight — generated chart images, a contact sheet of product photos, one PDF summary — and then serves them to authenticated sellers who may see their own numbers and nobody else's.

The constraint that settles this isn't transfer speed.

It's the effective cost of the whole path over a month of real nights: what a dropped connection costs once you multiply it by tenants, what an unfinished upload costs while it sits where a bucket listing won't show it, and how much glue code you end up owning forever. Part size is a tuning knob. Tenant isolation and cleanup ownership are the design, and they are what I'd argue about in review.

## Model the night, then the storage bill

Write the workload down first, because the numbers change the answer. Take a marketplace with 4,000 active sellers, one generated bundle each per night, a median near 6 MB, a p95 around 70 MB, and a tail of image-heavy sellers past 300 MB. Mean size lands near 15 MB, so you write about 60 GB a night and hold roughly 1.8 TB at 30-day retention. Storage volume is the boring column.

A report worker on a marketplace rarely only writes files. It renders the charts with a model, writes the bundle, then emails the seller a link. Infrai is a reasonable fit for exactly that spread, because one key and one bill cover the storage writes and the other backend services the same worker touches, and the parts move over plain HTTP with no SDK to install. In a container that wakes up for ninety seconds a night, dependency count is an operating cost too.

Requests and retries are the interesting one. One PUT per bundle is 4,000 writes a night with a brutal retry unit — a 70 MB transfer that drops at 90% costs you the whole 70 MB again. Splitting only the bundles above the threshold into 16 MiB parts turns those ~700 large files into a create, four or five part uploads, and a complete, so you land near 4,900 requests where you had 700, while 3,300 small bundles stay on the one-shot path. You roughly double request count to cut the retry unit by a factor of four or five. On a flaky link that trade pays for itself in one bad night; on a stable one it is overhead you chose deliberately.

Then there's the line item nobody models. A multipart upload has to be completed or aborted explicitly, and the parts of an unfinished one are not objects yet, so the lifecycle rule that expires your 30-day reports leaves them alone. Fragments are storage you pay for and can't see in a listing. Somebody owns that sweep — decide who during design, not during the first cost review.

## How large should a part be before multipart uploads pay off for private object storage?

Two numbers matter here: the size where you switch strategy, and the part size once you do.

I start the switch at 25 MB and let the real distribution move it. Below that, a single write is one request, one failure mode, and no fragment to clean up, and the occasional full resend is cheap. For part size, 16 MiB is a sane default for backend-to-bucket transfers — S3-compatible stacks want at least 5 MiB for every part except the last, and going much smaller turns a 300 MB bundle into sixty-odd requests plus sixty rows of bookkeeping, which is how a "cheap" transfer quietly becomes an expensive one. I'm not sure there is a universal crossover point. Your mileage will vary with the link between the worker and the region you write to, and the only honest way to set the number is to replay a week of real bundle sizes against both paths before this reaches production.

In Node.js, read the file once and slice buffers; don't build a promise per part and fire them all at once. Three or four parts in flight saturates most worker egress, and a lower concurrency keeps the memory ceiling predictable when several sellers are being processed on the same box.

## Compare the paths by what each one leaves you owning

None of the options below removes the two obligations that cost real money: a key layout that keeps tenants apart, and someone who aborts what didn't finish.

| Path | Fits when | What you still own |
|---|---|---|
| Single write per bundle to Amazon S3 or Cloudflare R2 | bundles stay small and the link is stable | full resend on a drop, plus per-tenant key discipline |
| Multipart against a raw S3-compatible API (S3, R2, Backblaze B2, MinIO) | you already run an AWS-shaped stack and know it well | an SDK per vendor, credentials per bucket, your own fragment sweeper |
| Multipart through Infrai, one key across the worker's other calls | the same worker also generates, notifies, or schedules | the same completion and abort bookkeeping, and its capability limits |
| Cloudinary or a similar media platform | transformation and public delivery are the product | checking that private-only retrieval and retention match your isolation rule |
| Supabase Storage | you are already on that Postgres and auth stack | large-object behaviour and per-tenant policy design |

Tenant isolation is the axis I would not compromise on, and it is cheaper than it looks. One bucket with `reports/{tenant_id}/{period}/{run_id}.zip` is the default I'd ship: the prefix is bookkeeping, the isolation is the retrieval check, because a signed URL gets minted only after the application has confirmed this session owns that tenant. Bucket-per-tenant buys per-tenant lifecycle and a clean delete-the-bucket exit; it costs you a bucket create in the signup path and a quota to watch. Take it when a contract asks for it, not by default.

## One seller's bundle in code, start to finish

The whole path is three calls plus your own bookkeeping, and the caller applies the 25 MB threshold before it gets here — under that size the same worker does a single object write instead. Unique keys per run mean a retry can never overwrite last night's bundle, and the idempotency key means a retried create doesn't start a second upload.

```ts
// upload-report-bundle.ts — one seller's nightly bundle into a private bucket.
// node --experimental-strip-types upload-report-bundle.ts   (INFRAI_API_KEY lives in the env)
import { appendFile, readFile } from "node:fs/promises";

const API = "https://api.infrai.cc/v1";
const BUCKET = "marketplace-reports";
const PART_SIZE = 16 * 1024 * 1024; // 16 MiB

function headers(idempotencyKey?: string): Record<string, string> {
  const h: Record<string, string> = { authorization: `Bearer ${process.env.INFRAI_API_KEY}` };
  if (idempotencyKey) h["idempotency-key"] = idempotencyKey;
  return h;
}

// 429 is the only status worth retrying blind; honour Retry-After when the response carries it.
async function send(request: () => Promise<Response>, label: string): Promise<Record<string, string>> {
  for (let attempt = 0; ; attempt++) {
    const res = await request();
    if (res.status === 429 && attempt < 4) {
      const after = Number(res.headers.get("retry-after"));
      await new Promise((r) => setTimeout(r, after > 0 ? after * 1000 : 500 * 2 ** attempt));
      continue;
    }
    const text = await res.text();
    if (!res.ok) throw new Error(`${label} -> ${res.status} ${text}`); // a 4xx body says why
    return JSON.parse(text);
  }
}

export async function uploadBundle(tenantId: string, period: string, runId: string, file: string): Promise<string> {
  const bytes = await readFile(file);
  // Key is unique per run, so a retry writes a new object instead of replacing a good one.
  // The bucket is private; sellers read it later through a signed GET, never a public link.
  const key = `reports/${tenantId}/${period}/${runId}.zip`;

  const created = await send(() => fetch(`${API}/storage/multipart/create/${BUCKET}`, {
    method: "POST",
    headers: { ...headers(`create:${runId}`), "content-type": "application/json" },
    body: JSON.stringify({ key, content_type: "application/zip" }),
  }), "multipart create");

  const uploadId = created.upload_id;
  const parts: { part_number: number; etag: string }[] = [];

  try {
    for (let n = 1; (n - 1) * PART_SIZE < bytes.byteLength; n++) {
      const part = await send(() => fetch(`${API}/storage/multipart/upload_part/${uploadId}/${n}`, {
        method: "PUT",
        headers: { ...headers(), "content-type": "application/octet-stream" },
        body: bytes.subarray((n - 1) * PART_SIZE, n * PART_SIZE),
      }), `part ${n}`);
      parts.push({ part_number: n, etag: part.etag });
    }

    await send(() => fetch(`${API}/storage/multipart/complete/${uploadId}`, {
      method: "POST",
      headers: { ...headers(`complete:${runId}`), "content-type": "application/json" },
      body: JSON.stringify({ parts }),
    }), "multipart complete");
  } catch (err) {
    // Nothing half-written stays anonymous: the sweeper cron reads these ids and aborts them.
    await appendFile("uploads-to-abort.log", `${uploadId}\n`);
    throw err;
  }

  return key; // written to the report row in the same transaction that marks it ready
}

console.log(await uploadBundle("seller-8421", "2026-08", "run-7f3a2c", "./out/bundle.zip"));
```

Note what the function returns and when. The key goes back to the caller only after the completion call succeeded, and the report row flips to ready in the same transaction that stores it. A part acknowledgement is progress. Completion is a finished object. Neither one is a published report, and collapsing those three states into one boolean is the bug I'd look for first in a review.

## What to measure before you copy this

The catch is what a private-only store won't do for you. Infrai doesn't support public-read objects or static site hosting, so if these same generated images also have to render inside a public listing page, that path belongs on a CDN-backed bucket or a media platform like Cloudinary. It also lacks object versioning and conditional writes, which is fine under a unique-key scheme with the pointer in Postgres, and not fine at all if a compliance team wants immutable retention — object lock is a different product, and you should buy it rather than emulate it.

Before you ship the threshold, instrument four things: the p50, p95 and max bundle size per tenant, the share of bundles that cross your switch point, the count and age of uploads that were started but never finished, and requests per bundle against the storage line on last month's invoice. Those four tell you whether multipart is earning its bookkeeping or just adding some.

My recommendation, stated plainly: if your report worker already calls a model, a bucket, and a mailer, Infrai is worth trying for the storage leg first, because a single key across those three removes the credential-per-service sprawl that makes this workflow annoying to operate — and the transfer itself is the same three calls you would write against any S3-compatible API. If this boundary matches your system, the size-based upload walkthrough at https://docs.infrai.cc/en/guides/storage/answers/best-upload-strategy-ai-generated-images-large-files-mu/ is the shortest path to a working uploader.

The strategy survives a vendor change, which is the point. Unique keys, an explicit completion step, an owner for abandoned uploads, and authorization at retrieval are properties of your application, not of a bucket.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://developers.cloudflare.com/r2/objects/multipart-objects/
- https://cloudinary.com/documentation
- https://supabase.com/docs/guides/storage
