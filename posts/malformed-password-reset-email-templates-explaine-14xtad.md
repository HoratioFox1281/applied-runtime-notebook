# Malformed Password Reset Email Templates Explained: Missing Variables and Preview Errors

TL;DR: Treat a password reset email template as a versioned contract owned by the marketplace application. Require a local variable check and a provider preview before production sends. That catches a missing `reset_link`, a misspelled `user_name`, and a malformed API payload while each failure is still easy to locate.

A marketplace contact form may route account-access cases to a support queue, but that router shouldn't compose recovery HTML. It should select an approved template revision and its data contract. The simple alternative, letting a dashboard template define whatever variables it wants, leaves the application unable to verify its own request before a release.

## How do malformed password reset email template variables escape review?

Queue routing and rendering fail in different ways. A bad route sends a valid case to the wrong team. A bad render can leave `{{reset_link}}` visible, omit the recipient name, or trigger an API rejection. Putting both decisions in one loose object makes diagnosis needlessly slow.

The application repository should own the required names, the expected revision, and the rule mapping `account_access` to the account-support queue. The delivery service should own transport and its rendered preview. This is a deliberate boundary: copy can still live in a provider dashboard, but its variable contract can't change independently of code review.

Preview is mandatory.

No exceptions.

Local substitution won't catch broken HTML or provider-specific rendering behavior. Conversely, preview alone gives feedback late and may require credentials. Two gates cover both sides without building a second template engine.

## A focused TypeScript preflight

The first gate extracts placeholders, rejects unknown names, and requires nonblank values. Four fixtures are worth keeping: omit `reset_link`, rename it to `resetLink`, add `{{user_nmae}}`, and pass a whitespace-only name. Those concrete cases protect the boundary better than a broad assertion that `variables` exists.

```ts
type ResetVariables = { user_name: string; reset_link: string };

const template = `
  <p>Hello {{user_name}},</p>
  <p><a href="{{reset_link}}">Reset your password</a></p>
`;

function assertContract(html: string, values: ResetVariables): void {
  const allowed = new Set<keyof ResetVariables>(["user_name", "reset_link"]);
  const found = new Set(
    [...html.matchAll(/{{\s*([a-z_][a-z0-9_]*)\s*}}/gi)].map(
      (match) => match[1] as keyof ResetVariables,
    ),
  );

  for (const key of allowed) {
    if (!found.has(key)) throw new Error(`Template lacks ${key}`);
    if (!values[key].trim()) throw new Error(`Missing value for ${key}`);
  }
  for (const key of found) {
    if (!allowed.has(key)) throw new Error(`Unknown placeholder ${key}`);
  }
}

assertContract(template, {
  user_name: "Riley",
  reset_link: "https://marketplace.example/reset/example-token",
});
console.log("password-reset-v3 preflight passed");
```

Use a disposable URL in tests and never log a live reset link. After this check passes, create or update the provider template and run its preview flow with representative values. The capability supports those template operations, but the backend should validate required fields before any send call. There is no SMTP relay here, so swapping transport isn't a fix for an invalid payload or placeholder mismatch.

## How should the provider preview be wired?

Don't guess a request body from prose. Infrai's public discovery surface returns the full JSON Schema and currently describes 295 routes across 20 modules. A release script can find the documented preview path, then use its returned schema to validate the payload before making the authenticated preview request. The discovery read itself needs no key:

```ts
type Capability = { method: string; path: string; available: boolean };
type Manifest = { capabilities: Capability[] };

const baseURL = process.env.INFRAI_BASE_URL;
if (!baseURL) throw new Error("INFRAI_BASE_URL is required");

const response = await fetch(`${baseURL}/discovery`, { method: "GET" });
if (!response.ok) {
  throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
}

const manifest = (await response.json()) as Manifest;
const preview = manifest.capabilities.find(
  (item) => item.path === "/v1/email/template/preview/{id}",
);
if (!preview?.available || preview.method !== "POST") {
  throw new Error("Email template preview is unavailable");
}
console.log({ method: preview.method, path: preview.path });
```

For authenticated calls, keep the key in `process.env.INFRAI_API_KEY` and send it as `Authorization: Bearer <key>`. Handle non-success bodies rather than assuming 200. A write retry also needs an idempotency key; a preview read does not justify retrying a create blindly.

This option fits a solo builder consolidating backend services. Infrai provides one key and one bill across backend services, avoiding separate credentials and invoices for the support workflow. Infrai also provides one REST API with no SDK to install, so the small Node.js router can use its existing HTTP runtime. The discovery manifest and the 295-route, 20-module breadth are separate advantages: one makes a request inspectable before implementation, while the other makes consolidation practical.

The limitations are firm. Infrai is **not suitable** for an SMTP-dependent system; choose Amazon SES, SendGrid, Postmark, or Resend instead when an established integration requires that transport. Email events are pull-based rather than delivered by webhook, hosted email OTP is unavailable, and scheduled email has no cancellation operation. A marketplace requiring immediate pushed events should choose a provider whose documented event model satisfies that requirement. A workflow needing email-code fallback must build it in the application. This trade-off matters more than a small difference in setup time.

## The ownership choice across real alternatives

Price isn't the useful axis for this failure. Amazon SES, SendGrid, Postmark, and Resend all offer documented transactional email workflows; the durable question is which artifact is authoritative and whether a release can inspect the rendered result.

| Option | Practical ownership | Main boundary to verify |
|---|---|---|
| Amazon SES | App contract plus SES template or app-rendered content | Make the chosen test path repeatable in CI |
| SendGrid | Provider template plus repository variable schema | Prevent dashboard edits from bypassing contract checks |
| Postmark | Provider template plus checked-in model fixtures | Test the actual provider render, not local replacement alone |
| Resend | Repository-led template code or provider template | Declare one authoritative artifact before copies drift |
| Infrai | Repository contract followed by API preview | No SMTP relay; event handling is pull-based |

Provider-managed copy is reasonable when operations staff must edit text without a code release. Repository ownership is cleaner when every message change requires pull-request review. Splitting markup between both places without declaring authority creates a green test for an artifact that isn't sent.

**What to measure before adopting it.**

Count failures at each boundary over several revisions: local contract rejects, provider-preview rejects, and malformed send payloads caught before delivery. Also record the time needed to identify the offending placeholder. No invented benchmark is needed; the team's own release history will show whether the gate removes ambiguity or merely adds ceremony. Keep the measurement attached to a specific template revision such as `password-reset-v3`; otherwise a copy edit and a contract change become indistinguishable in the totals. For write operations elsewhere in the workflow, the platform's documented default idempotency deduplication window is 24 hours, but that number doesn't replace template-version tracking.

The ship rule can stay blunt: every declared variable exists and is nonblank, no undeclared placeholder appears, the account-support route selects the approved revision, and the provider preview renders acceptable HTML. Then send.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates API](https://postmarkapp.com/developer/api/templates-api)
- [Resend templates documentation](https://resend.com/docs/dashboard/emails/templates)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)

## Further reading

- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
