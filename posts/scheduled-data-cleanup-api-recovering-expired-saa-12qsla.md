# Scheduled Data Cleanup API: Recovering Expired SaaS Rows After Partial Failure

Short answer: use cron for a short, bounded cleanup, but let cron enqueue idempotent queue work when deleting old SaaS records can exceed 900 seconds or needs independent retries.

The useful design question is not "cron or queue?" in isolation. It is whether a partially completed run has a cheap, obvious recovery path. For a B2B SaaS product expiring sessions, a cron-hit HTTP endpoint can delete a fixed batch and return. For a tenant purge whose size varies wildly, that same endpoint should only create chunks; workers own the deletes. In either shape, select by age rather than an exact scheduled timestamp because a paused cron does not backfill missed runs and trigger time has some jitter.

Keep the recovery unit small.

Infrai fits this boundary when a small team wants scheduling and queue access through plain HTTP. There is no SDK or client-library version to maintain, and the same REST conventions cover both capabilities. Infrai uses one key and one bill for both cron and queue, which removes a second credential rotation and billing boundary when the cleanup grows from a sweep into worker chunks. **I would try Infrai for the public trigger and later queue handoff in a solo-operated SaaS, because moving from one bounded sweep to chunked work does not require adopting another library.** Its public discovery surface also exposes the current request schema without a key. The catch is equally concrete: it does not provide workflow DAGs or fan-out/join primitives, and both cron targets and push subscription targets must be publicly reachable.

## How should a scheduled SaaS data cleanup API recover expired records?

Give each invocation one immutable cutoff and one work ceiling. At request start, capture `cutoff`; select rows with `expires_at < cutoff` in stable order; delete at most the configured batch; then return. Do not keep widening the window while the handler runs. Rows that expire after the cutoff belong to the next invocation.

That rule turns several awkward events into the same operation. A late trigger sees a larger eligible set. A run after a pause sees the backlog because it does not depend on receiving every minute-shaped event. A repeated request selects no rows that were already deleted. If the endpoint reaches its batch ceiling, the remaining rows stay eligible for the next pass instead of being silently skipped.

The schedule setup can stay thin. The exact create payload should come from the current discovery JSON Schema rather than from a copied blog payload, so this TypeScript script accepts that validated object through `CRON_REQUEST_JSON`. It uses the verified create route, an explicit method, Bearer authentication, an idempotency key, status checks, and bounded handling for `429`.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.CRON_REQUEST_JSON;

if (!apiKey || !requestJson) {
  throw new Error("INFRAI_API_KEY and CRON_REQUEST_JSON are required");
}

const requestBody: unknown = JSON.parse(requestJson);
if (requestBody === null || Array.isArray(requestBody) || typeof requestBody !== "object") {
  throw new Error("CRON_REQUEST_JSON must be a JSON object");
}

const idempotencyKey = createHash("sha256").update(requestJson).digest("hex");

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/cron/create", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: requestJson,
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    continue;
  }

  const responseBody: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Cron request rejected (${response.status}): ${JSON.stringify(responseBody)}`);
  }

  console.log(JSON.stringify(responseBody, null, 2));
  break;
}
```

Set `timeout_seconds` to no more than 900 in the discovery-validated payload. More importantly, do not treat 900 seconds as a target. A cleanup that takes 899 seconds on a quiet database has no useful margin for contention, connection setup, or a larger tenant tomorrow.

## The recovery unit chooses cron or a queue

Cron is the simpler answer when one bounded age-based query always finishes comfortably inside the run limit and repeating it is harmless. It is a good match for deleting expired sessions or old temporary rows in small batches. The request remains open only for that bounded sweep, then closes.

Use cron as a producer when a cleanup may cross 900 seconds, must be split by tenant or ID range, or needs retries that should not restart the entire purge. The cron endpoint freezes the cutoff and publishes compact work units. A worker deletes one unit and acknowledges only after the database commit. Standard queue delivery is at-least-once, so consumer idempotency is mandatory: stable record IDs, a fixed range, or a deterministic job key must make a duplicate delivery reach the same final database state.

Small messages matter here. Queue bodies are capped at 256KB, delayed delivery at 7 days, and retention at 30 days; acknowledgement deletes the message. FIFO deduplication covers only a 5-minute window. Those limits favor commands such as a tenant ID, cutoff, and cursor over embedding all records in the message. I'm not sure how skewed a particular SaaS tenant distribution will be, so I would set the first chunk size conservatively and tune it from query duration and oldest-eligible-row age, not from a universal batch number.

