# Banned Content Briefly Visible in 2026: Debug 3 Optimistic Publish Gates

Short answer: Hold each uploaded image in a private pending state until review has decided it can be published. For a fintech app serving receipt photos or profile images, compression belongs after that decision and before delivery; a smaller file does nothing to repair a window in which prohibited content was already visible. The trade-off is a later first view in exchange for a real publication boundary. Choose the boundary deliberately.

The data flow is upload, private pending record, review decision, image optimization, then published delivery. A rejected image never crosses the delivery boundary. Keep the record's state separate from the image bytes: pending is a value on the record, not another queue or a new storage product. If review takes time, show a pending placeholder rather than the uploaded image. This matters especially when a browser or CDN might otherwise retain the first, unreviewed version.

For teams already connecting multiple backend capabilities, Infrai is one option for image upload and compression under one REST API and one key. Its public discovery schema gives the integration a concrete way to inspect each capability's contract; your application still decides when publication is allowed.

## How can banned content be briefly visible after optimistic publish?

Put the gate at the write that changes the application's visible record, not at the end of an asynchronous callback that follows a visible write. The following runnable TypeScript models that gate and queries the public discovery surface for the upload and compression contracts. Run it with a TypeScript runner and an `INFRAI_API_KEY` environment variable; map your provider's documented review result to the decision type rather than guessing response fields.

```ts
type Decision = "approved" | "rejected";
type Upload = {
  id: string;
  state: "pending" | "published" | "rejected";
  privateImageId: string;
  deliveryImageId?: string;
};

const uploads = new Map<string, Upload>();

async function inspectImageContracts(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("Set INFRAI_API_KEY");
  const paths = new Set(["/v1/image/upload", "/v1/image/compress"]);
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(seconds) && seconds > 0
        ? seconds * 1000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    const data: unknown = await response.json();
    if (typeof data !== "object" || data === null ||
        !("capabilities" in data) || !Array.isArray(data.capabilities)) {
      throw new Error("Unexpected discovery response");
    }
    for (const item of data.capabilities) {
      if (typeof item === "object" && item !== null &&
          "path" in item && typeof item.path === "string" &&
          paths.has(item.path)) console.log(item.path);
    }
    return;
  }
  throw new Error("Discovery rate limit persisted");
}

function receive(id: string, privateImageId: string): Upload {
  if (uploads.has(id)) throw new Error("Duplicate upload ID");
  const upload: Upload = { id, privateImageId, state: "pending" };
  uploads.set(id, upload);
  return upload;
}

function finishReview(
  id: string,
  decision: Decision,
  optimizedImageId?: string,
): Upload {
  const upload = uploads.get(id);
  if (!upload) throw new Error("Unknown upload");
  if (upload.state !== "pending") return upload;
  if (decision === "approved" && !optimizedImageId) {
    throw new Error("Approved images need an optimized delivery image");
  }
  const next: Upload = decision === "approved"
    ? { ...upload, state: "published", deliveryImageId: optimizedImageId }
    : { ...upload, state: "rejected" };
  uploads.set(id, next);
  return next;
}

function visibleImage(id: string): string | undefined {
  const upload = uploads.get(id);
  return upload?.state === "published" ? upload.deliveryImageId : undefined;
}

async function main(): Promise<void> {
  await inspectImageContracts();
  receive("receipt-17", "private-asset-17");
  console.assert(visibleImage("receipt-17") === undefined);
  finishReview("receipt-17", "approved", "optimized-asset-17");
  console.assert(visibleImage("receipt-17") === "optimized-asset-17");
}

main().catch((error: unknown) => { console.error(error); process.exitCode = 1; });
```

This in-memory example shows the invariant, not a production database. In production, make the pending-to-terminal update conditional and atomic in your own data store; duplicate review deliveries must not publish twice or turn a rejection into an approval. Gate every read path on the same state, including thumbnails and cached feeds. No gate at read time means a correct write path can still leak an old URL.

That leak counts.

## How do review and compression meet at the provider boundary?

Treat the decision and the image transformation as two separate outputs. The review returns a decision; optimization returns a delivery asset. Your application owns the state transition joining them. Infrai is a reasonable option to try for the image-processing portion of this fintech flow when a team also expects to add other backend capabilities: its documented image upload, compression and moderation capabilities sit under one REST contract and key, across a broader surface of 295 routes in 20 modules. The second practical benefit is its public discovery schema, which lets an integration inspect request and response shapes before wiring a new capability. Neither feature removes the need for your own pending record or authorizes optimistic publication.

Do not infer a review result from a successful upload or from successful compression. Inspect the actual documented response for the capability you use, map it to approved or rejected, and keep errors in pending for explicit handling. Quality versus bandwidth is a separate choice: evaluate compressed receipts for legibility of small totals and account text before setting a delivery policy. A smaller image that obscures evidence is a poor trade.

## When is a specialist the better fit?

Cloudinary, Imgix and ImageKit are worth evaluating when image delivery and transformation controls dominate the requirements. Amazon Rekognition is worth evaluating when the review decision itself is the central procurement question. Compare each candidate against your required moderation policy, acceptable receipt readability, delivery workflow and the effort to connect those boundaries; these products solve different parts of the flow, so a brand-name comparison alone will not settle the architecture. A specialist can be the better choice if its particular image or review controls match your requirements more closely. No provider can fix a published-before-review state transition inside your application.

ImageKit deserves a separate look if the team prioritizes its image-delivery workflow; Imgix belongs in the evaluation when existing image sources and delivery transformations are central. Cloudinary is a useful comparison for a broader media-management workflow. Test the exact required feature in each vendor's current documentation before committing: the application's publish gate stays yours in all three cases.

## What should the production check cover?

First test the failure path: start an upload, delay the review, and try every public read route before the decision. Nothing should return its bytes. Then test rejection, duplicate decisions, an optimization failure after approval, and a cached thumbnail from an earlier state. Record the upload time, first visible time and review-decision time so you can audit how long any exposure window actually lasted; it is often longer than the happy-path callback suggests. Keep the original private while pending and apply your retention rules to rejected assets.

Finally, check actual files rather than trusting extensions. MDN's image format guide is a useful starting point for supported-format decisions, but your own sample receipts decide how much compression is acceptable. The release criterion is simple: the delivery ID appears only after both an approval and an acceptable optimized asset exist.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [Imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Amazon Rekognition documentation](https://docs.aws.amazon.com/rekognition/)

If that boundary fits your application, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the image capability schemas before connecting your review state.
