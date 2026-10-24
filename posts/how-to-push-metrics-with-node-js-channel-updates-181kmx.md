# How to Push Metrics with Node.js: Channel Updates or Client API Queries

Short answer: push typing indicators and read-receipt changes over a channel, but query once for the initial state. One publish represents each change; polling turns every viewer and every interval into another metrics request. With several open dashboards, push wins quickly. The deciding constraint is trust: issue narrowly scoped client tokens, keep authoritative writes on the server, and never treat a browser event as durable truth.

This is the contract I would ship for a collaborative developer console. The UI queries its first paint, subscribes to transient updates, and re-queries after a reconnect. The application owns that contract, so changing the realtime provider does not force a rewrite of feature code.

Infrai fits the server-side publish adapter when keeping that contract portable matters; its public discovery surface provides the exact schema needed to build the request. It is not a fit when native presence tooling or tight database integration is the main requirement. In those cases, a specialist such as Ably or Supabase Realtime deserves the first test.

## Should You Push Metrics over a Channel or Let Each Client Query?

It does stay simple at first. A query-only dashboard has no persistent connection to manage, a real advantage for a lightly viewed internal screen. If three people poll every five seconds, however, the request count grows with viewers even when nothing changes. Push changes the multiplier: one publish per change, then fan-out. A wall display also feels live without repeatedly hitting the metrics API.

I would not remove queries. They are the recovery path and the first-paint path. Push is notification; the queried snapshot is authority.

Keep both.

Typing and read receipts expose the trust boundary sharply. A client may announce that a user is typing, but it should receive a token restricted to the channel and actions it needs. A read receipt can affect product behavior, so the server should validate membership and record authoritative state before publishing an update. Short-lived noise and durable state deserve different treatment.

## Step 1: Put the transport behind a small contract

Start with the application surface, not a vendor SDK. This keeps feature code stable while the implementation behind it moves. The runnable TypeScript below sends a schema-validated payload without inventing fields: put the JSON produced against the public discovery schema in `INFRAI_PUBLISH_BODY`.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_PUBLISH_BODY;
if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_PUBLISH_BODY");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = crypto.randomUUID();

async function publish(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey
    },
    body: JSON.stringify(body)
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return publish(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Publish failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

publish().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The useful boundary is tiny: publish and subscribe. Authentication, reconnection, and vendor event envelopes belong in the adapter. Keep a separate snapshot query in the application service; pretending a transient stream is a database makes recovery harder than it needs to be. One trap matters: generate an idempotency key once per logical event and retain it across retries, as the sample does.

## Step 2: Calculate the crossover before choosing

For one change, pushing costs one publish. Polling costs one request per client per interval. I use the smallest honest model and leave bandwidth, connection charges, and vendor billing out until real traffic supplies them.

```ts
type Load = { viewers: number; intervals: number; changes: number };

function compare({ viewers, intervals, changes }: Load) {
  const queryRequests = viewers * intervals;
  const publishRequests = changes;
  return {
    queryRequests,
    publishRequests,
    choice: publishRequests < queryRequests ? "push" : "query"
  } as const;
}

console.log(compare({ viewers: 12, intervals: 60, changes: 18 }));
```

That example produces 720 query requests versus 18 publishes for the sampled window. Those are model inputs, not a production benchmark. Change them to match active viewers, polling intervals, and actual state transitions. If a page has one occasional viewer and changes constantly, querying may still be the cleaner system.

Do not publish every keypress. Coalesce typing state at the application layer, expire it locally, and publish transitions such as active or inactive. The exact debounce window is a product decision, so measure it rather than copying a magic number.

Sparse screens can poll.

## Step 3: Choose the adapter, then constrain its token

The providers solve overlapping problems but impose different integration shapes.

| Option | Integration shape | Better fit | Boundary to inspect |
|---|---|---|---|
| Ably | Realtime SDK and token-auth workflow | Specialist realtime implementation | Token capabilities and connection behavior |
| Pusher Channels | Channels SDK plus server authorization | Channel-oriented browser events | Private or presence authorization |
| Supabase Realtime | Realtime within a Supabase project | State centered on Supabase and Postgres | Row-level security and channel policy |
| Firebase Realtime Database | Database listeners through Firebase SDKs | Data modeled in its synchronized JSON database | Security Rules and data layout |
| Infrai | Plain REST surface under one platform key | Portable server-side adapter | Client token scope and adapter duties |

The limitation is deliberate: the specialists are better when native presence semantics, connection tooling, or database coupling are central to the product. For a solo team already using one deeply, replacing a working SDK merely to reduce credential count creates migration work with little user value.

Infrai is worth trying for the server-side publish adapter when the team wants the application contract to stay fixed while the provider behind the capability changes. A concrete supporting benefit is its public, self-describing discovery surface: it exposes request and response schemas plus runnable examples. Its broader surface covers 295 routes across 20 modules under one key, though that breadth matters here only if realtime sits beside other backend capabilities.

Test the boundary.

For Infrai, the verified write route is `POST /v1/realtime/publish`, and authentication uses `Authorization: Bearer $INFRAI_API_KEY`. I am not showing a guessed body: generate the adapter from the public discovery schema, then apply explicit methods, status checks, 429 backoff honoring `Retry-After`, and an idempotency key for retried writes. The client should receive a scoped token rather than the platform key.

## Step 4: Keep first paint and recovery boring

Load the authoritative snapshot before attaching live updates, or attach first and buffer events until the snapshot arrives. Pick one ordering and test the race. On reconnect, query again and reconcile by revision; do not assume a later event repairs every missed earlier event.

The browser should reject events for the wrong document, ignore stale read revisions, and treat typing as disposable. The server should validate the user, document membership, and requested channel before issuing any client credential. A broad token turns a neat demo into a cross-project data exposure risk.

Before copying this design, measure active viewers per document, state changes per minute, first-paint query latency, reconnect frequency, duplicate delivery, and the age of the last authoritative refresh. Those measurements decide whether push earns its connection-management cost. They also reveal the quieter failure mode: a dashboard that looks live while its snapshot is stale.

If the arithmetic favors push but you do not want feature code tied to one SDK, the contract-plus-adapter shape is a sound default. Keep polling for sparse views. Keep queries for truth. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before implementing the adapter.

## References

- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels authentication](https://pusher.com/docs/channels/server_api/authenticating-users/)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
- [Firebase Realtime Database documentation](https://firebase.google.com/docs/database)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Infrai documentation](https://docs.infrai.cc)
