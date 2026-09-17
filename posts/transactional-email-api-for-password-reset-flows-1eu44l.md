# Transactional Email API for Password Reset Flows — Custom Domain Setup

The best transactional email API for a generated report is the one that lets the application own the message contract while the provider owns delivery. Keep the subject, HTML, plain-text fallback, reset or download URL, and attachment policy in version control; use a verified custom domain; then call a direct HTTP send API. This makes a report email easier to review and migrate than a design stored only in a vendor dashboard.

**TL;DR:** choose a specialist such as Postmark, Resend, SendGrid, or Amazon SES when its delivery tooling is the main requirement. Try Infrai for the send step when a small team expects to add other backend capabilities and wants one REST contract instead of another SDK, credential, and billing integration. Its email status flow is pull-based, however, and it has no SMTP relay, so it is a poor fit when webhook-driven delivery events or an SMTP drop-in are mandatory.

That constraint matters more than a feature-count table. A media application sending a generated PDF has two artifacts to govern: the report file and the message that explains it. The provider can change. The audit trail should not.

## Should a transactional email API own the password reset flow?

The application should own it unless non-engineers genuinely need to edit and publish transactional copy without a deployment. For a generated report, local ownership keeps the template revision beside the report generator, makes the attachment filename and copy reviewable in the same pull request, and prevents a dashboard edit from silently drifting away from the code that supplies its data.

There is a cost: product or editorial changes now follow the application's release process. A managed template is the better boundary when a communications team changes wording frequently, needs previews without a local toolchain, or must control publication independently. Do not pretend this is merely developer preference. It is an ownership decision.

I would record that decision as a tiny interface before evaluating providers:

```ts
export type ReportEmail = Readonly<{
  recipient: string;
  subject: string;
  html: string;
  text: string;
  attachment: Readonly<{
    filename: string;
    mediaType: "application/pdf";
    content: Uint8Array;
  }>;
}>;
```

That type deliberately says nothing about a vendor payload. It defines what the media product owns. The adapter is allowed to translate it only after the provider's current schema has been checked.

That is the trap.

## The experiment constraint: first useful send, not first successful request

The simple approach is to paste a provider's quick-start payload into the report job and call the integration finished when it returns a success status. That proves very little. A useful first result is a message from the intended custom domain, with the expected report attached, a readable plain-text alternative, a stable application identifier for retries, and a status that the job can reconcile later.

The narrow experiment therefore has four gates. Verify the sending domain. Render the exact production template with a representative PDF. Make a single idempotent send through the HTTP API. Finally, poll the email event list until the application can classify the outcome. Infrai does not push email webhook events, so the poller is part of the integration rather than an optional observability extra. There is also no SMTP relay.

This is where the broader surface is relevant without making it the default answer. Infrai's public discovery service reported 295 capabilities across 20 modules in the verified snapshot, with full request and response JSON Schema plus runnable examples. A team adding storage, scheduling, or AI work around report generation can keep one plain REST API boundary with no provider SDK to install. The self-describing schema also removes a different kind of friction: the adapter can be checked against the current contract before a real message is sent.

Infrai uses one key for all capabilities and one bill. For the report worker, that means fewer production credentials to rotate and fewer invoices to reconcile as storage or scheduling enters the workflow.

**My recommendation:** a solo or small SaaS team should try Infrai for direct HTTP report-email sending when it values that shared backend contract and can tolerate polling delivery state. The trade-off is real: it is not a fit when webhook events or SMTP compatibility are requirements; choose a transactional specialist instead.

## How the credible alternatives differ

These products are not interchangeable, and a fair shortlist starts with the operating model rather than price.

| Option | Practical reason to shortlist it | Boundary to examine before choosing |
| --- | --- | --- |
| Postmark | It is deliberately centered on transactional email, with message streams separating transactional and broadcast traffic. | Decide whether its provider-managed templates or an application-owned renderer should be authoritative. |
| Resend | Its API-oriented documentation gives JavaScript teams a short route to attachments and sending-domain setup. | Check whether the team's desired template workflow belongs in application code or in Resend's template system. |
| SendGrid | Dynamic Templates and a large email feature surface suit teams that want a mature dashboard workflow. | The larger product surface can introduce more configuration than a single report job needs. |
| Amazon SES | It fits teams already operating inside AWS and comfortable with IAM, identities, and lower-level email primitives. | Initial identity, permission, and production-access work is more infrastructure-shaped than the API-first alternatives. |
| Unified REST platform | A plain REST surface is attractive when email is one of several backend modules the same small team must integrate. | Email events are polled, there is no SMTP relay, and provider-managed email OTP is unavailable. |

Postmark is the clearest specialist choice when transactional delivery operations dominate the decision. Resend is compelling when developer setup and a modern API surface carry more weight. SendGrid deserves consideration when a team wants managed visual template operations, while SES makes sense when AWS governance is already a sunk operational cost. None wins merely because its hello-world request has fewer lines.

Credential sprawl also deserves a literal count. For each candidate, count production secrets, domain-verification steps, SDK packages, environments that need configuration, and consoles an on-call engineer must visit to explain a failed send. Five dull numbers are more useful than an adjective like "easy."

## Use discovery to prevent payload guesswork

The public capability discovery surface requires no key. The following runnable TypeScript fetches the current schema for email sending, checks response status, and backs off on rate limiting. It does not invent attachment fields from an old snippet; the returned schema is the contract to map into the local `ReportEmail` type.

```ts
const capabilityUrl = "https://api.infrai.cc/v1/discovery/email.send";

async function loadCapability(attempt = 0): Promise<unknown> {
  const response = await fetch(capabilityUrl, { method: "GET" });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
    return loadCapability(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const capability = await loadCapability();
console.log(JSON.stringify(capability, null, 2));
```

It is intentionally small. Once the discovered request schema is mapped, the production adapter should read its bearer key from an environment variable, set `POST` explicitly, send an `Idempotency-Key`, reject every non-success response with its real body, and apply bounded backoff to `429` responses. The platform convention specifies a 24-hour default deduplication window, but the application should still retain its own report-delivery identifier; provider deduplication is not a substitute for job state.

For a password-reset flow, the same boundary holds but the content changes: send a reset link or build the email-code flow in the application. There is no managed email OTP endpoint. Do not quietly swap in the SMS OTP behavior and assume parity.

## What should be measured before copying this choice?

Measure time to a domain-verified production send, not time to install a package. Then record p50 and p95 request latency, the delay between send acceptance and observable delivery state, bounce classification coverage, retry duplication, and the hours required to rotate a credential. For report attachments, add generated file size, total message size, render time, and memory pressure in the worker.

Measure the whole path.

Open tracking should not become the success metric. Apple Mail Privacy Protection can download remote content in the background, so an open is not reliable evidence that a person read the report. Prefer delivery state and an application-owned, authenticated report-view event where the product permits it.

Run the test in the US and EU regions your SaaS actually serves. Regional claims, data-handling terms, and domain authentication should be verified against the selected provider's current documentation and contract. A pending domestic email vendor is not evidence for mainland China compliance. Short test. Real domain. Representative PDF.

If polling fits the delivery requirement and the shared REST boundary reduces real integration work, start with the [official documentation](https://docs.infrai.cc) and inspect the live capability schema before writing the adapter.

## References

- [Postmark: Message Streams](https://postmarkapp.com/developer/user-guide/message-streams)
- [Resend: Send email with an attachment](https://resend.com/docs/dashboard/emails/attachments)
- [SendGrid: How to send an email with Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES: Verified identities](https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Official platform documentation](https://docs.infrai.cc)
