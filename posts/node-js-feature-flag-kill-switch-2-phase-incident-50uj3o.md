# Node.js Feature Flag Kill Switch: 2-Phase Incident Response for Delivery Alerts

Short answer: put one server-authoritative kill switch in front of delivery-notification dispatch, and make disabling it a 2-phase operation: record a versioned decision, then verify that workers have observed that version. A boolean endpoint alone can return 200 while stale Node.js processes keep sending. The useful API is still small, but its success condition is operational: broken dispatch stops, queued work remains recoverable, and the rollback can be reversed without redeploying.

For a logistics service, optimize for rollback safety before rollout flexibility. A delayed SMS is inconvenient; duplicate failure notices can confuse a recipient and make an already late delivery harder to resolve. Keep the first implementation boring: one flag, three runtime states, a monotonic version, a short local cache, and an alert that watches business outcomes rather than HTTP status alone.

## How should a feature flag kill switch contain an incident?

The switch should guard the side effect, not the route that accepted the work. An API may receive a `delivery.failed` event, validate it, and place it in a queue. A worker later chooses whether to send the notification. Put the check immediately before that send. This preserves the event for replay while preventing the risky action.

That boundary matters during an incident. If the switch rejects ingestion, upstream systems retry and create pressure in the least convenient place. If it only hides a user-interface control, background workers continue. Guarding dispatch gives the operator a narrow blast radius: delivery tracking continues, failure records remain available, and only customer notification pauses.

That is the trap.

Use three states: `enabled` permits dispatch, `shadow` records what would have happened without performing the side effect, and `disabled` blocks dispatch while recording the reason. `shadow` is the recovery step. After the notification fix is deployed, it lets the team inspect eligibility decisions before restoring sends.

## A small Node.js control path

This example uses process memory so the contract is visible without a framework or database. In production, the control record needs shared durable storage and authenticated operator access. Workers may cache reads, but the record remains authoritative.

```ts
import { createServer } from "node:http";
import { randomUUID } from "node:crypto";

type Mode = "enabled" | "shadow" | "disabled";
type Control = {
  mode: Mode;
  version: number;
  reason: string;
  changedAt: string;
  changeId: string;
};

let control: Control = {
  mode: "enabled",
  version: 1,
  reason: "initial state",
  changedAt: new Date().toISOString(),
  changeId: randomUUID(),
};

const server = createServer((request, response) => {
  if (request.method === "GET" && request.url === "/controls/delivery-alerts") {
    response.writeHead(200, { "content-type": "application/json" });
    response.end(JSON.stringify(control));
    return;
  }

  if (request.method === "PUT" && request.url === "/controls/delivery-alerts") {
    let raw = "";
    request.setEncoding("utf8");
    request.on("data", (chunk: string) => { raw += chunk; });
    request.on("end", () => {
      const input = JSON.parse(raw) as {
        mode: Mode;
        expectedVersion: number;
        reason: string;
      };
      if (input.expectedVersion !== control.version) {
        response.writeHead(409, { "content-type": "application/json" });
        response.end(JSON.stringify({ current: control }));
        return;
      }
      control = {
        mode: input.mode,
        version: control.version + 1,
        reason: input.reason,
        changedAt: new Date().toISOString(),
        changeId: randomUUID(),
      };
      response.writeHead(200, { "content-type": "application/json" });
      response.end(JSON.stringify(control));
    });
    return;
  }
  response.writeHead(404).end();
});

server.listen(3000);
```

The `expectedVersion` field is cheap insurance. Two responders can act at nearly the same time; without a compare-and-set condition, an older browser tab can overwrite a newer recovery decision. A conflict forces the operator to read the current state and decide again.

Do not treat the response from `PUT` as proof that mitigation worked. It proves only that the control record changed. Each worker should report the version it has observed, and dispatch telemetry should include that version. The incident is contained when active workers have converged and attempted sends have fallen as expected.

Consider a concrete sequence with three workers. Version 12 is `enabled`; an alert fires; an operator writes version 13 as `disabled`. Worker A refreshes immediately, worker B still has a valid local cache, and worker C cannot reach the control store. The write succeeded, yet only A has definitely stopped. B must stop no later than its cache deadline. C must apply the chosen failure policy when its cached record expires. The dashboard therefore needs to show A on 13, B on 12 with the remaining cache age, and C as refresh-unavailable. This is why the second phase is observation rather than another write. It also gives the responder a concrete completion rule: no active worker may keep authorizing dispatch with version 12 after the allowed cache window. The exact version numbers are illustrative, but the ordering rule is not optional.

## Make stale decisions visible

A local cache protects the control store from a read on every notification. It also creates the main failure mode: the operator disables dispatch while one worker continues using an old `enabled` value. The cache therefore needs a defined maximum age and observable freshness. Pick the age from the incident objective and expected store load; do not copy a fashionable number.

Here is the worker-side decision. It fails closed when it cannot refresh an expired record. That choice fits delivery-failure notices because pausing is safer than sending duplicates. A security alert or emergency message may require a different rule. The policy belongs in design review, not in an exception handler written during an outage.

