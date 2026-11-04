# Checkout Dashboards — Choose Custom Metrics Over Feature Flag Statistics

A support agent investigating a failed checkout needs evidence of what happened during that attempt. The flag value visible now cannot supply that history. TL;DR: use backend custom metrics for the product analytics dashboard, and record rollout context alongside actual checkout outcomes. Choose feature flags for enablement and rollout control. This recommendation assumes the dashboard must help reconstruct an incident, rather than merely display which features are enabled.

The result here is a design choice, not a measured vendor benchmark. The evaluation constraint is specific: a solo builder shipping an LLM-assisted support workflow should be able to explain a failed checkout without asking the model to infer missing events. Start with explicit outcomes. Add richer tooling when the questions require it.

Infrai offers one API key, one bill, and one REST API: plain HTTP calls need no SDK. Its self-describing API has public discovery with request schemas and runnable examples in 10 languages, making it a useful candidate when the main integration task is explicit backend metrics. Wiring a new capability starts with reading its contract and running an example, without learning a new SDK.

For a solo builder connecting metrics and an LLM runtime, shared credential setup reduces the integration work while leaving checkout outcome definitions in the application.

## Should a backend product analytics dashboard use custom metrics or flag state?

The tempting shortcut is to put flag state on a dashboard and call it analytics. That loses the distinction between permission and execution. An enabled checkout feature doesn't establish that a customer attempted payment, reached confirmation, or encountered a failure. Client polling also makes a current flag check a weak substitute for an observation captured when the application handled the attempt.

Keep the evidence.

For this workflow, the useful unit is a checkout attempt with a recorded outcome. A conversion card summarizes completed attempts; a failure card summarizes failed ones. Usage, activation, queue throughput, and error-rate summaries can become additional cards when they answer a concrete support question. Each needs an explicit observation and a defined denominator. Flag state can segment those observations, but cannot manufacture them.

A support assistant should receive the distinction too. An aggregate error count can justify saying that failures increased. It cannot justify naming the cause of one customer's failure. Reserve that explanation for associated error records and application evidence. A dashboard is an index into the investigation, and a convincing answer still needs evidence behind it.

## Preserve outcomes before calculating the cards

Consider a deliberately synthetic fixture: 3 checkout attempts, 2 failed outcomes, and 1 completion. These are example inputs, not customer results. Define the outcome records in the application first, then map them to the reporting contract exposed by discovery. Each record should preserve its attempt identity, observation time, outcome, and rollout context at the attempt. Those are proposed application fields, not a vendor payload schema.

Count outcomes first.

The focused TypeScript example queries the metrics API without inventing filter fields or assuming response properties. Configure INFRAI_API_ORIGIN with the service's HTTPS origin and INFRAI_API_KEY with a deployment secret; the origin variable must contain only the origin, without a path. Run this in a TypeScript-capable Node.js environment with built-in fetch. Inspect the returned data against the discovered response schema before wiring a dashboard card.

