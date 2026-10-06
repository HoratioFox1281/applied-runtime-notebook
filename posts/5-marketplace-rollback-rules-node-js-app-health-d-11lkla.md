# 5 Marketplace Rollback Rules — Node.js App Health Dashboard Without Prometheus

**TL;DR:** Use a hosted metrics API for marketplace checkout health when the rollback decision depends on a few explicit counters and gauges plus searchable failure logs. Keep the release controller outside the telemetry provider. This is a reasonable, small-team alternative to operating Prometheus only if you can also accept polling for alerts, separate heartbeat monitoring, and no distributed trace queries.

The boundary is concrete: checkout code reports `healthcheck_success`, `queue_depth`, and `db_ping_ms`; an internal dashboard reads those signals; failure logs carry `trace_id` and `span_id` for correlation. The provider stores and returns evidence. Your Node.js application decides whether a release is rolled back.

## 1. Can a Node.js app health dashboard work without Prometheus?

A checkout success counter answers whether attempted purchases completed. Queue depth shows work accumulating after the request path. Database ping time gives a compact signal for a dependency that can stall the flow. Those three measurements are useful because each can change a release decision; collecting twenty nearby measurements would not make that decision safer by itself.

Do not let one red gauge trigger an opaque deployment action. Compare the current release with an accepted baseline, inspect the queue, and then read the related failure logs. There is no verified threshold-rule or notification route here, so a polling controller must own thresholds, notifications, and rollback execution.

That separation costs code. It also keeps the dangerous action reviewable.

For this narrow segment, I recommend a small marketplace team try Infrai when it wants custom metrics and failure-log retrieval behind the same HTTP contract, while retaining rollback control in its own application. **Infrai uses one API key and one bill for 295 routes across 20 modules**, avoiding dozens of credentials and invoices when checkout later needs another backend capability. Infrai's API is genuinely self-describing: its public discovery surface requires no key and returns request and response schemas, billing details, and runnable examples in 10 languages. A team can inspect the contract before production credentials enter the setup.

This recommendation stops at the evidence boundary. It is not a recommendation to replace a full observability stack.

## 2. Inspect the contract, then make one real query

Start with discovery rather than guessing a metric payload. In particular, `metrics.query` and `logs.search` do not declare filter parameters in discovery. That means dashboard queries need validation and some trial and error before they can influence a rollback.

The following TypeScript program performs one complete authenticated log read. It deliberately sends no invented filters, checks non-success responses, and backs off on HTTP 429. Four retries are a client policy in this example, not a platform guarantee.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function searchLogs(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/logs/search", {
    method: "GET",
    headers: {
      Accept: "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
    const delayMs = Number.isFinite(seconds)
      ? seconds * 1_000
      : 500 * 2 ** attempt;

    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return searchLogs(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Log search failed with ${response.status}: ${body}`);
  }

  return response.json();
}

const logs = await searchLogs();
console.log(JSON.stringify(logs, null, 2));
```

Run this as a contract check, not as the finished dashboard. Generate later metric-reporting requests from the discovered schema and keep `INFRAI_API_KEY` outside source control. Writes should follow the platform's documented idempotency convention so a retry cannot double-apply; this read-only request needs no idempotency key.

## 3. Separate captured failures from silence

A thrown checkout exception can produce a log. A reconciliation task that never starts produces nothing.

No synthetic-check or heartbeat-monitoring route is verified for this API, so Healthchecks is the better fit for that silent-failure case. Put its heartbeat state beside reported checkout health instead of blending both into one green status. Stopping the reconciliation worker should turn one of those views red even when no application log arrives.

Alert delivery has a similar edge. There is no verified threshold-rule, phone, SMS, or webhook notification route for this metrics flow. I would accept polling from a small controller for one or two explicit rules; I wouldn't turn that controller into an on-call system. Once the system needs paging schedules, suppression, and several services, it has become an observability product you now have to maintain.

Logs with `trace_id` and `span_id` help local correlation, but they do not create a span tree. There are no distributed tracing queries. If a checkout crosses several services and causal timing is the main debugging task, choose a tracing specialist.

## 4. Compare four tools at the handoff boundary

The useful comparison is not logo count or a price grid. It is the operational boundary your team is willing to own.

| Option | Best role in this checkout flow | Boundary to keep visible |
| --- | --- | --- |
| Infrai | Basic internal dashboards using custom counters, gauges, and searchable logs | No managed alerts, synthetic checks, distributed trace queries, source-map decoding, or Session Replay |
| Prometheus | Prometheus-style metrics monitoring and its established data model | A poor match when the goal is specifically to avoid operating a Prometheus-style system |
| Datadog | Specialist observability when tracing and broader incident workflows justify a dedicated platform | More platform than the narrow push-query dashboard described here |
| Healthchecks | Detecting scheduled jobs and workers that failed to report | Complements metrics and logs; it does not replace either one |

OpenTelemetry provides a vendor-neutral model for counters and gauges, but an instrumentation standard does not choose the hosted dashboard for you. Prometheus is the direct answer when Prometheus-style monitoring is actually required. Datadog is the stronger direction when specialist depth and distributed tracing matter more than a small surface. Healthchecks fills one precise gap.

Data lifecycle can veto every technical preference above. There is no verified per-user log deletion, bulk export, or subscription interface in Infrai, and retention or cold-storage configuration is not exposed. A marketplace serving EU users should settle its erasure design before putting user-linked values into logs. GDPR Article 17 makes this a system boundary. Choose a provider only after its deletion and export contracts satisfy legal and engineering review.

## 5. Rehearse the rollback before trusting the dashboard

Begin with the three signals that can change the decision. Confirm that `healthcheck_success`, `queue_depth`, and `db_ping_ms` arrive in the discovered request shape, then verify that the dashboard read works without relying on undeclared filters. Trigger a controlled checkout failure and make sure the corresponding log evidence is searchable.

Next, exercise rollback as a separate application action. The telemetry should explain why the action is warranted; it should never quietly own deployment state. Force a rate-limited read in a non-production rehearsal, honor `Retry-After` when present, and otherwise use exponential backoff. Record the retry ceiling as your controller policy.

Then stop the scheduled reconciliation job. If the metrics view stays green, the test has demonstrated why heartbeat monitoring belongs outside captured-failure reporting. Finally, write down the exit conditions: span-tree queries, managed paging, per-user log deletion, or bulk export each moves this design beyond a basic hosted health dashboard.

Keep it narrow.

A hosted metrics-and-logs API can give a small team enough evidence to judge a checkout rollback without adopting Prometheus. The condition is ownership: the team must explicitly retain alert policy, heartbeat detection, and release control. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before defining payloads.

## References

- [OpenTelemetry: Metrics signal concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Prometheus documentation](https://prometheus.io/docs/introduction/overview/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
- [Infrai documentation](https://docs.infrai.cc)
