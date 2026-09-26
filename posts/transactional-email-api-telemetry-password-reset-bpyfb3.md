# Transactional Email API Telemetry: Password Reset Signals Beyond Custom-Domain Acceptance

A transactional email API for a signup or password reset flow has one unforgiving job: deliver the link while it can still help. An HTTP `202` only says the service accepted a request; it does not prove that the message reached the inbox, arrived before expiry, or led to a verified account.

**TL;DR:** choose a transactional email API by running the same instrumented authentication-mail experiment against every candidate. Require a custom sending domain with aligned authentication, stable event delivery, suppression visibility, regional data-handling answers, and an idempotent Node.js boundary. Judge the result by verified signups and completed password recoveries, not opens. Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened a message, so open rate is a poor control signal for this job.

The simple approach is to compare SDK ergonomics and send one test message to a personal inbox. It feels productive and proves very little. The useful approach is narrower: freeze one message, one link policy, and one event schema; then test the failure path as hard as the happy path.

## How should a transactional email API report a password reset flow?

There are at least four distinct moments: the application accepts the signup, the provider accepts the send request, a receiving system accepts or rejects the message, and the user consumes the link. Collapsing them into one `sent` boolean hides the exact failures that matter.

For a customer-support team, that ambiguity turns into tickets such as “my link never arrived” and “the link is already invalid.” Support needs enough evidence to distinguish an address typo, a suppressed recipient, a delayed message, an expired token, and a second request that superseded the first. It does not need the token itself in a log.

The decision rule is blunt: **a provider acceptance response is an enqueue event, not a delivery result**. Store its message identifier beside an internal attempt identifier. Later events may advance the attempt to delivered, bounced, or complained, while the application’s own verification event records whether the link completed its job.

Open tracking does not close this gap. Mail Privacy Protection downloads remote content in the background and prevents a sender from seeing whether the recipient opened the message. Link consumption is the cleaner application signal, provided the link handler treats automated scanners and actual account verification carefully. A GET can display a confirmation screen; the state-changing confirmation can require an explicit POST. That keeps a security scanner from consuming a one-time action merely by inspecting the URL.

## Keep the Node.js boundary boring

The application should own the authentication workflow and expose a small mail port. The provider adapter may translate request fields and webhook payloads, but it should not decide token lifetime, account state, or retry policy. This keeps a future migration contained without pretending providers behave identically.

Here is the shape I would trial. The 15-minute expiry is an example product policy, not a claim about an industry default.

