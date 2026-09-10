# Logistics 2FA SMS OTP: Geo Fences, Toll-Fraud Signals, and Rate Budgets

Short answer: to prevent SMS OTP abuse and toll fraud in a US/EU SaaS login, rate-limit each identity and destination, apply a country rule before delivery, and retain evidence for every 2FA decision.

A logistics SaaS login has an awkward failure mode: a stolen credential can be useful, but a scripted OTP endpoint can also turn your account into a phone-billing relay. A driver signing in from Ohio and an attacker spraying destinations from a cloud subnet should not consume the same budget. The control plane needs to know who asked, which account they touched, where the request claims to originate, and why a message was sent. 

## How can a logistics team prevent SMS OTP abuse and toll fraud?

Start with an event, not a provider call. Create an immutable request ID, normalize the phone number to E.164, and attach tenant, user, IP, ASN, device, country, and purpose fields. The purpose matters for compliance: “dispatch report access” is a different risk decision from “change payout number.”

The decision service then evaluates three independent signals. Identity limits stop one account from generating a burst. Destination limits stop a single number from being hammered across accounts. Spend limits stop a tenant or the whole service from creating an unexpected bill. A country allowlist is a policy input, not proof that the user is legitimate; an allowed US number can still be automation.

Persist the decision and its inputs before enqueueing delivery. That ordering gives an auditor a useful trail even if the downstream messaging system is slow. It also lets operations answer the practical question: did we block a suspicious request, or did we merely fail to send it?

That is the evidence boundary.

## A small TypeScript gate that keeps policy separate from delivery

The example below uses generic interfaces so the same policy can sit in front of different SMS gateways. It deliberately returns a reason code for every branch.

```ts
type OtpRequest = {
  requestId: string;
  tenantId: string;
  userId: string;
  phoneE164: string;
  country: string;
  ip: string;
  asn: number;
};

type Decision = {
  allowed: boolean;
  reason: string;
};

const allowedCountries = new Set(["US", "DE", "FR", "NL"]);

export function decideOtp(req: OtpRequest, nowMs: number, recent: number[]): Decision {
  if (!allowedCountries.has(req.country)) {
    return { allowed: false, reason: "country_policy" };
  }

  const windowMs = 10 * 60 * 1000;
  const recentCount = recent.filter((t) => nowMs - t < windowMs).length;
  if (recentCount >= 3) {
    return { allowed: false, reason: "user_window_limit" };
  }

  return { allowed: true, reason: "policy_pass" };
}

export async function requestOtp(req: OtpRequest, send: (to: string) => Promise<void>) {
  const now = Date.now();
  const decision = decideOtp(req, now, await loadUserTimestamps(req.userId));
  await appendAudit({ ...req, ...decision, decidedAt: new Date(now).toISOString() });
  if (!decision.allowed) return decision;

  await send(req.phoneE164);
  return decision;
}
```

In production, loadUserTimestamps and appendAudit should use durable storage with an idempotency key based on requestId. The send operation belongs behind a queue with a bounded retry policy. Never retry a rejected decision, and never let a client retry loop bypass the server-side request ID. The code’s three-per-ten-minutes value is a starting policy for a low-volume tenant, not a universal truth. Your mileage may vary; tune it from verified login telemetry and carrier guidance.

## Where rate limits and geo blocks fail in practice

A single counter per IP is easy to deploy and easy to evade. Mobile carriers, corporate NAT, and shared warehouse Wi-Fi put many legitimate users behind one address. Use several keys with different windows: user, destination, tenant, IP or ASN, and a global spend bucket. A request should pass only when every applicable bucket has capacity.

Geo decisions have their own traps. IP geolocation can disagree with a phone’s country code, and roaming drivers cross borders during a shift. Use country as a risk signal with a review path, then make the policy explicit: allow US and selected EU countries for the tenant, challenge a new country, or require administrator approval. Do not silently convert a geo mismatch into an endless OTP loop.

It failed in review once.

The costly mistake is coupling “code generated” to “message sent.” Generate a nonce and hash it for storage; consume it once after verification; expire it quickly. Record carrier response class, but avoid logging the OTP itself or placing full phone numbers in general application logs. CTIA guidance and regional privacy rules make data minimization a design requirement, not a cleanup task.

## Which SMS delivery setup fits our compliance evidence needs?

A direct carrier connection can offer control over sender identity and routing, while an aggregator usually offers broader reach and simpler operations. An email fallback may reduce SMS exposure for known corporate users, but it changes the threat model and can weaken account recovery if email is already compromised. A self-hosted gateway gives ownership of the queue, yet it moves regulatory and carrier relationships onto a small team.

The right comparison axis is evidence quality under failure. Ask each option whether it exposes a stable message ID, delivery timestamps, destination classification, retention controls, and exportable logs. Confirm how opt-outs, quiet hours, sender registration, and US/EU data residency are handled. A glossy delivery percentage is less useful than a reproducible record for one denied request and one accepted request.

| Setup | Best fit | Main trade-off |
| --- | --- | --- |
| Direct carrier connection | Tight sender and routing control | More regulatory and carrier operations |
| Messaging aggregator | Broad reach with a small team | Less control over routing and evidence shape |
| Self-hosted gateway | Owning queue and retention behavior | Your team carries delivery and compliance work |

There is a boundary here. SMS is vulnerable to SIM swaps, recycled numbers, and interception, so it is a poor sole factor for high-impact actions such as changing a bank account. It is not suitable when a payout or privileged admin action depends on one message; use a phishing-resistant authenticator or hardware key there, and keep SMS as a bootstrap or recovery factor when the risk assessment permits it.

## An operational checklist for the first week

Before enabling the endpoint, replay synthetic US and EU sign-ins through the decision service and inspect the audit rows. Exercise bursts from one IP, many accounts targeting one number, and a tenant that reaches its spend bucket. Verify that denied requests never enter the delivery queue, that retries preserve the same request ID, and that dashboards distinguish policy denies from carrier rejects.

On day one, set an alert on unusual country mix, destination concentration, and spend velocity rather than on raw message count alone. Review a sample of decisions with support and compliance. I once assumed a low OTP volume meant low risk; the sharper signal was ten accounts sharing one destination in six minutes. That pattern was visible only after joining tenant and phone hashes.

Keep the policy in configuration, version every change, and require a review for allowlist expansion. Do not guess at the threshold. The goal is a boring login: predictable limits, a clear challenge when context changes, and enough evidence to explain the outcome months later.

## Further reading

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms

## References

- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- CTIA messaging interoperability and compliance commitments: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
