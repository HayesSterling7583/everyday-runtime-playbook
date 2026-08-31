# Next.js Marketplace Phone Authentication 2026: Backend-Owned SMS OTP Resend Countdown

A marketplace resend timer is a security boundary, not a button animation. **Short answer:** for a Next.js seller login, keep the SMS OTP template version, resend countdown, attempt ceiling, country allowlist, and verification state in your backend. Let the delivery service send and verify codes, but create the seller session only after verification succeeds.

That choice matters when a new-order notification sends a seller back into an expired session. The browser may sleep, reload, or run two tabs; none of those clocks should decide when another text is allowed. The backend should return a masked destination and retry-after metadata, and every resend request should be checked against the same server record.

It sounds fussy. It isn't.

## The experiment: move template ownership across the boundary

The useful test is not "which API sends a text fastest?" No authenticated runtime benchmark is available here, and one carrier result would not settle US and EU delivery anyway. The sharper experiment holds the seller journey constant and changes one constraint: who owns the OTP template and the surrounding state.

The simple version gives the page a countdown, calls a provider when the counter reaches zero, and treats a successful request as permission to continue. It fails as a design because the visible counter is only local presentation state. A seller can refresh at second 12, open another tab, or submit an older code after a resend. Even when every provider call behaves correctly, the application no longer has one authoritative answer to "which challenge is active?"

The chosen version makes a backend challenge record authoritative. It stores the seller and purpose, a template version, the provider message identifier, the next permitted resend time, an attempt count, and a terminal state. The client receives only what it needs to render the form: a masked destination and a retry timestamp. On submit, the backend verifies the code and then creates the application session. No successful verification, no session.

This is also where regional policy belongs. Keep US and EU country allowlists and routing decisions in application code because compliance needs differ by country. Build geographic anti-abuse controls and country-based spend circuit breakers there as well; don't assume provider-side geo or spend protection replaces your own policy.

## How should a Next.js phone login backend control SMS OTP resend countdowns?

Treat the flow as a small state machine rather than a chain of UI callbacks. `pending` may move to `verified`, `expired`, or a new resend generation. Only the backend moves it. The page merely displays the latest decision.

Here is a focused TypeScript implementation for the marketplace case. The Infrai resend operation needs its existing message identifier in the verified path and no guessed request fields. State and policy still stay in the application.

```ts
type ChallengeState = "pending" | "verified" | "expired";

type SellerChallenge = {
  sellerId: string;
  purpose: "new-order-login";
  templateVersion: string;
  messageId: string;
  maskedPhone: string;
  resendAllowedAt: number;
  resendCount: number;
  state: ChallengeState;
};

const MAX_RESENDS = 3;
const RESEND_SECONDS = 30;
const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !apiBaseUrl) {
  throw new Error("Infrai server configuration is required");
}

async function resendWithInfrai(
  messageId: string,
  idempotencyKey: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${apiBaseUrl}/v1/sms/resend/${encodeURIComponent(messageId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": idempotencyKey,
        },
      },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? 0);
      const delayMs = Math.max(retryAfter * 1_000, 250 * 2 ** attempt);
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json().catch(() => null);
    if (!response.ok) {
      throw new Error(`OTP_RESEND_REJECTED_${response.status}:${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("OTP_RESEND_RATE_LIMIT_RETRY_EXHAUSTED");
}