```ts
type CachedControl = Control & { fetchedAtMs: number };
let cached: CachedControl | undefined;
const maxAgeMs = 5_000;

async function readControl(): Promise<Control> {
  const response = await fetch(
    "http://control.internal/controls/delivery-alerts",
    { signal: AbortSignal.timeout(500) },
  );
  if (!response.ok) throw new Error(`control read failed: ${response.status}`);
  return response.json() as Promise<Control>;
}

async function mayDispatch(nowMs = Date.now()): Promise<Control> {
  if (cached && nowMs - cached.fetchedAtMs <= maxAgeMs) return cached;
  const fresh = await readControl();
  cached = { ...fresh, fetchedAtMs: nowMs };
  return fresh;
}

type DeliveryFailure = { eventId: string; shipmentId: string };

async function handleFailure(event: DeliveryFailure): Promise<void> {
  let current: Control;
  try {
    current = await mayDispatch();
  } catch {
    recordDecision(event.eventId, "blocked_control_unavailable");
    return;
  }
  recordControlVersion(current.version);
  if (current.mode !== "enabled") {
    recordDecision(event.eventId, `blocked_${current.mode}`);
    return;
  }
  await sendDeliveryFailureNotice(event);
  recordDecision(event.eventId, "sent");
}

declare function recordControlVersion(version: number): void;
declare function recordDecision(eventId: string, result: string): void;
declare function sendDeliveryFailureNotice(event: DeliveryFailure): Promise<void>;
```

The `5_000` and `500` values are explicit engineering inputs, not universal defaults. Test them under control-store latency and worker concurrency that resemble the service. A shorter cache reduces exposure to stale state but increases control-plane traffic. A longer cache does the opposite. I would accept a little more read traffic here because the switch exists for a rare moment when time matters.

Five seconds is a choice, not a promise.

Keep notification content out of control telemetry. Event identifiers, decision results, control versions, and timestamps are usually enough to investigate convergence. Personal data copied into logs widens the data that must be governed and erased. Article 17 of the GDPR describes a right to erasure in applicable circumstances; minimizing copied payloads makes those obligations easier to execute.

## Alert on the outcome, not the toggle request

The primary alert should detect the delivery-notification failure mode: a rise in failed or duplicate dispatch outcomes relative to attempted dispatches. The control change creates a separate event with actor, reason, previous mode, new mode, version, and change identifier. Correlate the two streams. Do not declare success because the control endpoint was healthy.

Four signals cover the incident path without turning the system into a telemetry project:

1. Dispatch attempts and outcomes, split by control version.
2. The age of the last successful control refresh on each active worker.
3. Count of workers still reporting an older version after a change.
4. Queue age and depth while dispatch is paused.

The fourth signal protects recovery. A kill switch can stop immediate harm while a backlog quietly grows. When dispatch returns, releasing the entire queue at once may reproduce the incident or overload the downstream channel. Resume in `shadow`, inspect decisions, then enable a bounded drain rate. Preserve idempotency by using the event identifier as the dispatch key so replay does not become a second send.

Test the ugly paths. Run one test where a worker has a warm cache during disable, another where the control store is unavailable after cache expiry, and a concurrency test where two responders submit the same expected version. Then exercise disable, shadow, and enable against a synthetic queue. These tests reveal more about rollback safety than a dashboard screenshot.

## Choosing the least complex API

The selection criterion is not the longest feature list. Choose the smallest control mechanism that can provide atomic versioned updates, bounded read staleness, an audit event, and worker-level convergence evidence. A database row can meet that bar for a small service. A dedicated flag system can fit when many services, environments, or approval policies share the same control plane. The runtime contract should remain yours either way.

This design has limits. An in-process cache is not a fit when a side effect must stop with no stale window, and a single database row is a poor fit when hundreds of independently deployed services need delegated approvals or distinct regional controls. The trade-off is plain: fewer moving parts make the incident path easier to understand, while a richer control plane can provide distribution and governance at the cost of more dependencies. For a hard real-time safety boundary, use an enforcement point that fails closed at the shared side-effect gateway rather than asking every worker to converge eventually.

Avoid a client-side-only toggle for this job. Browser state does not govern queue workers. Avoid environment variables as the incident control because changing them commonly couples mitigation to process replacement, while the point of the switch is to separate a reversible runtime decision from deployment. Also avoid per-shipment targeting until there is a real operational need; each targeting dimension adds another path responders must reason about under pressure.

Cost deserves one line in the design, not the headline. Estimate control reads as active workers divided by cache duration, then decide whether the store and telemetry budget can carry that steady load. The expensive failure is an ambiguous switch that looks disabled while side effects continue.

During an incident, the operator records a reason and changes `enabled` to `disabled` with the current version. The responder watches worker convergence and dispatch outcomes, while another engineer fixes the notification path. Recovery moves first to `shadow`, checks queued decisions and downstream readiness, then returns to `enabled` with a bounded drain. Afterward, retain the control audit record according to policy and delete unnecessary personal payloads from diagnostic data. That is the whole playbook.

## Further reading

- GDPR Article 17, Right to erasure: https://gdpr-info.eu/art-17-gdpr/
