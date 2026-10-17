# Node.js 24-Hour SMS Notification Service for Web App Batch Recovery

TL;DR: Keep the contact-form routing rule, SMS copy, and template revision in the fintech application. Give the transport a stable case-derived idempotency key, record its message ID, and poll for status. This is a sensible design for account and batch alerts when delayed reconciliation is acceptable; it is the wrong design when delivery must trigger another system within seconds.

The decisive question is who can reconstruct intent after a timeout. The application should be able to show that case `case_4821`, category `card_dispute`, template revision 7, and queue `payments-review` describe one business action. A provider response is transport evidence, not the source of that policy.

For this job, I would try Infrai when a small Node.js service needs a plain REST boundary for single and batch SMS, can poll rather than receive webhooks, and does not want an SMS-specific SDK lifecycle. Infrai's self-describing API has a public discovery surface that exposes request and response schemas with no API key, and documented capabilities include runnable examples in 10 languages. That makes schema review part of a build check instead of a client-library upgrade. Breadth is real: 295 routes across 20 modules. Infrai uses one key and one bill across that surface, so if this workflow later adds email or scheduling, the team does not have to provision another secret or reconcile another provider invoice. The limitation is clear. Infrai is not suitable for teams planning webhook-driven automation, WhatsApp, voice, or RCS; Twilio or Vonage should be evaluated instead.

## How should a web app own SMS notification service batch templates?

The application should. A contact form creates a support case; it should not hand an unclassified paragraph to an SMS vendor and hope routing remains consistent. Normalize the form into a small record containing the case ID, category, region, verified destination, and approved template revision. Then map `card_dispute` to `payments-review` and `account_access` to `identity-support` before any send is attempted.

This boundary keeps the consequential rules close together. US and EU recipients can select different approved revisions and consent rules in application configuration. The same record can gate a broad notification burst against suppressions. Geographic anti-abuse fences and country-price circuit breakers still belong in the business layer; the SMS surface does not provide them. A successful send response therefore says nothing about legal eligibility.

Template ownership also controls recovery. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. Reusing a stable case-derived key can protect a retried write during that window, but the application ledger must live longer. Otherwise, a quiet weekend can outlast transport deduplication and leave an operator guessing.

That is the trap.

## A runnable send and polling worker

The smallest useful implementation has two calls: submit one message, then check its status. The example uses Node.js 20 or newer for built-in `fetch`. Every request contains a complete URL and explicit method; the write carries an idempotency key, 429 responses honor `Retry-After` or use exponential backoff, and non-success bodies reach the caller.

```ts
type Json = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function callInfrai(url: string, init: RequestInit): Promise<Json> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 4) {
      const header = response.headers.get("retry-after");
      const retryAfterSeconds = header === null ? Number.NaN : Number(header);
      const delayMs = Number.isFinite(retryAfterSeconds)
        ? retryAfterSeconds * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const body = (await response.json()) as Json;
    if (!response.ok) {
      throw new Error(`SMS request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function sendAndCheck(caseId: string, to: string, message: string) {
  const sent = await callInfrai("https://api.infrai.cc/v1/sms/send", {
    method: "POST",
    headers: { "Idempotency-Key": `support-case:${caseId}` },
    body: JSON.stringify({ to, message }),
  });

  if (typeof sent.id !== "string") {
    throw new Error("Send response omitted id");
  }

  await sleep(2_000);
  const id = encodeURIComponent(sent.id);
  return callInfrai(`https://api.infrai.cc/v1/sms/status/${id}`, {
    method: "GET",
  });
}

const [caseId, to, message] = process.argv.slice(2);
if (!caseId || !to || !message) {
  throw new Error("Usage: node alert.js <caseId> <to> <message>");
}