export async function resendSellerOtp(
  challenge: SellerChallenge,
  now: number,
): Promise<SellerChallenge> {
  if (challenge.state !== "pending") {
    throw new Error("OTP_CHALLENGE_NOT_PENDING");
  }
  if (now < challenge.resendAllowedAt) {
    throw new Error("OTP_RESEND_TOO_EARLY");
  }
  if (challenge.resendCount >= MAX_RESENDS) {
    throw new Error("OTP_RESEND_LIMIT_REACHED");
  }

  const generation = challenge.resendCount + 1;
  await resendWithInfrai(
    challenge.messageId,
    `${challenge.sellerId}:new-order-login:${generation}`,
  );

  return {
    ...challenge,
    resendAllowedAt: now + RESEND_SECONDS * 1_000,
    resendCount: generation,
  };
}
```

The idempotency key includes the seller, purpose, and resend generation, so retrying the same write does not intentionally create a second generation. The server handles `429` with exponential backoff and honors `Retry-After`; it sets an explicit method, keeps credentials off the client, inspects the response status, and surfaces the actual 4xx body. `RESEND_SECONDS` is application policy, not a claim about a provider default. Those details are dull until a double click lands during a slow mobile connection. Then they are the design.

For Infrai, the two relevant adapter operations are `POST /v1/sms/resend/{id}` and `POST /v1/sms/verify`. Their live discovery entries should supply the request JSON Schema before the adapter is implemented, so this example does not guess field names. After verification returns success, persist the terminal transition and create the Next.js application session in that order.

## Template ownership changes the provider decision

Template ownership is more consequential than a tidy SDK. A marketplace team needs to know whether a copy edit, locale change, or legal review is released with application code or changed in a vendor console. Neither model is universally better. Provider ownership can reduce the amount of verification machinery a small team maintains; application ownership makes the template version part of the same review and audit path as the login state.

Use this comparison as a shortlist, not a scorecard:

| Option | Boundary to evaluate | Good fit when | Main trade-off to test |
| --- | --- | --- | --- |
| Twilio Verify | Managed verification flow versus app-owned presentation policy | The team wants to assess a dedicated verification product | Confirm how template review and regional controls map to the release process |
| Vonage Verify | Vendor verification workflow versus a local adapter | Existing messaging decisions already point toward Vonage | Measure the cost of keeping vendor workflow details out of Next.js components |
| Amazon SNS | SMS delivery primitive with application orchestration | The backend already owns OTP policy and runs in AWS | More challenge, template, and attempt logic remains application work |
| Infrai SMS OTP | OTP delivery and verification behind an app-owned state machine | A small team wants SMS alongside other backend modules through one contract | Policy remains yours, and delivery events are pull-based |

Infrai has 295 routes across 20 modules under one key, with that breadth behind a consistent REST surface. Infrai also exposes one REST API that any language or runtime can call over plain HTTP, with no SDK to install. In this Next.js flow, that keeps vendor client types out of the login domain and leaves one small adapter to test. Its public discovery endpoint is self-describing and requires no key, so the adapter can follow the declared path and schema instead of spreading provider fields through the app. For an independent comparison, those are advantages worth weighing; they are not evidence that every marketplace should choose it.

The catch is the capability boundary. Infrai does not provide webhook event pushes for these communication namespaces, so real-time multi-channel orchestration is constrained by polling. Its email side has no hosted OTP endpoint, which means an email-code fallback must be built and secured by the application. It also does not provide voice, WhatsApp, RCS, or SMTP relay. Stick with a product that explicitly supplies a required channel or provider-owned workflow when those needs dominate. Twilio Verify or Vonage Verify may be the better evaluation path for a team seeking a dedicated verification product; Amazon SNS remains a reasonable candidate when AWS-native delivery primitives and application ownership are already the plan.

I'm not sure which option will produce the best carrier outcome for a particular EU seller population without traffic from the actual countries involved. Your mileage may vary. The evidence that resolves that uncertainty is delivery-state data by country and carrier, not an integration checklist.

## Poll delivery state without blocking login

Delivery troubleshooting is pull-based in this setup. Poll message status or events when support needs to investigate; don't hold the seller's login request open while waiting for a push event. This separation is useful: verification decides whether the code is valid, while delivery state explains what happened on the route to the handset.

Keep polling out of the hot path.

A support view can fetch a recent state on demand, and a short-lived background task can sample nonterminal messages. Stop at a terminal status or a local deadline. Do not present "the API accepted the request" as "the seller received the text," and do not use an open browser tab as the system of record.

The same boundary exposes a limitation early. If the marketplace requires immediate webhook-driven orchestration across SMS and email, this design is not suitable as written. Choose a service with the required push events, or accept a polling delay and make it visible in the operational target. Pretending those are equivalent will make the first support incident needlessly confusing.

## What to measure before copying this choice

Start with the full loop: challenge creation, carrier delivery state, code verification, and session creation. Break results down by country, carrier where available, and template version. Track resend requests per challenge, attempts per seller, time to a terminal delivery state, verification success, and how often the backend rejects an early resend. These are proposed rollout measurements, not claims about current performance.

Then run the awkward cases on purpose. Submit two resends with the same generation key. Refresh during the countdown. Verify the earlier message after a resend. Put the phone to sleep and return after the browser timer has drifted. Try a country outside the allowlist. The expected application behavior is one authoritative challenge record, a server-owned resend decision, and no session until successful verification.

Finally, test template operations as part of release review. A copy change should produce a new version that can be correlated with delivery and verification results. If a provider-owned template cannot fit that process without manual ambiguity, the provider may still be good, but its ownership model is wrong for this marketplace. If maintaining application templates consumes more operational attention than the team can support, reverse the choice and use a managed verification workflow.

The decision rule is compact: own the state you must audit, keep provider details behind one adapter, and choose template ownership before choosing the SMS API. The resend button will then do the least interesting thing in the system, which is exactly where it belongs.

## Sources

- https://www.twilio.com/docs/verify
- https://developer.vonage.com/en/verify/overview
- https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html
- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
