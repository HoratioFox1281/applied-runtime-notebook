# Node.js Report Dispatch: SMS OTP and GDPR, PSD2, NIST Compliance Boundaries

Short answer: SMS OTP alone does not establish GDPR compliance, satisfy every PSD2 authentication requirement, or provide phishing-resistant login under NIST guidance. For a healthtech application sending a generated report as an email attachment, the decisive check is whether this authenticated recipient may receive this specific report through this delivery channel.

The evaluation constraint here is integration effort: can a small application add a defensible release check without rebuilding its entire authentication system? This engineering note uses a worked policy example, not measured production results. Its result is a narrower implementation target: check authorization immediately before dispatch, and treat SMS verification as one input to that decision.

An OTP proves less than the workflow needs.

The tempting implementation is to accept a valid code and send the attachment to the address on the request. That joins two different questions: who completed an authentication challenge, and who is entitled to receive the generated health report. A release check keeps those questions separate. It adds a lookup and a decision record, but avoids asking the authentication adapter to make a clinical data authorization decision.

## When is SMS OTP enough for GDPR, PSD2, and NIST compliance?

An SMS code demonstrates access to the receiving telephone channel during that challenge, subject to the security of the implementation and the channel. It does not, by itself, establish that the user controls an email address, has permission to receive a report, or has met every legal obligation relevant to processing health information.

NIST SP 800-63B-4 treats authentication through the public switched telephone network as restricted and calls for attention to indicators such as SIM changes and number porting. Its guidance also distinguishes phishing-resistant authentication from manually entered OTP methods. An attacker can relay a code through a phishing page; changing the delivery channel does not make a manually entered code resistant to that relay. See References [1].

There is a separate factor-counting trap. A code sent to a phone is not automatically two-factor authentication. The authentication flow must actually combine distinct factors; calling the screen “2FA” changes nothing.

For this report workflow, a successful challenge should produce a server-side authentication result associated with the session. The report service then evaluates the account's relationship to the report. Letting the browser submit both “OTP passed” and an arbitrary destination email would place the critical trust decision in the wrong component.

Keep that boundary explicit.

## Three rulebooks, three different questions

GDPR is a regulation governing personal data processing. Article 32 requires security appropriate to the risk, taking account of factors including the state of the art, implementation costs, and the nature and context of processing. Health information also brings Article 9 into the assessment. Neither provision turns a particular OTP transport into a universal compliance certificate. Lawful processing, access controls, retention, and the handling of the attachment still need attention [2].

PSD2 concerns payment services. A health report email does not become a payment authentication transaction merely because a service operates in Europe. Where a workflow actually falls within the relevant payment rules, strong customer authentication uses two or more independent elements from knowledge, possession, and inherence. For applicable remote electronic payments, dynamic linking ties the authentication to the amount and payee. An SMS code alone does not demonstrate those properties [3, 4].

NIST guidance describes technical authentication requirements and assurance levels. It is not a general US privacy law or a substitute for identifying the legal obligations of a particular healthtech service. Under SP 800-63B-4, AAL2 requires two distinct authentication factors and an available phishing-resistant option. A manually entered SMS OTP does not supply that phishing resistance [1].

These distinctions change the implementation task. The first work item is an applicability decision, followed by an authentication policy. Buying a different message transport cannot answer either question.

For an application that also takes payments, keep the payment decision attached to the payment operation. A report dispatch check should not claim to implement transaction-specific requirements that its inputs do not even contain.

## A focused release check before sending the attachment

The following TypeScript is an illustrative application policy, not a compliance test. It assumes trusted server-side records: the report service supplies ownership, the account service supplies the verified destination, and the authentication service supplies the method and timestamp. Its freshness window of 300 seconds is an example policy choice, not a value prescribed by the cited regulations.

```ts
type ReleaseInput = {
  actorId: string;
  reportOwnerId: string;
  reportState: "draft" | "approved";
  destinationVerified: boolean;
  destinationChangedAtMs: number | null;
  authenticatedAtMs: number;
  authentication: "sms-otp" | "phishing-resistant";
};

type ReleaseDecision =
  | { allowed: true }
  | {
      allowed: false;
      reason: "ownership" | "report" | "destination" | "step-up";
    };

function authorizeAttachment(
  input: ReleaseInput,
  nowMs: number
): ReleaseDecision {
  if (input.actorId !== input.reportOwnerId) {
    return { allowed: false, reason: "ownership" };
  }
  if (input.reportState !== "approved") {
    return { allowed: false, reason: "report" };
  }
  if (!input.destinationVerified) {
    return { allowed: false, reason: "destination" };
  }

  const ageMs = nowMs - input.authenticatedAtMs;
  const destinationChanged =
    input.destinationChangedAtMs !== null &&
    input.destinationChangedAtMs >= input.authenticatedAtMs;

  if (
    ageMs < 0 ||
    ageMs > 300_000 ||
    destinationChanged ||
    input.authentication !== "phishing-resistant"
  ) {
    return { allowed: false, reason: "step-up" };
  }
  return { allowed: true };
}
```