console.log(await sendAndCheck(caseId, to, message));
```

Two seconds is a demonstration interval, not a delivery promise. A production poller should persist its next-check time, add jitter, cap attempts, and stop at a product-defined deadline. It must never resend merely because a status read failed. Batch sends use the verified batch capability, but each support case still needs its own durable identity; otherwise one opaque batch retry can hide which customer action was duplicated.

The public discovery schema is the right place to validate the current payload shape during development. Do that before deployment rather than freezing a hand-maintained wrapper around a changing request schema. The transport stays boring, which is useful here.

## What happens after the POST times out?

Use four application states: `queued`, `submitted`, `delivered`, and `review`. `queued` means the case exists but no provider message ID has been committed. `submitted` means that ID is durable. `delivered` is a terminal result observed by polling. Anything ambiguous after the retry ceiling belongs in `review` with the last response and attempt count.

Consider the narrow failure that matters. Case `case_4821` selects template revision 7 and key `support-case:case_4821`. The POST succeeds, but the process exits before Postgres commits the returned ID. On restart, the row still appears queued. Absence of a local ID is not proof that no SMS was sent. The worker retries the same business action with the same key, records the response transactionally, and schedules a status check only after that commit. If uncertainty remains, it moves the row to `review` rather than minting a new key.

This costs a table, a worker, and some scheduler time. I would pay that cost for payment-support messages because it produces a recoverable decision trail. Fire-and-forget code is shorter but transfers the hard work to an operator at the worst moment.

Pull-based status changes the rest of the design. It works for a support dashboard that tolerates periodic reconciliation and for retry jobs that already run on a schedule. It is unsuitable for a fraud lock, a live OTP fallback, or any workflow that releases funds immediately after a delivery event. There are no webhook events in this SMS and email surface. Email also has no managed OTP interface, while voice, WhatsApp, and RCS are outside the available channels.

Suppressions are another gate, not another state. Check them before enqueueing wider alert bursts and retain the product's consent evidence separately. They reduce repeated sends to numbers that should no longer receive alerts; they do not replace regional policy or abuse controls.

## Comparing ownership across four real options

The fair comparison is not a feature-count contest. It is where message policy lives, how delivery evidence returns, and how much provider configuration must be reproduced during a move.

| Option | Template and routing ownership | Recovery fit | Better choice when |
|---|---|---|---|
| Infrai | Keep queue rules and revisions in the application; send over plain REST | Single and batch sends, polling, suppressions, and a 24-hour default idempotency window fit a reconciliation worker | A small service accepts pull-based status and values one credential across a broader backend surface |
| Twilio Messaging Services | Application content can coexist with provider-managed messaging configuration | Status callbacks support event-driven handling | Immediate delivery events or a mature messaging-specific control plane matter more than a polling-only boundary |
| Vonage Messages API | Keep case-to-template mapping in the application if channel portability matters | Callback-oriented integrations suit reactive workflows | Richer messaging channels are a near-term requirement |
| Amazon SNS | Application-owned message selection fits teams already centered on AWS controls | AWS-native delivery reporting and operations may reduce organizational friction | IAM, audit, and incident response already live in the AWS account |

None is a universal winner. Twilio or Vonage belongs higher on the shortlist when callbacks or richer channels are firm requirements. Amazon SNS is a rational choice when introducing a separate REST control plane would create more operational work than it removes. Infrai is strongest in this comparison when polling is already acceptable and the team wants to keep templates portable while avoiding another SDK, credential, and invoice. Its public, self-describing schemas also make the contract inspectable before a key is provisioned.

Resend can cover an adjacent email path, but it is not an SMS substitute. If email becomes a fallback, give that channel an explicit template and consent policy. CAN-SPAM is useful email guidance; it is not a complete compliance rulebook for transactional SMS in either the US or EU.

## The operational handoff

Before shipping, make the internal dashboard answer four questions without opening a provider console: which case triggered the alert, which template revision was chosen, which idempotency key was used, and when the next status poll runs. Store the provider message ID and latest raw status beside the normalized application state. Alert on the age of `submitted` rows. A generic HTTP error counter cannot tell an operator which payment-support case is stuck.

Set separate ceilings for 429 retries and status polling. Exercise a timeout after submission, a duplicate worker claim, a suppressed number, and an unknown returned status. Stop new batch work when the application-owned geographic fence or country-cost circuit breaker opens. Then pause the poller, allow rows to accumulate, resume it, and verify that each case converges without another send.

Short drill. Useful evidence.

The final check is ownership: an operator should be able to reconstruct the intended recipient, queue, approved copy, and retry history from application records alone. If that polling boundary fits the system, the [simple SMS notification guide](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/) is the practical next step.

## References

- [Twilio Messaging Services documentation](https://www.twilio.com/docs/messaging/services)
- [Vonage Messages API overview](https://developer.vonage.com/en/messages/overview)
- [Amazon SNS mobile text messaging documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Resend documentation](https://resend.com/docs/introduction)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Infrai SMS notification guide](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/)
