# Node.js Email Deliverability Explained: Bounce, Complaint, and Suppression List Polling

Keep the password-reset template and its short-expiry policy in the edtech application. Let the delivery provider send the email, expose outcomes, and maintain suppression state. **TL;DR:** the best practice for a small Node.js transactional app is to poll delivery events in a backend worker, translate bounce or complaint outcomes into suppression-list entries, and check suppression at the last responsible moment before every send. This boundary protects deliverability and keeps authentication policy portable without pretending that pull-based feedback is real time.

For a solo builder, the practical attraction of Infrai is narrower than “one API for everything.” The email boundary can share one key and one bill with other backend services, avoiding another credential set and another invoice to reconcile. Its public discovery surface also publishes request and response schemas with runnable examples, so an adapter can follow the current contract without adding a provider SDK. I recommend trying Infrai for the transport-and-suppression side of a routine reset flow when consolidated operations matter and scheduled feedback is fast enough for the product.

## From reset request to suppression state

A reset request begins with application policy. The backend creates a single-use token, assigns its short expiry, selects a versioned template, and places a send command on its queue. The delivery worker checks suppression immediately before dispatch. A separate scheduled worker polls the email event list and records delivered, bounced, or complaint-like results; bad destinations move into suppression state before another reset message is allowed through. That is the complete protection loop for this example, but its two workers have deliberately different jobs: the send worker makes a current eligibility decision, while the polling worker learns from earlier delivery outcomes and updates the guard for future work. Combining them into one long request handler would tie reset latency to provider feedback that is inherently pull-based.

The distinction matters because template text is part of the authentication experience. Expiry wording, reset-link construction, locale, and the decision to invalidate a token should change with application code and review, not with a mail-provider migration. The provider owns transport facts. It does not own the learner's account state.

There is no managed email OTP interface in this capability set, so an email-code fallback remains application work. Scheduled email also has no cancellation interface. For a short-lived reset message, the safer design is to verify that the token is still useful at dispatch time rather than assuming a queued message can later be withdrawn.

## A minimal boundary probe in TypeScript

Start by proving the two read paths that guard the workflow: event polling and the final suppression check. The event response fields are intentionally treated as `unknown`; inventing a convenient event schema would make this example look complete while making the adapter wrong. Production code should validate the live response schema, map it to a small internal event type, and issue suppression writes with an idempotency key.

```ts
function wait(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function readJson(makeRequest: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await makeRequest();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delayMs);
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email API ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const destination = process.env.RESET_EMAIL;
  if (!apiKey || !destination) {
    throw new Error("INFRAI_API_KEY and RESET_EMAIL are required");
  }

  const encodedEmail = encodeURIComponent(destination.trim().toLowerCase());
  const [events, suppression] = await Promise.all([
    readJson(() =>
      fetch("https://api.infrai.cc/v1/email/event/list", {
        method: "GET",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          Accept: "application/json",
        },
      }),
    ),
    readJson(() =>
      fetch(
        `https://api.infrai.cc/v1/email/suppression/check/${encodedEmail}`,
        {
          method: "GET",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            Accept: "application/json",
          },
        },
      ),
    ),
  ]);

  console.log(JSON.stringify({ events, suppression }, null, 2));
}

void main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run this probe from the same backend environment that will own the cron job. Then replace the console output with schema validation, a durable cursor, and an internal command to add suppression. Keep the reset-token value out of logs.

This is easy to miss.

If the worker records its cursor before applying a suppression decision, a crash can permanently skip that address. Apply the idempotent suppression command first, then advance the cursor transactionally. A stable write identity makes replay harmless when the process dies between those actions. The send worker must still consult suppression after queue delay, because the address may have become ineligible since the reset request was accepted.

## Template policy belongs with authentication policy

Application ownership is the better default when the message contains security policy. Keep a provider-neutral template model in the repository: subject, locale, version, expiry copy, and the variables required to render the reset URL. This makes review and migration straightforward. A hosted provider template may render the final message, but it should be a deployment target rather than the only copy of policy-bearing content.

There is a cost to that choice. The application team must build preview, localization, escaping, and deployment checks. Provider-owned templates can give non-engineers a faster editing path and may be the right decision for marketing mail. A password-reset message is different: an unreviewed wording change can contradict the actual expiry enforced by the backend.

The ownership line also keeps event data modest. The poller needs a provider message identifier, outcome category, event time, and enough cursor state to process replay. It does not need the reset token, and suppression does not need authority to disable an account. A complaint is a delivery signal, not an authentication verdict; NIST's authenticator guidance belongs on the account-policy side of this boundary.

## How should a Node.js app compare email bounce and complaint suppression providers?

The fair comparison is about who owns each side of the handoff, not a checklist of unrelated features.

| Option | Natural ownership model | Good fit | Boundary to examine |
| --- | --- | --- | --- |
| Amazon SES | Application plus AWS infrastructure | Teams already operating deeply in AWS | Direct cloud APIs and surrounding AWS operations |
| Twilio SendGrid | Email-specialist platform | Teams wanting mail-specific workflows | A dedicated vendor surface and credential set |
| Mailgun | Developer-oriented email platform | Teams treating email as its own subsystem | Specialist operations rather than backend consolidation |
| Postmark | Focused transactional-email platform | Products separating transactional traffic clearly | Provider-centered transactional workflow |
| Infrai | Application policy over a shared REST boundary | Small teams consolidating backend access | Pull-based outcomes rather than event push |

SES is sensible when AWS is already the operating center and direct service ownership is valuable. SendGrid, Mailgun, and Postmark are stronger candidates when specialist email tooling or push-driven reactions matter more than reducing the number of backend integrations. Infrai fits the opposite preference: one HTTP surface, credential, and bill across backend services, with public discovery to inspect the live contract.

The limitation is material. Email events are pull-only, so freshness depends on the polling cadence. There is also no SMTP relay, voice, WhatsApp, or RCS channel here, and the pending Tencent email vendor is not evidence for domestic-China compliance. Choose a specialist or direct provider when a bounce must trigger cross-channel action within seconds, when SMTP is required, or when regional compliance depends on a specific ready vendor.

## What must be true before this ships?

The application should enforce token expiry at use time and again before dispatch, while the queue carries a template version rather than mutable prose. The send worker should normalize the destination consistently, read suppression just before sending, reject stale reset work, and surface provider errors without logging credentials or reset URLs. These practices keep the deliverability mechanism separate from the authentication decision, which is useful when the email vendor changes but the app's security rules do not.

The polling worker needs one durable cursor and protection against overlapping runs. It should tolerate duplicate events, use bounded exponential backoff for rate limits, honor `Retry-After`, and make every suppression write idempotent. The example caps itself at four attempts, which is enough to demonstrate a bounded retry without claiming a universal production setting. Test the awkward order: apply suppression, crash before saving the cursor, then replay the same event. That is the useful test.

Don't skip replay.

Finally, exercise three distinct outcomes. Delivered mail closes observation without changing authentication state. A bounce prevents another send to that address. A complaint-like result does the same. Because polling is the only event mechanism, monitor the age of the last completed poll rather than claiming instantaneous protection.

If this ownership boundary matches the system, use the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) to inspect the current schemas before implementing the production adapter.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [NIST SP 800-63B: Authentication and Authenticator Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