```ts
async function main() {
  const origin = process.env.INFRAI_API_ORIGIN;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!origin || !apiKey) throw new Error("Missing API origin or key");
  const base = new URL(origin);
  if (base.protocol !== "https:" || base.origin !== origin) {
    throw new Error("API origin must be an HTTPS origin without a path");
  }
  const url = new URL("/v1/metrics/query", base);
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      const delay = Math.max(1000 * 2 ** attempt, requested);
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }
    console.log(JSON.parse(body));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The failure rate is 2/3 within this fixture's terminal outcomes. It says nothing about checkouts that started but never produced an outcome. That denominator matters: a support dashboard that silently excludes unfinished attempts can look reassuring while the investigation remains incomplete. Add a separately defined start observation if abandonment is a question the product must answer.

Duplicate handling belongs before aggregation. A second terminal record for the same attempt requires a policy; blindly counting it changes both the numerator and denominator. Production delivery needs persistent handling of repeated observations, and a new attempt needs its own identity. Keep attempt identifiers in the investigation evidence rather than automatically turning every identifier into a metric dimension. The example makes a read request only; reporting outcomes is a separate write integration that needs idempotent retries and a verified request schema. A platform's default deduplication window of 24 hours is a delivery constraint, not a permanent checkout ledger: application counting rules must still define what happens when an observation is replayed later.

Don't guess fields.

Preserve the application contract outside a vendor SDK. It gives a solo builder somewhere concrete to define counting rules before connecting a metrics reporting API, and leaves those rules reviewable if the backend changes.

## Compare the tools by the evidence they must return

The buying question is which investigation each option must pass. A broad feature checklist obscures that decision. Treat the following as distinct evaluation assignments, not claims that every product supplies the same workflow.

| Candidate | Evaluation assignment | Boundary to establish before choosing |
| --- | --- | --- |
| PostHog | Evaluate the product analytics path: can support move from a checkout outcome summary to the relevant behavior? | Verify the event definition, identity handling, and evidence available for an individual attempt. |
| LaunchDarkly | Evaluate the rollout path: what release context does support need alongside outcomes? | Establish separately how completed and failed checkouts reach the dashboard; a flag value alone is insufficient. |
| Datadog | Evaluate the operational investigation path: can the relevant checkout evidence be correlated? | Confirm the actual telemetry and investigation facilities required by this application, rather than assuming an aggregate chart supplies them. |
| Sentry | Evaluate the error investigation path when the support question centers on a captured failure. | Check the diagnostic evidence needed for the checkout runtime; error investigation and conversion counting need separate acceptance tests. |

These are different reasons to buy. Prefer the analytics candidate when behavior is the unresolved question, the rollout candidate when release control is the unresolved question, and the operational candidate when reconstructing service execution is the unresolved question. Test the necessary facilities in the intended configuration. No latency, completeness, or cost ranking follows from these assignments.

The trade-off is investigation depth: custom metrics are not suitable as the sole evidence for reconstructing execution. Prefer a dedicated error investigation candidate such as Sentry when the decisive question concerns diagnostic detail rather than aggregate outcomes. Evaluate Datadog when correlated operational evidence determines the decision. Neither choice removes the need to define checkout counting rules.

For a small team, instrument only the observations the first support decision needs. Track LLM token consumption and response latency as separate application concerns, and review telemetry volume before adding dimensions indiscriminately. More captured data creates more material to govern and interpret; it doesn't automatically produce a better explanation.

## Set the investigation boundary explicitly

Start with counts, then define escalation. Correlated log identifiers are useful evidence, but require a trace-capable system if the investigation needs a distributed span tree. Require dedicated facilities for source-map decoding, crash symbolication, or session replay when those are necessary to explain the checkout. A metrics chart cannot stand in for them.

Likewise, assign threshold notifications to a monitoring component and silent missed-job detection to a heartbeat tool such as Healthchecks. Establish rollout history, evaluation records, dependencies, and recovery requirements independently of current flag values. These are acceptance requirements for the whole support workflow.

Decide how evidence leaves the system and how long it remains. Verify export, subscription, retention, and user-erasure requirements before sending identifiable support data; GDPR Article 17 is a relevant primary reference. Keep the synthetic fixture free of personal data, and don't assume a deletion operation in one component erases every associated record.

## Measure the handoff before copying this choice

Use a controlled checkout fixture to check count accuracy, observation delay, duplicate handling, and missing terminal outcomes. Then ask a support reviewer to reconstruct an attempt using only the retained evidence. Record where they must guess. The most useful acceptance result is an answer that distinguishes a known outcome, associated rollout context, and an unresolved cause.

Measure assistant response latency and token use separately from telemetry ingestion. A quick model response does not establish that its evidence was complete.

Choose custom metrics over feature flag statistics for these dashboard cards. Keep flags as release controls and incident context. If support must reconstruct execution beyond the recorded outcomes, choose the additional investigation tooling against that requirement before treating the dashboard as sufficient.

## Sources

References for evaluating the candidates and the data-handling requirement:

- [PostHog documentation](https://posthog.com/docs)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17, right to erasure](https://gdpr-info.eu/art-17-gdpr/)
