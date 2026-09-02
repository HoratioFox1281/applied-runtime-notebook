# Private Document Storage for Europe-US Web Apps (A Compliance Retention Experiment)

Private document storage for a Europe-US web app is acceptable only when every tenant backup can prove its retention deadline, deletion state, and selected restore snapshot.

Short answer: gate private document storage on tenant isolation, region and compliance evidence, deterministic retention, deletion verification, and selected-snapshot restore tests; only compare cost among candidates that pass every gate.

For an e-commerce web app, model actual operating pressure: one merchant asks for yesterday's catalog documents back while another merchant's expired invoices must be deleted. A mutable `latest/` prefix is the failed/simple approach. It makes restoration depend on listing order and gives deletion workers no immutable unit of work. The stronger choice is a snapshot manifest per tenant plus explicit active, expired, and deletion-verified states.

This isn't a vendor ranking. AWS S3, Cloudflare R2, Wasabi, and Bunny Storage belong in the same harness, but a product page cannot decide whether a particular account configuration, region, contract, and workload pass your requirements.

## How should a Europe-US web app prove private document storage retention?

Start with two synthetic tenant fixtures in each required geography and three snapshots per tenant. Each snapshot should contain a manifest, several small documents, one larger document, and expected hashes. The test is about control-plane and data-plane behavior, not production-data migration.

Give every object an opaque tenant identifier and immutable snapshot identifier, such as `tenants/t_018/snapshots/s_20260813_0900/objects/invoice-0042.pdf`. Do not put names, email addresses, or order details in keys; keys can appear in logs, metrics, and support tooling. The manifest records each object key, byte count, media type, checksum, and retention decision. Restore membership comes from that manifest, never from a live prefix scan.

Test the boundary through the same credential shape and network path production will use:

1. Tenant A can create, read, and restore its own snapshot.
2. Tenant A cannot read or list Tenant B's keys.
3. A restore names one immutable snapshot and rejects objects absent from its manifest.
4. An expired snapshot enters a deletion queue; completion is recorded only after every governed key is confirmed absent.
5. Interrupted deletion is idempotent: replaying the job converges on the same verified state.
6. A download supplies the intended media type and a safe `Content-Disposition` value. MDN documents `inline`, `attachment`, and filename parameters; generate the header in the application instead of copying an untrusted filename.

The third test catches a costly design mistake. An endpoint that accepts only `tenantId` and selects the newest-looking prefix is not a selected-snapshot restore endpoint. Clock skew, partial uploads, or a retry can change what "newest" means. Require `tenantId`, `snapshotId`, and the manifest checksum.

Stop there.

I'm not sure a generic compliance checklist can settle a real cross-border deployment, because the answer depends on your data, contracts, subprocessors, account settings, and legal obligations. Resolve that uncertainty through documented privacy and security review of the exact configuration and data flow. An `EU` label in a dashboard isn't evidence by itself.

## Retention and deletion come before the cheapest quote

Retention is application policy expressed as dates and states. Record the document class, the event that starts its clock, who can place a hold, and what evidence closes deletion. A snapshot may contain catalog images, invoices, and merchant exports with different rules; don't pretend one bucket-wide duration is always precise enough. Split policy domains or store an object-level decision in the manifest. AWS documents object lifecycle management as rules that can transition and expire objects. That automation is useful, but the application still needs a ledger of intent: which tenant snapshot was eligible, which rule applied, whether a hold prevented action, and when verification completed. Treat lifecycle configuration as an executor of policy, not the only copy of policy. Keep backup expiry separate from account deletion. An ordinary job evaluates `deleteAfter` and any hold. Account deletion first blocks new writes, waits for in-flight writers to acknowledge the fence, freezes the governed inventory, deletes from that inventory, and records verification. Consider the race if those steps are collapsed into one callback: a merchant closes an account while a backup worker owns a queued upload; the deletion worker lists the prefix and completes, then the late object lands outside the inventory. Another scan is not the fix. Ordering is. Record each transition against the tenant and policy version so a retry resumes the same decision rather than taking a fresh view of a moving bucket.

The catch is that deletion guarantees can conflict with immutable retention. If policy requires a snapshot to remain undeletable for a fixed period, that configuration is not suitable for a promise of immediate erasure during the same period. Resolve the conflict before selecting an account configuration. Stick with region-specific deployments and separate control planes when a shared cross-region design cannot satisfy the approved data-flow map, even if shared infrastructure is easier to operate.

Cost gets one place in the decision: calculate stored byte-months, request volume, retrieval and transfer paths, minimum-duration effects, and operator time from your measured workload. Compare like-for-like configurations only after the compliance and restore gates pass. "Cheapest" without those inputs is an incomplete spreadsheet.

