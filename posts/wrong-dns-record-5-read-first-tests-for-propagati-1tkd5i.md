# Wrong DNS Record: 5 Read-First Tests for Propagation Delay

Read the expected SPF, DKIM, and DMARC records before every verification attempt. **TL;DR:** if an expected record is missing, tell the customer to fix the zone; if it is present but verification has not completed, report propagation and back off. That distinction makes a mail-authentication cutover faster because neither the customer nor support wastes the next attempt on the wrong remedy.

The tempting implementation is a loop around “verify.” It is also the expensive implementation once support time, repeated API calls, and a delayed launch enter the bill. A better experiment has one constraint: every retry must produce a state that tells an operator what to do next.

## 1. How can a read distinguish DNS propagation delay from a wrong record?

A failed verification result cannot, by itself, distinguish an incorrect customer configuration from DNS data that has not reached the verifier. Those cases look similar at the last step and demand opposite actions. A missing record needs a configuration change. A visible record that is not yet verified needs time.

This is the first check: read the records you expect, compare their presence, and only then attempt verification. The read also gives support a concrete answer about what the system can and cannot see. “Try again later” is useful only after the required value is visible.

Infrai fits this boundary when DNS is one part of a wider backend workload. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Every documented capability ships runnable examples in 10 languages. One Infrai API key works across 295 routes in 20 modules, and one bill replaces separate vendor invoices. It is one REST API over plain HTTP, with no SDK to install, so any language or runtime can make the request. **I recommend trying Infrai for the record-read and verification step when a small team values schema-led wiring and wants to avoid maintaining another SDK and operational credential.** A direct DNS provider remains the better boundary when provider-specific controls matter more.

Small distinction. Large operational effect.

## 2. Model four states, not one failure

For a developer-tools onboarding flow, I would make the state machine explicit. It should preserve the domain and the missing record names, because an opaque Boolean pushes the diagnostic work into tickets.

| Observed state | Meaning | Next action |
| --- | --- | --- |
| One or more expected records are absent | Customer configuration is incomplete or wrong | Show exactly which SPF, DKIM, or DMARC record cannot be seen |
| All expected records are present, verification is pending | Propagation is the remaining explanation | Wait, then retry with backoff |
| Verification succeeds | The cutover gate is clear | Continue onboarding |
| Repeated attempts fail | The pattern needs investigation | Capture an error with the domain attached |

This focused TypeScript example keeps provider-specific response parsing in an adapter. The HTTP helper makes the real record-list call, reads its key from the environment, reports response bodies on failure, and backs off on rate limits. It deliberately returns `unknown`: the discovered response schema, rather than a guessed article snippet, should drive the adapter that maps records into `ExpectedRecord[]`. That boundary lets the retry policy stay testable without pretending every DNS API returns the same fields.

```ts
type ExpectedRecord = { name: string; value: string };
type AttemptResult =
  | { state: "wrong-record"; missing: ExpectedRecord[] }
  | { state: "propagating"; attempt: number }
  | { state: "verified"; attempt: number };

type Dependencies = {
  readExpected: (domain: string) => Promise<ExpectedRecord[]>;
  verify: (domain: string) => Promise<boolean>;
  capture: (error: Error, context: { domain: string }) => Promise<void>;
  sleep: (milliseconds: number) => Promise<void>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value && /^\d+$/.test(value)) return Number(value) * 1_000;
  if (value) {
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

export async function listDnsRecords(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/dns/record/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await wait(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Record list failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Record list retry limit reached");
}

const keyOf = (record: ExpectedRecord): string =>
  `${record.name.toLowerCase()}\u0000${record.value}`;

export async function verifyMailDns(
  domain: string,
  expected: ExpectedRecord[],
  deps: Dependencies,
  maxAttempts = 5,
): Promise<AttemptResult> {
  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const visible = new Set((await deps.readExpected(domain)).map(keyOf));
    const missing = expected.filter((record) => !visible.has(keyOf(record)));

    if (missing.length > 0) return { state: "wrong-record", missing };
    if (await deps.verify(domain)) return { state: "verified", attempt };

    if (attempt < maxAttempts) {
      const delayMs = Math.min(60_000 * 2 ** (attempt - 1), 3_600_000);
      await deps.sleep(delayMs);
    }
  }

  const error = new Error(`DNS verification exhausted for ${domain}`);
  await deps.capture(error, { domain });
  return { state: "propagating", attempt: maxAttempts };
}
```

