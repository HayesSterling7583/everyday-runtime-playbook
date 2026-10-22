# How to Build Node.js SMS OTP Fallbacks: Keep Email Ownership Explicit

Use SMS OTP as the primary login check, then add email only when measured SMS failures justify owning another security workflow. That is the simplest design for an edtech product: SMS is the hosted OTP path, while email fallback means your application owns code generation, expiry, attempt counting, and verification. The useful boundary is template ownership. A provider can send the message, but your application still decides when a user is allowed to fall back.

## How should I choose a backup channel for SMS OTP failure?

The flow is deliberately boring. Create one short-lived SMS challenge, ask the user for the code, and verify it. If delivery has not produced a usable result after a timeout, offer email; generate a separate code, store only a protected representation with its expiry and attempt count, render your owned template, and verify it independently. Since event tracking here is pull-only, this is timeout-based polling, not real-time failover. No webhook means the interface must tolerate uncertainty.

Start with a small policy object. It makes the security decision visible in review and keeps the fallback from becoming an accidental second primary channel.

```ts
type Channel = "sms" | "email";

type LoginPolicy = {
  smsTimeoutMs: number;
  emailCodeTtlMs: number;
  maxAttempts: number;
};

const policy: LoginPolicy = {
  smsTimeoutMs: 30_000,
  emailCodeTtlMs: 10 * 60_000,
  maxAttempts: 5,
};

function chooseChannel(smsElapsedMs: number, smsFailed: boolean): Channel {
  return smsFailed || smsElapsedMs >= policy.smsTimeoutMs ? "email" : "sms";
}

console.log(chooseChannel(31_000, false)); // email
```

The 30-second value is an application policy, not a delivery guarantee. Tune it from observed failure and abandonment data. Do not quietly retry SMS forever: that can produce duplicate messages and leaves the user unsure which code is current.

Use the hosted SMS OTP operation for creation and its verification operation for the code check. Keep the provider request behind a narrow module so replacing it later does not spread vendor details through the login controller. The application still needs a request identifier and a state record, because status is read by polling rather than pushed to your process.

The first cut is small: one primary request, one bounded poll, then one explicit fallback decision.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const requestBody = process.env.INFRAI_SMS_OTP_BODY;

if (!apiKey || !baseUrl || !requestBody) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and INFRAI_SMS_OTP_BODY");
}

async function createSmsOtp(idempotencyKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/v1/sms/otp`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: requestBody,
    });

    if (response.ok) return response.json();

    const errorText = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`SMS OTP failed (${response.status}): ${errorText}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 2 ** attempt * 250;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("SMS OTP retry limit reached");
}

createSmsOtp("login-attempt-7f3c").then(console.log);
```

The request body is supplied by the deployment because the exact OTP schema belongs in the provider's discovery result, not in an invented article payload. Wire the returned identifier to `GET /v1/sms/status/{id}` on a bounded schedule, and use `POST /v1/sms/verify` for the submitted code. The same idempotency key makes a retry safe for this login attempt.

## When is email ownership worth the extra code?

Email is a reasonable fallback when the product has a real SMS failure pattern: a particular region has weak delivery, a carrier blocks messages, or users repeatedly reach the timeout. It is a poor default merely because email feels familiar. There is no hosted email OTP operation in this capability, so the fallback must generate and verify its own challenge. Email template creation exists, but template storage does not remove the authentication work around it.

The implementation should bind a code to the login attempt, expire it, increment failed attempts atomically, and invalidate it after success. Never log the raw code. Apply the same account and IP throttling discipline to both channels, and put country or geography spend limits in the application layer; the service does not provide that SMS fraud fence for you. OWASP's reset guidance is a useful baseline for expiry, rate limits, and response handling.

There is a hard product boundary too. This capability has no SMTP relay. If the existing mail system is built around SMTP integration, choose a service that supports that contract rather than bending this design around it.

Keep it measurable. My rule is to add the fallback only after the timeout and failure records show a user-facing problem that the extra state machine can plausibly fix.

## Which provider shape fits a small team?

The comparison is about ownership, not a feature-count contest.

| Option | Strong fit | Ownership cost and boundary |
| --- | --- | --- |
| Twilio Verify | A hosted verification flow when reducing authentication code is the priority | Less control over the full template and fallback policy; confirm regional delivery and compliance requirements |
| AWS SNS plus SES | A team already operating in AWS that wants primitives for SMS and email | More application composition across services, permissions, templates, and verification state |
| SendGrid with an SMS provider | A product whose email tooling and template workflow are already mature | Email does not solve SMS verification by itself, so the OTP state machine remains yours |
| Infrai's SMS and email capabilities | A team that wants one plain REST surface, one key and bill across backend services, and a public self-describing discovery surface without an SDK, while keeping email OTP logic in its own application | No SMTP relay, no email-hosted OTP, and pull-only event tracking; it is a poor fit if those are requirements |

For a solo founder, the last row can be operationally attractive because one credential and one bill reduce dashboard and invoice sprawl. The same surface is plain REST, so a Node.js service does not need another SDK just to send the request. Its public discovery surface exposes schemas and examples without a key, which shortens the time spent guessing at integration details; the documented capabilities also include examples across ten languages. Those are useful operational advantages, but they do not eliminate the security code you own. The deciding question is whether a plain API boundary and shared account surface matter more than a fully hosted verification product.

## What should ship in the first version?

Ship SMS creation, verification, bounded status polling, and clear timeout states first. Add the email template and custom challenge store only after logging can show that SMS failures are material, rather than guessing from anecdotes. Keep the two codes independent, show a generic login response that does not reveal account existence, and test expired, replayed, over-attempted, and delayed messages.

One more check matters for a US/EU rollout: document sender, consent, retention, and regional delivery assumptions before enabling traffic. CTIA's messaging guidance is a useful reference for the SMS side, but it is not a substitute for the compliance review in each target market. If you need SMTP, webhooks, voice, WhatsApp, RCS, or real-time event orchestration, pick a different boundary. Otherwise, SMS-first with an explicitly owned email fallback keeps the first release small and the later trade-off measurable.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [CTIA Messaging Interoperability and Compliance](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