For an illustrative generated report, “approved” means the application has completed its own release process. It does not mean an LLM output became medically valid because a boolean changed. Clinical review requirements belong in that upstream process, with the release function consuming its result.

This example deliberately requires phishing-resistant authentication for attachment release even if SMS remains available for some other account operations. That is a proposed policy for sensitive exports. The trade-off is extra enrollment and recovery work in exchange for protection against code relay, so the integration estimate must include both. A label such as “phishing-resistant” must reflect a verified authentication ceremony, not a client assertion. The simple owner check also has a clear limit: delegated access is outside this example. A real caregiver or clinician workflow needs a relationship-aware authorization check. Removing the comparison to accommodate delegation would silently remove the only report access rule in the function. For example, a proposed caregiver flow must establish the caregiver's active relationship to the patient and the scope of report access before it reaches this dispatch function; a matching telephone number cannot substitute for that relationship. Adding that scope is a product authorization decision with consequences for recovery and revocation, which is why it deserves a separate implementation estimate.

Don't skip recovery.

Run the decision in the sending worker against current records. If a queue contains an old authorization decision, a later change to report access or the destination can otherwise be missed. Bind the dispatch record to the report version and destination used for the decision. Retries also need duplicate-send controls; a successful login does not solve delivery bookkeeping.

That is extra integration work, but it stays small enough to reason about. The report generator need not know which authentication transport produced the session, and the email adapter need not decide who owns a report.

## Measure the release path before copying this policy

Start with the cases that could expose the attachment: another account's report, an unapproved version, an unverified destination, a destination changed after authentication, and a stale authentication result. Include the exact freshness boundary and a clock timestamp from the future. Then test recovery and delegated access separately. These are proposed tests, not claims about results already obtained.

Email adds a limitation that login controls cannot remove.

Once an attachment has been delivered, changing application permissions cannot retrieve that delivered copy. Assess whether the recipient, mailbox, retention expectations, and attachment handling fit the information being sent. If the workflow cannot accept that property, reconsider the attachment requirement before investing in more OTP plumbing.

Domain authentication addresses another layer. SPF authorizes sending hosts to use a domain in the SMTP identity it checks; RFC 7208 does not make SPF a recipient authorization check or an attachment confidentiality mechanism [5]. Likewise, messaging industry guidance is useful for delivery practices, but it does not establish that a particular health-data release satisfies privacy or payment requirements [6].

For a solo builder, the useful cost measures are the added authentication delay, step-up completion rate, recovery workload, duplicate dispatches, and time spent investigating denied releases. Keep report contents, OTP values, and attachment bytes out of operational logs; record the decision reason and a controlled internal reference instead. Set log retention deliberately.

Before deployment, check that the sender consumes the same report version and destination that the authorization decision evaluated. Measure queue delay as well as authentication latency. A fast challenge followed by a long queue can violate the example's freshness assumption unless the worker rechecks it.

The shipping decision is specific: release a report only after the applicable obligations, authentication assurance, report authorization, and destination checks have been resolved. SMS delivery success answers a much smaller question.

## References

[1] Authentication assurance, restricted telephone authenticators, and phishing resistance: NIST SP 800-63B-4. [2] Personal data security and special categories of data: GDPR Articles 32 and 9. [3–4] Strong customer authentication and dynamic linking: PSD2 and its regulatory technical standards. [5] Sending-domain authorization: RFC 7208. [6] Messaging practices: CTIA.

## Further reading

- [1] https://pages.nist.gov/800-63-4/sp800-63b.html
- [2] https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng
- [3] https://eur-lex.europa.eu/eli/dir/2015/2366/oj/eng
- [4] https://eur-lex.europa.eu/eli/reg_del/2018/389/oj/eng
- [5] https://datatracker.ietf.org/doc/html/rfc7208
- [6] https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
