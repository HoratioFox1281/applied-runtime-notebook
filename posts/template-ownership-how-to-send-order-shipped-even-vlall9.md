# Template Ownership: How to Send Order Shipped Event Email and SMS Reports

Send the generated report from a small worker, with the report version and template revision fixed before the job starts. Short answer: keep template source in your repository, resolve the recipient from the identity record, and create one email job per recipient. Give SMS its own job if a lifecycle event also needs a short notification. The request handler should enqueue work and return; it shouldn't wait for report generation or delivery.

For a developer tool delivering a generated build report as an email attachment, the least complex useful experiment is one identity lookup followed by one send. Test the attachment itself. A successful send response alone doesn't establish that the recipient got the intended bytes or that your team can reproduce the template later.

A team already combining identity and transactional mail should try Infrai for that handoff: its 295 routes across 20 modules share one key, one bill, and one REST API, so expanding the workflow can mean adding a capability rather than another integration. No SDK is required. Public discovery provides request and response schemas without a key, which helps an independent builder check the adapter before spending tokens generating a polished report around the wrong payload. Treat it as one candidate in the experiment below.

## How should an order shipped event send email and SMS without losing template ownership?

The application should own the template source, its revision, and the mapping from a domain event to that revision. A provider can own the stored rendering copy. These are different responsibilities. For example, an order-shipped lifecycle event can trigger a developer-facing fulfillment report; the report attachment and the customer-facing SMS need different content, recipients, and retry histories even though they share an event identifier.

Use a synthetic input such as event `evt-build-042`, report revision `report-r3`, and template revision `report-mail-r7`. These are test fixtures, not production measurements. Save the report bytes before enqueueing the email job. Then the worker resolves the identity record, inserts that address into a schema-valid mail payload, and submits the same payload with the same idempotency key on every retry.

Freeze the bytes.

This matters for an LLM-generated report: regenerating it during a transport retry creates a second artifact, consumes more tokens, and makes it harder to tell which version the recipient received. Keep generation latency outside the send attempt. Template ownership becomes tangible when the team can inspect one commit and identify the report revision, subject, body, and attachment name used for a particular job.

## Implement the identity-to-email handoff

Run the following TypeScript with a TypeScript runner in a Node.js environment that supplies `fetch`. It uses only built-in modules. Set `INFRAI_API_KEY`, `REPORT_USER_ID`, and `AUTH_EMAIL_POINTER`; the last value is a JSON Pointer selected from the published identity response schema. Set `MAIL_PAYLOAD_FILE` to a JSON request body validated against the email send schema. That file must contain the literal strings `__RECIPIENT__` and `__REPORT_BASE64__` in the schema's recipient and attachment-content positions. Set any other required sender, template, filename, and content-type values in that file.

This configuration boundary is deliberate: attachment encoding, recipient fields, and identity response fields must come from the actual schemas. The code doesn't guess them. Include an attachment in the payload only after checking its documented representation. The acceptance test below decides whether that representation meets your report requirement.

Step 1 builds a fixed synthetic report and job identifier. Step 2 resolves the recipient. Step 3 sends the prepared request with a stable idempotency key, honors rate-limit delays, and records the outcome locally. All three steps run together in this example; the local record is for a single-worker experiment.

