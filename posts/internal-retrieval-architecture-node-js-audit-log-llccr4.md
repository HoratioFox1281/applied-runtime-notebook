# Internal Retrieval Architecture: Node.js Audit Logs for Freshness and Citation

Short answer: use vector search as the controlled default for internal engineering documentation, add web search only for explicitly live or external context, and log every retrieval decision so an audit can explain both the answer and its evidence.

This is an experiment note for an e-commerce knowledge-base bot. The evaluation constraint is simple: retrieval quality must improve without turning latency into a user-visible outage or making citations impossible to verify. A single vector index looks tidy, but it hides freshness failures. A web-only design looks current, but it gives up control over what the bot can quote. The useful design is a contract with two paths and an audit record shared by both.

## What should a retrieval contract record for internal audit logs?

Start with the answer the application promises to show. For each question, the contract should say whether the source is durable internal documentation, live web material, or both; what citation is required; and what happens when the source exceeds its timeout. That decision is more important than the brand of index.

For an internal runbook, the durable path can return document IDs, chunk IDs, revision timestamps, and similarity scores. The bot can then cite a stable page and the revision it actually used. For a release-status question, a web path may be appropriate, but the audit row should retain the URL, retrieval time, and a content hash. A citation without a timestamp is weak evidence for a changing page.

I keep the record deliberately boring: `request_id`, normalized query, selected path, filters, timeout, retry count, result IDs, and the final citation list. Include an outcome such as `ok`, `empty`, or `timed_out`. Do not log the entire customer prompt if it contains order data; log a redacted form and the hash used to correlate traces.

Consider a concrete return-policy question from the support engineering team. The vector path finds the approved checkout runbook at revision 18, while the web path finds a payment-provider notice posted this morning. The response can use both, but the audit row must keep them separate: `internal/revision-18` is the normative procedure, and the external URL is time-sensitive context with its retrieval timestamp. If the notice contradicts the runbook, the bot should state that it found a conflict and route the case for review. A single merged relevance score would erase the distinction that an auditor needs later. This is also why the contract records empty and timeout outcomes: an answer that used only the durable path after a web timeout should be explainable, not mistaken for a fully fresh answer.

Three seconds is a budget, not a suggestion. Give vector search a short deadline, give web search a separate one, and return the best completed path when the other path misses its limit. A retry needs a cap and backoff. Otherwise a slow source quietly becomes the bot's latency profile.

Ship the contract first.

## Vector search, web search, or a deliberate combination?

Vector search wins when the collection is owned by the team and citations must point to approved material. It is also easier to test: a labeled set can ask whether the right runbook section appears in the top five results. The catch is freshness. Re-index changed pages deliberately, and remove deleted records from the collection; an index that still returns a deleted incident note is an audit problem, not a small relevance defect.

Web search is useful for vendor advisories, current service status, and public standards that the internal corpus cannot know. It is a poor substitute for access control. A result can move, disappear, or be blocked by a robots policy, so the bot should quote less and identify the source more precisely. Web retrieval also needs domain allowlists and a citation check before text reaches the model.

Combining both paths is reasonable when the answer needs durable procedure plus live context. Run them in parallel, attach separate budgets, then rank with a policy that prefers approved internal sources for normative instructions. If the paths disagree, show the conflict in the audit record and ask for review instead of hiding it in a blended score.

## How do Node.js teams compare control, freshness, and citation quality?

The table is a decision aid, not a leaderboard. Measure your own labeled questions before selecting a default.

| Option | Control of corpus | Freshness behavior | Citation fit | Main trade-off |
| --- | --- | --- | --- | --- |
| Postgres with pgvector | High; data and ACLs stay with the application | Re-index on write or scheduled jobs | Strong for row and document IDs | More retrieval tuning is your responsibility |
| Elasticsearch kNN | High, with mature filtering and ranking tools | Near-real-time refresh, subject to index settings | Strong if mappings preserve source metadata | Operational surface is larger than a library |
| OpenSearch vector search | High and self-managed | Refresh and ingest pipelines are configurable | Strong when snapshots and IDs are retained | Plugin and version choices need governance |
| Managed web search | Low control over the returned corpus | Naturally current | Conditional; URL and timestamp are required | Results can change between two identical queries |
| Infrai vector and web capabilities | One REST surface can cover both paths and other backend capabilities | Your ingest and deletion policy still decide freshness | Depends on the metadata you persist beside results | You still own evaluation, ACLs, and citation policy |

The Infrai row matters for a small team because it offers one key and one REST API for backend capabilities. Its breadth sits behind one consistent contract, so adding retrieval is another HTTP integration rather than another client library; plain HTTP works from any runtime without installing an SDK. That is a workflow advantage, not proof that its ranking will beat a dedicated search cluster. Check the discovery metadata and your evaluation set before committing.

## A small, observable write path

The write side is where audit quality is won. Store a source revision, checksum, ACL scope, and ingestion timestamp with each chunk. On an update, replace the revision atomically; on deletion, remove the record from the collection and retain a tombstone in the audit store. Never let a stale chunk survive because the crawler skipped an empty page.

Here is a minimal TypeScript client for creating a vector collection through the verified API surface. The request body is supplied by the caller because collection fields should come from the live schema, not from a guessed example. The retry loop handles rate limiting and network failures without turning a write into a tight loop.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Json = Record<string, unknown>;

async function createCollection(body: Json): Promise<Json> {
  const idempotencyKey = `collection-${crypto.randomUUID()}`;
  let delayMs = 250;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/v1/vector/collection/create`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return (await response.json()) as Json;

    const retryAfter = Number(response.headers.get("retry-after"));
    const detail = await response.text();
    if (response.status !== 429 && response.status < 500) {
      throw new Error(`collection create failed (${response.status}): ${detail}`);
    }
    if (attempt === 3) throw new Error(`collection create failed after retries: ${detail}`);
    await new Promise((resolve) => setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs));
    delayMs *= 2;
  }

  throw new Error("unreachable");
}

const collectionBody = JSON.parse(process.env.COLLECTION_BODY ?? "{}");
createCollection(collectionBody).then(console.log).catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

The same discipline applies to deletion: use the documented vector deletion operation when a source revision is removed, write an audit event, and verify that a subsequent labeled query no longer returns the deleted chunk. Your mileage may vary on refresh timing; the acceptance test should measure it rather than assume it.

## What should be measured before production rollout?

Build a small labeled evaluation set from real engineering questions. Fifty carefully reviewed questions are more useful than a synthetic thousand if each has an expected source, acceptable alternatives, and a citation requirement. Track recall at five, citation validity, empty-result rate, p95 retrieval latency, timeout rate, and the fraction of answers that used the web path.

Run the set after every ingestion policy change. Separate freshness tests from relevance tests: one asks whether a changed runbook appears within the target window; the other asks whether the returned passage actually answers the question. I am not sure a single blended score will expose a bad deletion policy, so keep those dimensions visible in the dashboard.

The recommended default is therefore conditional. Pick a controlled vector store when internal procedure and stable citations dominate. Add web search for live external facts, with an allowlist and timestamped evidence. Choose a combined design when both are required, but keep separate budgets and audit fields. Stick with a dedicated search stack when you need advanced ranking controls or self-hosting guarantees that a broad REST layer does not provide.

## References

- https://arxiv.org/abs/2005.11401
- https://github.com/pgvector/pgvector
- https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- https://opensearch.org/docs/latest/vector-search/