## A focused TypeScript retention boundary

Keep provider calls behind a narrow port. Domain code decides eligibility and verification; an adapter translates `delete` and `exists` into the candidate's supported interface. This preserves the experiment's meaning when protocol details differ.

```ts
interface SnapshotManifest {
  tenantId: string;
  snapshotId: string;
  deleteAfter: string;
  hold: boolean;
  objectKeys: string[];
}

interface ObjectPort {
  delete(key: string): Promise<void>;
  exists(key: string): Promise<boolean>;
}

type DeletionResult =
  | { state: "not_due" | "held" }
  | { state: "verified"; deletedKeys: number };

async function deleteExpiredSnapshot(
  store: ObjectPort,
  manifestKey: string,
  manifest: SnapshotManifest,
  now: Date,
): Promise<DeletionResult> {
  if (manifest.hold) return { state: "held" };
  if (now < new Date(manifest.deleteAfter)) return { state: "not_due" };

  for (const key of manifest.objectKeys) await store.delete(key);
  const remaining = await Promise.all(
    manifest.objectKeys.map((key) => store.exists(key)),
  );
  if (remaining.some(Boolean)) throw new Error("Deletion is not yet verified");

  await store.delete(manifestKey);
  if (await store.exists(manifestKey)) {
    throw new Error("Manifest deletion is not yet verified");
  }
  return { state: "verified", deletedKeys: manifest.objectKeys.length + 1 };
}
```

This example does not claim that one absence check proves physical erasure from every replica or backup. It proves the narrower application invariant that governed objects are no longer addressable through this boundary. Contractual deletion semantics and provider-held backups require separate evidence. Your mileage may vary with object count and consistency behavior, so test concurrency and verification timing rather than adding an arbitrary sleep.

For restore, read the named manifest, confirm its tenant and snapshot IDs, verify every fetched checksum, and publish only after all objects pass. Don't expose a half-restored directory. Stage it, validate it, then switch the application pointer in one controlled operation.

## Compare candidates with evidence, not logo columns

Put AWS S3, Cloudflare R2, Wasabi, and Bunny Storage on separate rows in your experiment, then fill the cells from current primary documentation, contracts, account configuration, and harness output. Do not assume "S3-compatible" means identical lifecycle behavior, authorization semantics, conditional requests, or tooling. Compatibility is a test scope, not a conclusion.

| Gate | Evidence to retain | Pass condition |
| --- | --- | --- |
| Tenant isolation | Denied cross-tenant read and list attempts | Every attempt is denied without object data |
| Placement | Approved architecture and configured locations | Actual data flow matches the reviewed map |
| Retention | Policy fixture, lifecycle setup, manifest ledger | Due snapshots progress; held snapshots remain |
| Deletion | Job records and post-delete checks | Every governed key reaches verified state |
| Restore | Named manifest, checksums, elapsed time | Selected snapshot is complete and unmodified |
| Download safety | Response headers and filename tests | Browser behavior matches attachment policy |
| Operations | Retry, credential rotation, and alert tests | Runbook meets the team's stated objective |

A candidate fails this deployment if it cannot produce evidence for a mandatory gate. It may still suit another system. Rejecting an account shape for a Europe-US document workflow is not a universal judgment about its product.

Test errors as first-class results. Record request IDs when available, classify authorization failures separately from missing objects, cap retries, and make deletion jobs resumable from the manifest ledger. Alert on the age of the oldest unverified deletion and oldest restorable snapshot, not merely worker process health. A green worker that makes no progress is still a failed backup operation.

## What should you measure before copying this storage choice?

Measure document size distribution, snapshots per tenant, changed bytes per snapshot, restore frequency, deletion queue age, verification latency, failed isolation probes, and complete selected-snapshot restore time. Run restores continuously in a non-production tenant. A backup format first tested during an incident is an archive of hope.

Also measure the human path. Can an operator identify one tenant's snapshots without seeing another tenant's metadata? Can the privacy owner trace an erasure request to verified jobs? Can an engineer rotate credentials without making old manifests unreadable? Those questions expose gaps that throughput charts miss.

Set the decision rule before the experiment: discard any candidate that fails isolation, approved placement, retention, verified deletion, or selected restore. For the remainder, choose the operating model with acceptable measured performance and total workload cost. Re-run the harness after policy, region, credential, adapter, or lifecycle changes.

No universal winner follows. There is only a configuration that passed disclosed gates for a defined e-commerce workload — and evidence you can inspect when the next retention request arrives.

## Sources

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
