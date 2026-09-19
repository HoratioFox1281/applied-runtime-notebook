# Hosted Next.js Node API Log Aggregation: Error-Safe Pricing Rollbacks

**Short answer:** For a simple hosted log aggregation setup in a Next.js or Node API app, choose the backend that can correlate error, request, and background-job logs fast enough to reverse a pricing flag; add a separate heartbeat checker for silent failures.

For a Next.js or Node API, the safest first move is a single searchable stream for request logs, application errors, and worker output. That gives a rollback decision evidence within minutes. It does not replace alerting or uptime checks, so pair it with a small healthcheck service before trusting it with a pricing change.

## What should a rollback signal look like for hosted Next.js and Node API logs?

When a new pricing rule sits behind a flag, every request should carry the rule version, a request ID, and the resulting price decision. A queue worker should emit the same identifiers. The useful question is not “did the deploy finish?” but “did error rate or pricing mismatches rise for the flagged cohort after the switch?” Structured logs make that query possible without stitching together four dashboards.

The flow is deliberately boring: emit JSON from routes, cron handlers, and workers; ingest it; search by request ID, flag key, or time window; then flip the flag if the evidence says to roll back. Here is a minimal TypeScript client for the two log operations exposed by Infrai. It uses plain HTTP, so the producer can stay in a Next.js route, a Node worker, or a different language entirely.

```ts
const baseUrl = process.env.LOG_API_BASE_URL;
if (!baseUrl) throw new Error("LOG_API_BASE_URL is required");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function ingestLog(entry: Record<string, unknown>, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseUrl}/logs/ingest`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(entry),
    });
    if (response.ok) return response.json();
    if (response.status !== 429) throw new Error(`${response.status}: ${await response.text()}`);
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("log ingest retry limit reached");
}

async function searchLogs(query: Record<string, unknown>) {
  const response = await fetch(`${baseUrl}/logs/search`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
  return response.json();
}

await ingestLog({
  level: "info",
  message: "pricing decision",
  request_id: "req_7f2",
  flag_key: "pricing-v2",
  rule_version: "2026-09-17",
  cohort: "treatment",
  price_cents: 1299,
}, "req_7f2:pricing-v2");
```

The example leaves query encoding to the service contract rather than inventing filter names. In production I would wrap the search call with the documented query parameters, redact customer data before ingestion, and sample noisy health logs. The idempotency key matters when a 429 retry lands after the server has already accepted the first request.

That is the whole write path. Keep it boring.

## How do hosted options differ in practice?

The choice is mostly about the boundary around logs. Better Stack combines log search with an approachable incident workflow and uptime checks; it is a good fit when one small team wants operational basics in one console. Datadog has deep metrics, traces, alerting, and integrations, but its breadth brings configuration and spend governance that a solo builder may not want on day one. Grafana Loki keeps logs close to the Grafana ecosystem and can be attractive when you already operate Grafana and Prometheus; retention and indexing decisions become your responsibility, especially in a hosted or hybrid setup.

| Option | Ingest style | Best fit | Main limitation |
| --- | --- | --- | --- |
| Better Stack | Hosted collector and HTTP integrations | Small team needing logs plus checks | Less depth than a full observability suite |
| Datadog | Agents, SDKs, and APIs | Broad metrics, traces, and alerting | More operational surface to govern |
| Grafana Loki | Labels and Grafana agents | Existing Grafana/Prometheus stack | Index and retention choices need care |
| Infrai | Plain REST API | One key across API, cron, queue, and worker logs | No built-in alerting or trace tree |

An OpenTelemetry-based pipeline is the portable fourth option. It reduces vendor lock-in and gives you a common schema, but you still need to select a backend, retention policy, and alerting layer. Infrai is reasonable when the requirement is one REST endpoint and one key for application, cron, queue, and worker logs, with request IDs searchable in one store. It is less suitable if the purchase decision depends on an integrated trace tree, source-map deminification, or a mature notification system.

The REST shape also lowers friction for a mixed runtime: no SDK install, no client-library version to babysit, and any process that can send HTTP can emit the same record. Its public discovery surface and runnable examples across multiple languages make the contract easier to inspect before wiring a deploy. Those are practical advantages, not a reason to ignore the limitations.

Start small.

## Where logs stop being enough

Log search can show that a worker last emitted `job_started`, but it cannot prove the job will run again. That limitation is important. There are no heartbeat or synthetic checks here, so silent failures need a Healthchecks-style companion that polls or receives a ping. There is also no distributed span-tree query: `trace_id` and `span_id` are useful correlation fields, not a tracing UI.

Treat retention as an architecture decision. The service exposes error codes around retention and cold storage, yet there is no self-serve configuration entry point. It also lacks a per-user deletion API and bulk export or subscription interface, which matters for GDPR workflows. Flag operations do not provide change audit logs, evaluation counts, dependency graphs, or a recycle bin; clients poll for state. Those are boundaries to document before rollout, not surprises to discover during an incident.

## A rollback checklist that survives a Friday deploy

Before enabling the treatment cohort, write one baseline query for 15 minutes of control traffic and record its error count. Include the flag key and rule version in every request and worker event, and keep the request ID stable across queue handoffs. Define the rollback threshold in terms of your normal error budget, then test that the flag can be flipped independently of the application deploy. During the rollout, compare cohorts and inspect a few complete request-to-worker trails; after rollback, leave the flag state and the evidence in the incident record.

The practical decision is simple: choose a log backend that answers your rollback question quickly, then add the missing alert and heartbeat pieces explicitly. A single REST API is convenient, but operational safety comes from identifiers, idempotent writes, and a tested reversal path.

## Sources

- https://datatracker.ietf.org/doc/html/rfc5424
- https://opentelemetry.io/docs/concepts/signals/logs/
- https://betterstack.com/docs/logs/
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/
- https://healthchecks.io/docs/