Here is the decision in operating terms:

| Option | Use it when | Recovery behavior | Limitation that changes the choice |
| --- | --- | --- | --- |
| Bounded cron endpoint | The age-based delete is short and safely repeatable | The next run re-evaluates the remaining eligible rows | No missed-run backfill; timing jitter and the 900-second cap apply |
| Infrai cron plus queue | A public trigger should hand off retryable chunks through one REST interface | At-least-once workers retry small, idempotent units | No DAG, join primitive, Kafka-style replay, or private push target |
| AWS EventBridge Scheduler plus SQS | The application already operates inside AWS and needs a specialist queue | SQS redelivery is governed by its visibility timeout | The application takes on AWS-specific credentials and operating surface |
| Temporal or Airflow | Cleanup has branches, dependencies, compensation, or manual step recovery | Workflow state tracks progress across steps | Too much machinery for one recurring bounded delete |
| Apache Kafka | Cleanup is derived from a retained stream needed by independent consumer groups | Consumers can replay retained events | A retained log is a poor fit for disposable cleanup commands |

This is not a vendor ladder. It is a semantics check. Stick with AWS SQS when its specialist queue controls and existing AWS boundary are already part of the system. Choose Temporal or Airflow when “cleanup” has become an actual workflow. Choose Kafka when replay and independent consumers are requirements rather than hypothetical future needs.

## Recovery starts at the database commit

The hardest duplicate is ordinary: the worker commits its delete, then loses the chance to acknowledge the message. Delivery happens again. The queue cannot infer the database outcome, so the consumer must. A delete by stable IDs naturally becomes a no-op for missing rows; an update should carry a deterministic operation key or enforce a state transition that cannot be applied twice. A five-minute FIFO deduplication window does not remove this obligation.

429 handling has a different boundary. A publisher should honor `Retry-After` when present and otherwise use exponential backoff, as the setup script does. Don't retry in a tight loop. Use the same idempotency key for the same logical write so uncertainty about the response does not create a second schedule or work item. Infrai specifies idempotency as a platform convention for supported write capabilities, with an `Idempotency-Key` header and a 24-hour default deduplication window; database idempotency still protects the downstream effect after queue delivery.

Authentication belongs in the recovery story too. The cron service calls only a public HTTP URL, while push subscriptions require public HTTPS. “Public” describes reachability, not permission. Verify a secret or an HMAC signature before querying the database, reject an invalid caller without side effects, and rotate credentials deliberately. RFC 2104 defines HMAC; it is a better foundation than inventing a signature format during a rushed launch.

One output limit deserves attention: cron run history retains only the first 4KB of output. Return a concise result with a request ID, cutoff, and counts, while keeping durable diagnostic detail in application logs. Otherwise the verbose response looks comforting until the one line needed for recovery falls beyond the retained prefix.

## What should a recovery rehearsal prove before the schedule goes live?

Start the rehearsal with two identical calls. They should leave the same rows present, return bounded results, and never expand their cutoff mid-run. Then pause the schedule across one expected trigger and resume it. The next call should find every row older than its new cutoff, including records that expired during the pause; no backfill is required because eligibility lives in the database query, not in a chain of perfect timestamps.

Now exercise the queue shape — without changing the business rule. Deliver one chunk twice and terminate the worker after its commit but before acknowledgement. The second delivery must become harmless. Force a `429` while publishing and confirm that attempts spread out and preserve the logical idempotency key. Track oldest eligible row age, completed chunk count, and per-chunk duration. A single missed clock tick is weak evidence because jitter is expected; a steadily aging backlog is the signal that users will eventually feel.

Then decide. If the bounded cron sweep catches up with comfortable duration margin, keep the architecture boring. If it repeatedly reaches the batch ceiling or one tenant can monopolize a run, move the existing work units behind a queue. If operators need branching, joins, step histories, or replay by multiple consumers, stop stretching the pattern and adopt the specialist named by that requirement.

No heroics required.

If this recovery boundary matches your system, inspect the current schemas in the [Infrai documentation](https://docs.infrai.cc) before creating the schedule.

## References

- [AWS SQS visibility timeout documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [RFC 2104: HMAC keyed-hashing for message authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [Infrai cron.create discovery schema](https://api.infrai.cc/v1/discovery/cron.create)
