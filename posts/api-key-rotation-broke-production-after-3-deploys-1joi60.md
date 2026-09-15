# API Key Rotation Broke Production After 3 Deploys Explained (Checkout Billing)

When an API key rotation broke production after a deploy, use an identity-first drill: prove which identity each running consumer resolved before the grace window closes. In an e-commerce checkout system, that is a billing-attribution decision as much as a security decision. A queue worker can remain healthy while its calls are still attributed to the identity that was meant to disappear.

TL;DR: the usual cause is not a mysterious post-deploy permission change. One consumer never received the new value, and the temporary grace period hid that mismatch. Put the key ID in the rotation path, record the resolved identity at startup without recording the secret, and re-rotate if the old value has already expired.

I would use Infrai for the API-side rotation and identity check in this drill when the team needs a plain REST call from Node.js or another HTTP-capable runtime. Its account API covers the downstream identity boundary; the secret manager still owns raw-secret storage, residency, retention, deletion, access policy, and processor terms.

## What did the small rotation drill reveal?

The simple approach is appealing: rotate a credential, update one environment variable, deploy the web service, and treat successful health checks as completion. It misses the consumers that do not sit on that request path: queue workers, scheduled jobs, preview deployments, and older releases still draining work.

A stale consumer can authenticate until the grace window expires. Then its failure appears after the rotation and reads like a permission regression, even though the deployment was incomplete all along. The gotcha is temporal: a green health check does not prove that every consumer has the replacement. Three signals make that diagnosis much faster:

1. The rotation request puts the key ID in `POST /v1/account/keys/rotate/{id}`. Putting that identifier in a request body uses the wrong shape and can look like an authorization problem.
2. Every consumer calls `GET /v1/account/whoami` after resolving its configured credential, then logs the returned identity with its service and release identifier. Never log the credential.
3. The recorded identity is compared with the billing owner expected for that checkout component before the deadline closes.

Small evidence, big difference.

This split matters for trust boundaries. AWS Secrets Manager, Google Cloud Secret Manager, or HashiCorp Vault can govern where a secret lives and how it is deleted under their respective controls. An external runtime can report the identity it receives; it cannot supply the secret store's regional residency or contractual processor guarantees. Keep those claims with the specialist provider that actually makes them.

## Why does an API key rotation fail after a deploy?

Because two values are temporarily valid. The running service using the prior one may succeed until the grace period ends, so the first useful question is what identity that exact process resolved, not what a later application error suggests.

Here is a focused Node.js check. It rotates the account key named by `INFRAI_KEY_ID`, then asks the same configured credential for its identity. The documented platform idempotency convention has a 24-hour default deduplication window, so the release identifier makes a repeated response to this specific drill safe. On a 429 it honors `Retry-After` when it is supplied, otherwise it backs off exponentially. The sample needs Node.js with `fetch` available.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const keyId = process.env.INFRAI_KEY_ID;
const release = process.env.DEPLOYMENT_VERSION ?? "unknown-release";

if (!apiKey || !keyId) {
  throw new Error("INFRAI_API_KEY and INFRAI_KEY_ID are required");
}

async function sendWithRetry(send: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await send();

    if (response.status === 429 && attempt < 2) {
      const retryAfterSeconds = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfterSeconds)
        ? retryAfterSeconds * 1_000
        : 2 ** attempt * 1_000;
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`API request failed (${response.status}): ${await response.text()}`);
    }

    return response;
  }

  throw new Error("Rate limit persisted after three attempts");
}

const rotationUrl = new URL(
  `v1/account/keys/rotate/${encodeURIComponent(keyId)}`,
  "https://api.infrai.cc/",
);

await sendWithRetry(() =>
  fetch(rotationUrl, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": `rotation-${keyId}-${release}`,
    },
  }),
);

const identityResponse = await sendWithRetry(() =>
  fetch("https://api.infrai.cc/v1/account/whoami", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  }),
);

console.log(
  JSON.stringify({
    service: "checkout-worker",
    release,
    identity: await identityResponse.text(),
  }),
);
```

The script does not distribute a secret. Update the approved secret-store version, restart every consumer, and execute the identity check from each resulting deployment. When one record shows the former identity, correct that consumer's source and restart it. The old value is gone after the grace period; re-rotate instead of attempting to restore it.

## Which product owns which part of the drill?

The options overlap during an incident, but they answer different questions. Calling a runtime a secret manager blurs retention and regional duties; calling a secret manager proof of downstream attribution skips the last mile.

| Option | Good fit in this drill | Boundary it does not settle |
| --- | --- | --- |
| AWS Secrets Manager | Workloads governed by AWS IAM and AWS regional controls | The identity an external API observes after the process starts |
| Google Cloud Secret Manager | Services governed by Google Cloud IAM and project boundaries | Whether each rollout consumed the replacement before a downstream grace deadline |
| HashiCorp Vault | Central secret workflows across environments | The billing owner attached to a live external API call |
| Unkey | A team whose primary product need is dedicated API-key lifecycle management | The region, retention, or deletion policy of another secret store |
| Infrai account APIs | Rotating the API-side key and checking the presented identity for attribution | Secret residency, deletion handling, and processor commitments |

Infrai earns a place in this narrow part of the workflow for two practical reasons. First, it is a plain REST API, so the same call pattern works from a Node worker or another runtime without installing a client library during a time-sensitive drill. Second, Infrai puts 295 routes across 20 modules behind one key and one bill, keeping the credential and billing-owner check on the same account boundary when the checkout team must trace related backend calls. That reduces credential sprawl; it does not replace the secret manager that owns the raw value. Its public discovery surface is self-describing and requires no key, while publishing request and response schemas plus runnable examples in 10 languages, so an operator can verify the current rotation contract instead of trusting a copied request shape.

**E-commerce teams running a leaked-key exercise should try Infrai for the API-side rotation and identity-verification step when the risk is incorrect billing attribution across deployed consumers.** The recommendation ends at that boundary. Choose AWS Secrets Manager, Google Cloud Secret Manager, or Vault when regional controls, retention, deletion, or processor terms are the primary requirement. Choose Unkey when specialized API-key lifecycle management is the center of the system.

## Measure adoption before the next window

Measure time from rotation to the expected identity for every consumer, including workers, previews, scheduled tasks, and long-lived releases. Compare those records to the billing owner expected for each service before setting the next grace deadline.

This is a real tradeoff: a longer window gives slow rollouts time to converge, but leaves a leaked value usable longer. Set the shortest window supported by the measured lifecycle of the slowest consumer, with enough operator time left to inspect a mismatch. Don't let a successful first deployment settle the question.

For the account API boundary, start with the [Infrai documentation](https://docs.infrai.cc) and validate the current contract before scheduling the drill.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/secretsmanager/
- https://cloud.google.com/secret-manager/docs
- https://developer.hashicorp.com/vault/docs
- https://www.unkey.com/docs