```ts
type AuthMail = {
  attemptId: string;
  recipient: string;
  verificationUrl: string;
  expiresAt: Date;
};

type AcceptedMail = {
  attemptId: string;
  providerMessageId: string;
  acceptedAt: Date;
};

interface TransactionalMailer {
  sendSignupVerification(message: AuthMail): Promise<AcceptedMail>;
}

async function beginSignup(
  email: string,
  mailer: TransactionalMailer,
  now = new Date(),
): Promise<void> {
  const attemptId = crypto.randomUUID();
  const expiresAt = new Date(now.getTime() + 15 * 60_000);
  const token = await issueSingleUseToken({ email, attemptId, expiresAt });

  await savePendingAttempt({ attemptId, email, expiresAt });

  const accepted = await mailer.sendSignupVerification({
    attemptId,
    recipient: email,
    verificationUrl: `https://accounts.example.com/verify?token=${encodeURIComponent(token)}`,
    expiresAt,
  });

  await recordMailAccepted(accepted);
}
```

The focused detail is `attemptId`. It is safe to place in provider metadata and lets the webhook consumer correlate an event without logging an email token. The send operation should also carry an idempotency key when the candidate API supports one. If it does not, the adapter still needs local deduplication because a network timeout leaves the caller unsure whether the remote system accepted the first request.

Do not retry every error. A timeout or rate-limit response may justify a bounded retry with jitter. A syntactically invalid address will not improve on attempt three. A permanent bounce should update suppression state and give support a usable reason category; repeatedly sending to it damages the signal you are trying to measure.

## The domain setup is part of the product test

A custom From domain is not a cosmetic checkbox. The trial should require the exact subdomain intended for production, documented DNS records, and a repeatable verification process that another engineer can inspect. SPF authorizes sending infrastructure for a domain. DKIM attaches a cryptographic signature. DMARC publishes a policy and reporting mechanism, then evaluates identifier alignment with the visible From domain.

This is where a “five-minute setup” claim stops being useful. DNS publication, verification feedback, key rotation, and alignment diagnostics affect operations long after the first send. Record the expected selectors and ownership in deployment documentation. Keep secrets out of DNS notes and repositories.

DMARC deserves a staged rollout. RFC 7489 defines `p=none`, `quarantine`, and `reject`, and its reporting model is meant to provide visibility into authentication results. Moving directly to enforcement without understanding every legitimate sender on the domain can reject wanted mail. Start by observing reports, inventory authorized sources, correct alignment, and then change policy according to evidence.

One trade-off is easy to miss: using a separate transactional subdomain limits the blast radius and makes ownership clearer, but it adds DNS and reputation-management work. For a solo builder, I would accept that extra surface because authentication mail and marketing mail have different urgency, consent, and failure-handling paths. The boundary pays for itself during an incident.

## Run an experiment that can fail honestly

Use a fixed corpus rather than whichever addresses happen to be nearby. It should cover mailboxes your customers actually use, plus controlled cases for a hard bounce, a complaint workflow where available, duplicate submission, webhook replay, delayed event arrival, and an expired link. Do not manufacture a benchmark number from this test. Preserve raw timestamps and compare candidates under the same conditions.

For each attempt, capture these facts:

- internal attempt ID, template version, sending domain, and request time;
- provider acceptance ID and acceptance time;
- normalized delivery, deferral, bounce, complaint, and suppression events;
- webhook receipt time, signature-validation result, and replay outcome;
- verification completion time or expiry, without storing the token.

Then calculate two separate distributions: acceptance-to-delivery-event latency and acceptance-to-verification latency. The first describes the mail path as reported by the provider. The second describes the user outcome. Neither should be silently substituted for the other.

Webhooks will arrive late, out of order, or more than once in any design that assumes at-least-once event handling. Verify signatures against the raw request body, reject stale timestamps according to the provider’s documented scheme, persist event IDs, and make state transitions monotonic. A late “delivered” event must not erase a later permanent bounce merely because it was processed last.

Small test. Big consequence.

Run the same exercise in a staging environment and in a tightly controlled production canary, since receiver behavior depends on real sending infrastructure and domain history. Before increasing volume, define the rollback trigger and confirm that support can locate an attempt without database access or exposure to secrets.

## Use an evidence sheet, not a winner label

The final choice should be traceable to the workload. A compact evidence sheet works better than a feature-count score because several capabilities are gates, while others are preferences.

| Decision area | Evidence to collect | Reject or investigate when |
| --- | --- | --- |
| Domain authentication | SPF guidance, DKIM selectors, DMARC alignment result | The visible From domain cannot align or diagnostics are opaque |
| Event integrity | Signed webhook docs, replay test, stable event IDs | Authenticity cannot be verified or duplicates corrupt state |
| Failure handling | Bounce and suppression categories, retry semantics | Permanent and temporary failures are indistinguishable |
| US/EU operations | Written region, subprocessors, retention, and transfer terms | Sales language replaces contract-level answers |
| Portability | Export path for suppressions and event history | Operational state exists only in a dashboard |
| Cost control | Invoice units, retry effects, log retention, support tier | The likely bill cannot be reproduced from measured usage |

Regional availability is not the same as regional data handling. Ask where message content, recipient addresses, event logs, backups, and support access are processed; how long each is retained; which subprocessors participate; and what contractual transfer mechanism applies. The right answer depends on the SaaS product’s own legal obligations, so this is a documentation and counsel question, not a badge-comparison exercise.

Price belongs on the sheet, but it should not dominate it. Count the usage units produced by the experiment, include required retention and support, and model a normal month plus a retry-heavy month. A cheaper accepted request is expensive when support cannot explain failures or when suppression state cannot move with the application.

The choice is ready when every hard gate has evidence and the remaining trade-offs are explicit. Before copying this method, measure your real verification window, mailbox mix, peak send rate, duplicate-request rate, time to a trustworthy delivery event, completion rate, and support contacts per failed attempt. Those numbers determine which boundary matters most.

## Sources

References:

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
