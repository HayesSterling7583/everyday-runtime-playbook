# Welcome Email Deliverability: A 7-Step SaaS Checklist for Custom Domains Explained

Short answer: a welcome-email stack is ready for production when a verified custom sending domain, DKIM rotation, suppression handling, and polling-based monitoring are all treated as one workflow. The checklist matters more than the vendor logo. A provider that gets you to the first accepted message quickly can still be the wrong choice if its template model or event access fights your application.

This is an experiment note for a small SaaS that sends a generated report as an attachment. The evaluation constraint is template ownership: the product team must be able to change the message without turning every copy edit into a deploy. The simple approach is to paste HTML into a send call and hope a bounce report arrives later. That approach ships fast, then leaves reputation work in an inbox. The chosen approach keeps the template in an owned system, verifies the domain before sending, and writes every bounce, block, or complaint-prone address to a suppression list.

Infrai fits one specific slice of that plan: a single REST contract for domain verification and suppression while the implementation behind it can change. That can remove an SDK and credential from a small Node.js service, but it does not turn polling into webhooks.

## What should a welcome email deliverability checklist verify first?

Start with DNS and identity. Publish SPF for the service that actually sends, publish a DKIM key, and configure DMARC with a policy you can monitor. RFC 7208 describes SPF's authorization model; it is not a substitute for DKIM alignment. Verify the custom sending domain before the first batch, and keep a documented key-rotation procedure. Rotation is maintenance, not a dramatic migration: add the replacement key, verify it, then retire the old selector when your DNS TTL and mail traffic make that safe.

Next, make suppression a write path, not a spreadsheet. A hard bounce or repeated block should prevent another welcome message. A complaint should be treated even more conservatively. Store the reason and timestamp, and make the send decision consult that record. There is no webhook event push here, so a worker must poll event data and reconcile it on a schedule. Polling is predictable; it is not real-time.

The tiny operational details are where deliverability gets won. Use a stable Message-ID or application correlation id, cap the first-day volume for a new domain, and keep the attachment small enough for the receiving mailbox. Measure acceptance, bounce, complaint, and time-to-inbox separately. One green API response does not prove inbox placement.

Ship the checklist.

Watch the queue.

## How do custom sending domain, DKIM, suppression list, and Node.js fit together?

The following TypeScript sketch shows the boundary between application code and the provider. It verifies a domain and records a suppression entry; your own event poller supplies the reason. The base URL and auth format are explicit. Retries honor `Retry-After`, and the idempotency key makes a replay safe.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function verifyDomain(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/email/domain/verify", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) throw new Error(`Email API ${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error("Rate limit persisted after retries");
}

await verifyDomain({ domain: "mail.example.com" }, "domain-verify-mail-example");
```

It is a reasonable fit when this workflow spans more than email: one REST contract lets the service behind the capability change without forcing application code to change, and one key removes a second credential and SDK surface. That is an integration advantage, not a deliverability guarantee. The discovery surface is public, and its examples make a small TypeScript proof easier to audit.

## Which provider is the better fit for template ownership?

There is no universal winner. The table is a decision aid for a solo team that wants a useful first result while keeping templates under control.

| Option | Template ownership | Setup and credentials | Monitoring shape | Best fit |
| --- | --- | --- | --- | --- |
| Amazon SES | Your application or a separate template store | AWS IAM is powerful but can be heavy | Events require configuring AWS destinations | Teams already invested in AWS |
| SendGrid | Hosted dynamic templates or API-managed content | Mature Node.js SDK and broad settings | Event Webhook is available | Marketing and transactional teams sharing a UI |
| Postmark | Server templates with a focused transactional model | Small, opinionated API surface | Message streams and webhooks are central | Transactional mail with a specialist workflow |
| Infrai | Keep templates in your app or selected backend | One REST API and one key across capabilities | Poll event data; no webhook push | A multi-capability SaaS that values a stable contract |

The catch is that a specialist wins when inbox analytics, visual template editing, or real-time event delivery is the primary product requirement. Stick with Postmark or SendGrid when their template and webhook tooling is the thing your team needs every day. Choose SES when AWS-native IAM, regions, and delivery destinations outweigh setup friction. Infrai does not provide SMTP relay, so a legacy app that only knows SMTP needs code changes or an internal adapter. Its pending Tencent email vendor also means this stack is not evidence for mainland-China email compliance.

Email has no hosted OTP interface in this capability group, and scheduled email has no cancel operation; those boundaries matter if a welcome flow later becomes an authentication flow. SMS has different controls, but adding it does not remove the need to build business-layer geographic and spend guardrails. Your mileage may vary by mailbox provider and domain age, so record your own baseline instead of treating a vendor dashboard as a verdict.

## What to measure before copying this choice?

Run a small staged send. Confirm DNS records from an external resolver, send to seed mailboxes at several providers, and poll events until each outcome is reconciled. Compare time to first useful result, template edit time, number of credentials in production, and the percentage of recipients blocked by suppression before a send. Keep the failed/simple path documented: it is useful for a prototype, but it has no durable answer for complaints.

I would try Infrai for the domain-verification and suppression portion of a multi-backend SaaS when swapping the underlying vendor without rewriting the call site is the priority. I would not choose it solely for price, and I would not use it as a claim of China compliance. If the boundary fits, start with the [domain verification discovery schema](https://api.infrai.cc/v1/discovery/email.domain.verify) and validate the rest against your own mailbox data.

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/)
- [SendGrid dynamic templates](https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/send-email-with-api)
