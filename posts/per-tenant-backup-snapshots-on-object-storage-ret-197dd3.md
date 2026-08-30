# Per-Tenant Backup Snapshots on Object Storage: Retention, Signed Links, No Public Bucket

Use one private bucket per region, a tenant-scoped key prefix inside it, a retention rule per tenant tier, and short-lived signed download links issued by your app — that is the layout I keep coming back to for app data backups in the US and EU. The deciding constraint isn't the monthly storage rate. It's whether you can restore one customer's snapshot without reading, moving, or exposing another customer's data.

I run a small customer support product. Tickets, attachments, and a per-tenant settings blob, nothing exotic. The backup requirement is boring on paper: nightly snapshots, keep them for the contracted window, and be able to hand a named snapshot back to one tenant when they delete something they shouldn't have.

The drill that decides the architecture is narrow. Restore tenant 4711 from the snapshot taken nine days ago, from the EU copy, while everything else keeps writing.

Cheapest is a real requirement here — I'm a solo founder and storage bills compound quietly — but the cheap number on a pricing page is the archive rate, and that's rarely the number that hurts. Requests and the bytes you pull during a restore are.

## The single-object nightly dump that failed the drill

The first version was one compressed dump of the whole database per night, one object per day, `backups/2026-08-02.sql.gz`. Simple to write, simple to schedule, and it passed every test I bothered to run, because the only test I ran was "did the upload succeed."

Then I tried the actual restore. To recover one tenant's tickets I had to download the entire archive, decompress it on a restore host, and filter out a few hundred megabytes of rows from tens of gigabytes covering everyone else. The download cost is annoying. The isolation problem is worse: a routine, single-tenant restore now puts every tenant's data on a temporary disk, in a process written under time pressure, during an incident. That's a blast radius nobody signs off on when you describe it out loud.

Retention broke the same way. One tenant is on a 30-day contractual window, another negotiated a year, and a third asked for deletion on offboarding. A single object that contains all three can't be expired by any one of those rules — lifecycle expiration in the S3 model operates on object keys and prefixes, with a minimum granularity of one day, so the retention policy has to be expressible in the key layout or it doesn't exist.

Per-tenant snapshots fix both problems at once, and they cost more in object count and in scheduler complexity. That's the trade I took.

Test the restore, not the upload.

## How should per-tenant backup retention and signed download links work without a public bucket?

Pick the isolation boundary first; everything else follows from it. There are three sane layouts, and the honest differences between them are operational rather than technical.

| Layout | Isolation boundary | Where it hurts |
|---|---|---|
| One bucket, `t/<tenant>/` prefix | Application code and IAM prefix conditions | A logic bug in link issuance crosses tenants; one lifecycle config to keep correct |
| One bucket per tenant | Bucket policy and lifecycle config | Bucket quotas — AWS documents a default of 100 general purpose buckets per account, raisable by request — and config drift across hundreds of buckets |
| One account or project per tenant | Credentials and billing | Real isolation, real overhead; only worth it when a contract or auditor demands it |

For a few hundred tenants the prefix layout is the one I'd defend. The important part is that the prefix is not decoration: the credential the backup worker holds should be restricted to `t/<tenant>/*` when it runs for that tenant, and the restore path should refuse to sign a key whose prefix doesn't match the authenticated session's tenant. Isolation enforced only by string concatenation in application code is one typo away from a cross-tenant leak.

The bucket itself stays private. No public read, no ACL exceptions, no "temporarily" open bucket for a migration that never gets closed. When an operator or a customer needs the file, the app issues a presigned URL with a short expiry — a signature over the request that grants time-boxed access to one object without making the object public. AWS documents that mechanism directly, and every S3-compatible implementation I've worked against follows the same shape.

The link is a bearer credential.

Two consequences follow from that, and both catch people out: anything that logs full URLs (proxies, browser history, support chat transcripts) now holds a working download link until it expires, and a presigned URL does not grant a browser cross-origin permission. If your restore UI fetches the object with JavaScript rather than navigating to it, the bucket needs a CORS configuration or the request fails in the browser while the same URL works fine from curl.

## The snapshot manifest is the part that gets skipped

Server-side metadata isn't searchable in the S3 model. Listing filters by prefix, nothing else. So "restore the snapshot from nine days ago" is only answerable if the key layout and a small manifest make it answerable — otherwise you're listing thousands of objects and parsing filenames during an incident.

I keep the tenant, dataset, and timestamp in the key, and write one manifest object per snapshot next to the data. The manifest carries the row counts and a content hash, which is what turns "the file downloaded" into "the restore is valid."