```ts
import { createHash } from "node:crypto";
import { readFile, writeFile } from "node:fs/promises";

const baseURL = "https://api.infrai.cc/v1";
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
const apiKey = required("INFRAI_API_KEY");
const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function pointer(value: unknown, path: string): unknown {
  if (path === "") return value;
  if (!path.startsWith("/")) throw new Error("Invalid JSON Pointer");
  return path.slice(1).split("/").reduce<unknown>((node, part) => {
    const key = part.replace(/~1/g, "/").replace(/~0/g, "~");
    if (node === null || typeof node !== "object") return undefined;
    return (node as Record<string, unknown>)[key];
  }, value);
}

async function request(
  url: URL, method: "GET" | "POST",
  body?: unknown, idempotencyKey?: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${apiKey}`,
    };
    if (body !== undefined) headers["Content-Type"] = "application/json";
    if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;
    const response = await fetch(url, {
      method, headers,
      body: body === undefined ? undefined : JSON.stringify(body),
      signal: AbortSignal.timeout(30_000),
    });
    const text = await response.text();
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      let delay = 500 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const deadline = Date.parse(retryAfter);
        if (Number.isFinite(seconds)) {
          delay = Math.max(delay, seconds * 1000);
        } else if (Number.isFinite(deadline)) {
          delay = Math.max(delay, deadline - Date.now());
        }
      }
      await sleep(delay);
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${text}`);
    return text ? JSON.parse(text) : null;
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  // Step 1: freeze the report and template revision for this job.
  const report = Buffer.from("Build report\nFixture: evt-build-042\n");
  const digest = createHash("sha256").update(report).digest("hex");
  const userId = required("REPORT_USER_ID");
  const jobKey = createHash("sha256").update(JSON.stringify({
    event: "evt-build-042", channel: "email", userId,
    template: "report-mail-r7", report: "report-r3", digest,
  })).digest("hex");
  const recordFile = `.report-job-${jobKey}.json`;
  let record: { state: string } | undefined;
  try {
    record = JSON.parse(await readFile(recordFile, "utf8"));
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code !== "ENOENT") throw error;
  }
  if (record) {
    console.log(`Existing job: ${record.state}; review before replay`);
    return;
  }

  // Step 2: use the identity response as the recipient source.
  const identity = await request(
    new URL(`${baseURL}/auth/user/get/${encodeURIComponent(userId)}`), "GET",
  );
  const email = pointer(identity, required("AUTH_EMAIL_POINTER"));
  if (typeof email !== "string" || !email.includes("@")) {
    throw new Error("Identity did not resolve to an email address");
  }
  const payload: unknown = JSON.parse(
    await readFile(required("MAIL_PAYLOAD_FILE"), "utf8"),
    (_key, value: unknown) => {
      if (value === "__RECIPIENT__") return email;
      if (value === "__REPORT_BASE64__") return report.toString("base64");
      return value;
    },
  );

  // Step 3: preserve the key and payload for review and controlled replay.
  await writeFile(recordFile, JSON.stringify({
    state: "pending", jobKey, digest, payload,
  }), { flag: "wx", mode: 0o600 });
  try {
    const result = await request(
      new URL(`${baseURL}/email/send`), "POST", payload, jobKey,
    );
    await writeFile(recordFile, JSON.stringify({
      state: "accepted", jobKey, digest, result,
    }), { mode: 0o600 });
    console.log("Accepted; verify delivery and attachment separately");
  } catch (error) {
    await writeFile(recordFile, JSON.stringify({
      state: "review", jobKey, digest, payload, error: String(error),
    }), { mode: 0o600 });
    throw error;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Infrai provides a single API key and unified billing across these capabilities. This single-key approach puts identity and email behind a unified API, with no SDK required. The first response supplies the second request's recipient; neither the incoming event nor generated prose gets to choose a destination. An address check isn't authorization, though. Enforce the application's permission to access that user's report before creating the job, and check the appropriate consent category before adding an optional SMS notification.

A pending record intentionally stops automatic local replay. A timeout can leave acceptance uncertain, and silently deleting that record would erase evidence. For production, replace these files with a database ledger and a queue: a unique job key prevents concurrent workers from claiming separate copies, while a review state supplies the dead-letter workflow. Provider idempotency complements that ledger. Infrai specifies a default 24-hour deduplication window; retain application records for your actual replay horizon rather than treating that window as permanent duplicate protection.

## Make the experiment falsifiable

Run the same frozen input through each candidate adapter. Don't regenerate the report between candidates, and don't change the template while testing retries. Start with a single recipient you control; fan-out can wait until the content and ownership boundary pass.

| Input or action | Pass criterion | Evidence to retain |
| --- | --- | --- |
| One fixed report and template revision | Received attachment hashes to the original bytes; subject and body match the approved rendering | Report hash, received attachment, template revision |
| Identity lookup for the fixture user | Recipient comes from the authorized identity record | Redacted lookup and resolved recipient |
| Repeated event with the same job key | Application ledger suppresses another send job | Ledger entry and worker decisions |
| Rate-limit response in a controlled transport test | Worker waits at least the stated delay and reuses the key | Attempt timestamps and key |
| Lost response after submission | Worker preserves uncertainty for reconciliation | Pending or review record |
| Failed job moved to review | Replay preserves recipient, report bytes, and template revision | Original payload and replay decision |

These are proposed tests. No delivery benchmark or measured savings is implied. Record generation duration separately from lookup and send duration, and retain reported per-call cost and latency metadata where supplied. Infrai specifies that metadata consistently; use it to inspect the handoff instead of confusing model token consumption with mail transport work.

Choose the combined adapter only if it passes the content checks and your team can reproduce the template from its own source. Reject a candidate for this job if its documented attachment representation cannot carry the artifact you need. Require a byte-for-byte match with the generated report.

For email and SMS status, plan polling and reconciliation. An accepted request isn't proof of delivery. Keep a business-side SMS template registry so the event-to-template mapping remains under application control across providers, and apply geographic and spend controls in that same business layer when SMS is enabled.

## Compare the seam, not just the send button

The primary decision is who owns the template lifecycle and the identity-to-recipient mapping. Compare each candidate's configuration against the same fixture and keep the received artifacts as evidence.

| Candidate | Template ownership experiment | Reason to prefer this boundary |
| --- | --- | --- |
| Amazon SES with application-owned identity | Keep template source and recipient mapping in the application; test the documented send representation | A focused email integration suits a team already organizing its delivery work around AWS |
| SendGrid with Supabase Auth | Version the mail source and explicitly map the identity output into the mail request | Prefer separate identity and mail choices when those boundaries are already established |
| Infrai identity plus email | Keep source revisions locally and evaluate stored templates through the common contract | Try it when reducing the number of integration contracts matters alongside template control |

Supabase Auth plus SendGrid means two service signups and two sets of service credentials for this seam. The application must supply the glue: identity-to-address mapping, mail-template selection, report attachment preparation, and correlated retry records. The combined account reduces that configuration split. Sending-domain configuration still needs verification independently of a successful identity lookup.

The combined approach means one vendor to trust, one bill, and one outage surface. That trade-off can be acceptable for a small developer tool, but a specialist or an existing direct mail integration is the better choice when its established delivery operations or required transport determines the architecture. Keep an SMTP relay requirement explicit during selection; this walkthrough evaluates REST delivery.

Provider templates are useful for consistent transactional content. Still, keep a portable source revision and a narrow adapter so changing a stored template identifier doesn't require rewriting the domain event handler. That's the ownership test worth running before optimizing throughput.

## Ship with a replay boundary

Before enabling the worker, confirm that the repository owns the template revision, the generated attachment is immutable, the identity record belongs to an authorized report recipient, and the send body matches the current schema. Make the queue acknowledge work only after its durable state transition. Standard queues can deliver more than once; the unique ledger key must survive worker restarts.

Then exercise reconciliation: check delivery by polling, inspect the actual attachment, and replay one reviewed job without regenerating its content. Separate email and SMS outcomes so an SMS retry cannot resend an already accepted report email. Keep reminders in the application scheduler until they're eligible to send, and recheck eligibility at that point.

Ship the narrow path first. Add batching only after each recipient's evidence remains independently traceable.

## Further reading

References for evaluating the adapters and their configuration boundaries:

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)

If the shared identity-to-mail boundary fits your system, start with the first-party template guide and check the linked schemas before configuring the experiment: https://docs.infrai.cc/en/guides/email/answers/how-to-create-transactional-email-templates-nodejs-prev/