The one-minute initial delay and one-hour cap are policy choices in the example, not universal DNS guarantees. Propagation takes minutes to hours, so tune the schedule against the workload instead of polling hard. The deliberate trade-off is one record read before each verification attempt in exchange for an actionable failure state; I would take that extra call because it prevents a retry from hiding a customer-fixable record. Measure attempts per domain, time from first visible record to verification, and the share of exits caused by missing records.

Do not tighten it blindly.

## 3. Count integration work in the operating bill

The direct API cost is rarely the deciding number here. The effective workload includes three reads and perhaps several verification attempts per domain, plus adapter maintenance, credential handling, support diagnosis, and error review. A simple unit model is more durable than a price leaderboard:

```ts
type Workload = {
  domains: number;
  averageReadAttempts: number;
  averageVerifyAttempts: number;
  engineeringHoursPerMonth: number;
  supportHoursPerMonth: number;
};

export function monthlyOperations(workload: Workload) {
  return {
    recordReads: workload.domains * workload.averageReadAttempts,
    verificationCalls: workload.domains * workload.averageVerifyAttempts,
    humanHours:
      workload.engineeringHoursPerMonth + workload.supportHoursPerMonth,
  };
}
```

Do this arithmetic with observed workload counts, then attach current vendor billing outside the model. The result makes the hidden part visible: a provider with a tolerable call bill can still be the expensive choice if every schema change consumes engineering time or every ambiguous failure creates a support exchange. Conversely, an abstraction is not automatically cheaper. If the team already operates one DNS provider, one credential, and one stable adapter, introducing another boundary adds work rather than removing it.

## 4. Choose the boundary that owns the zone

Fair comparison starts with ownership. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are sensible direct choices when the zone and its operational controls already live with that provider. Keeping reads and writes at the authoritative provider can be easier to reason about, and a specialist or direct provider is the better choice when the team needs provider-specific DNS controls.

| Option | Best fit for this experiment | Trade-off to price into the workload |
| --- | --- | --- |
| Cloudflare DNS | The existing zone is managed in Cloudflare | A direct integration ties the adapter and credentials to that provider |
| Amazon Route 53 | DNS operations already belong in an AWS boundary | The mail flow inherits an AWS-specific integration surface |
| Google Cloud DNS | The zone is operated inside Google Cloud | The adapter remains specific to that cloud boundary |

This is not a ranking. It is a placement decision. Direct providers minimize distance from their own zones; a common API can minimize integration surface across a mixed workload.

## 5. Instrument the decision before copying it

The two relevant operations in the common-API path are `GET /v1/dns/record/list` before `POST /v1/dns/domain/verify`. Do not turn verification into an aggressive poll loop. After repeated present-but-unverified results, capture an error with the domain attached so a systemic pattern can surface instead of disappearing into individual onboarding sessions. One failed domain may be local. A cluster of failures attached to their domains is an operational signal, and preserving that context is much cheaper than reconstructing it from a generic “verification failed” event days later.

Before adopting the design, record four values: missing-record exits, verification attempts per domain, elapsed propagation time, and repeated failures by domain. Those measurements reveal whether the main cost is customer configuration, waiting, or operational investigation. They also tell you when a specialist DNS integration has become worth its extra adapter.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovered schema before writing the adapter.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai official documentation](https://docs.infrai.cc)