```ts
type Snapshot = {
  tenantId: string;
  dataset: "tickets" | "attachments" | "settings";
  takenAt: string;      // ISO-8601, UTC
  bytes: number;
  rows: number;
  sha256: string;
};

const prefixFor = (tenantId: string) => `t/${tenantId}/`;

function snapshotKey(s: Snapshot): string {
  const day = s.takenAt.slice(0, 10);
  const stamp = s.takenAt.replace(/[:.]/g, "-");
  return `${prefixFor(s.tenantId)}${s.dataset}/${day}/${stamp}.sql.gz`;
}

// The store is any S3-compatible backend behind a five-method interface.
// Keeping it this small is what let me swap providers per region later.
interface ObjectStore {
  list(prefix: string): Promise<string[]>;
  getJson<T>(key: string): Promise<T>;
  signGet(key: string, ttlSeconds: number): Promise<string>;
}

async function issueRestoreLink(
  store: ObjectStore,
  session: { tenantId: string; actor: string },
  snapshotKeyPath: string,
): Promise<string> {
  // Isolation check before signing, not after. The signature is the permission:
  // once it's minted for the wrong prefix, no downstream check can take it back.
  if (!snapshotKeyPath.startsWith(prefixFor(session.tenantId))) {
    throw new Error(`cross-tenant restore blocked: ${session.tenantId}`);
  }

  const manifest = await store.getJson<Snapshot>(`${snapshotKeyPath}.manifest.json`);
  if (manifest.tenantId !== session.tenantId) {
    throw new Error("manifest tenant mismatch");
  }

  const url = await store.signGet(snapshotKeyPath, 900);
  audit.log("restore_link_issued", {
    tenant: session.tenantId, actor: session.actor,
    key: snapshotKeyPath, ttl: 900, sha256: manifest.sha256,
  });
  return url;
}
```

Fifteen minutes is my default expiry, and I'm not sure it's right for everyone — a 20 GB snapshot on a slow connection can outlive it, and a link that dies mid-download during an incident is its own kind of outage. Measure your restore duration before picking the number. The audit line matters as much as the signature: when a customer asks who downloaded their data, "we don't log that" is not an answer you want to give.

Attachment archives get a second consideration. Anything past a few gigabytes should go up as a multipart upload, and incomplete uploads leave paid-for fragments behind that no listing shows you by default — a lifecycle rule to abort incomplete multipart uploads after a few days is the cheapest line item you'll ever add.

## Where this design stops working

The catch is immutability. Prefix isolation and signed links are access control, and access control is not retention. An attacker or a buggy cleanup script with delete rights on the bucket can remove backups exactly as easily as the backup job created them. Versioning plus object lock in a separate account with separate credentials is the standard answer, and if your retention window is contractual or regulatory, stick with a backup system built for WORM rather than reproducing it out of lifecycle rules.

Prefix isolation also stops being enough when a tenant requires their own encryption key, their own audit trail, or a residency guarantee stronger than "the bucket is in the EU." That's a bucket-per-tenant or account-per-tenant conversation, and the extra cost is the point rather than a regrettable side effect.

Cross-region is not free either. A US bucket and an EU bucket are two buckets, two lifecycle configurations, and two restore paths that both need drilling; replication between them is an explicit feature you configure and pay for, not something S3 compatibility gives you. I'd rather run region-local backups for region-local tenants and accept that a full DR failover is a slower, separate procedure. There's a compliance angle to that decision as well: once EU tenant data has a copy sitting in a US region for convenience, the residency answer you gave in a security questionnaire quietly stops being true, and discovering that during a customer audit is far more expensive than the second bucket you avoided. Region-local by default, replicate only where a contract says to, and write down which tenants are in which region so the restore runbook doesn't have to guess.

## What to measure before copying this

Run the drill on real data, and record five numbers.

Time to locate the right snapshot for one tenant from a cold start. Bytes downloaded to complete that restore, which is your egress bill during a bad week. Wall-clock time from link issuance to a verified import, compared against the link expiry. Whether the retention sweep actually deleted a 40-day-old object under a 30-day rule — check the object, don't trust the console. And the per-request count of your nightly job, because thousands of small per-tenant objects change the request line on your invoice in a way the per-gigabyte rate won't show you.

Do all of it against a local S3-compatible server in CI first. MinIO runs as a single container, which makes a weekly automated restore drill cheap enough that you'll actually keep it. The drill is the deliverable; the bucket layout is just what makes the drill survivable.

One more thing worth writing down: the restore runbook should name the human who is allowed to request a snapshot for a tenant, and the manifest hash should be checked before anyone claims the restore succeeded. A file that arrives is not a backup that works.

## References

- [AWS S3: Multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [AWS S3: Sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [AWS S3: Managing the lifecycle of objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [AWS S3: Bucket restrictions and limitations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html)
- [AWS S3: Using S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
