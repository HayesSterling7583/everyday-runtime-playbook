# Ticketing CAPTCHA Controls: Escalating Friction from Phone Login Risk Signals

A ticket sale has two clocks: the customer's patience and the attacker's retry loop. Put a CAPTCHA in front of every phone login and legitimate buyers pay the tax before they can even see whether tickets remain. Put it only after obvious abuse and a bot may already have consumed verification attempts, created sessions, or reached the purchase queue.

**Short answer:** score the login attempt before sending a one-time code, challenge the uncertain middle with CAPTCHA, and require fresh verification when a session crosses into a high-value ticket action. Treat the challenge as one state in an authentication flow, not as a permanent wall or a substitute for rate limits.

That choice is intentionally narrow. It protects an existing phone-code login without making every customer solve a puzzle during a high-demand window. It also leaves room for the support team to recover accounts without quietly creating an easier route around the same controls.

## How should ticketing bot defense apply CAPTCHA as risk changes?

CAPTCHA placement should follow the action's risk, the confidence of the available signals, and the cost of being wrong. An anonymous visit to an event page is cheap. Sending a phone code consumes a limited attempt and can be abused at scale. Redeeming that code creates an authenticated session. Entering a ticket queue or changing the phone number attached to an account has a different impact again.

The useful mental model is a small policy ladder:

| Decision | Example condition | User experience | Control that still applies |
| --- | --- | --- | --- |
| Allow | Familiar device and ordinary request rate | Send the code | Per-account and per-network throttles |
| Challenge | New device plus elevated retry velocity | Complete CAPTCHA, then send the code | Attempt budget and expiry |
| Step up | Existing session reaches a sensitive sale or recovery action | Verify a fresh code | Session rotation and action authorization |
| Deny temporarily | Attempt budget is exhausted | Generic retry response | Cooldown, logging, and support review path |

Those conditions are examples, not universal thresholds. A venue with infrequent reserved-seat sales has a different traffic shape from a platform opening several general-admission events at once. I'm not sure a fixed score can remain trustworthy across both; a replay of recent sale traffic, labeled by outcome, is what would settle it. The policy should expose which signals affected a decision so the team can tune it without turning a single opaque score into an oracle.

OWASP's Authentication Cheat Sheet supports this layered direction: CAPTCHA can slow automated login attempts, but it should be defense in depth rather than the only control. The same guidance covers login throttling, generic authentication responses, logging, and reauthentication after risky events. That combination matters because a solved or outsourced challenge doesn't erase the request history.

Keep the first branch cheap. Device continuity, account attempt history, network velocity, and proximity to a sensitive action can inform a decision without asking the customer to do anything. Signals aren't proof. Shared mobile networks can make many real buyers look alike, while a distributed bot fleet can make each address look quiet. A policy that treats an IP address as an identity will punish the wrong people.

And don't challenge twice.

If the customer passed CAPTCHA immediately before requesting a code, bind that result to a short-lived login transaction and consume it once. The transaction should identify the intended phone-login attempt without putting the raw phone number in client-visible state. Reusing a challenge result across accounts or across the purchase boundary turns one successful solve into a reusable credential, which is exactly the wrong incentive.

## The simple placement fails at both ends

The simplest implementation puts CAPTCHA on the login form and calls the problem handled. It is easy to explain, easy to screenshot, and poorly aligned with the actual risk. Returning customers on known devices get the same interruption as a burst of first-time attempts. Accessibility failures become authentication failures. During a sale, a challenge provider's extra round trip also sits directly in the path that every buyer must traverse; the design has coupled general availability to a control that only some requests need. The opposite shortcut is no better: wait until several code-verification failures, then show a challenge. By that point the system may already have sent multiple messages. It has spent the resource the control was meant to protect. A bot that rotates target phone numbers can stay below a per-account threshold, and a bot that rotates networks can stay below a per-address threshold. This is why the decision belongs before code issuance, with aggregate budgets evaluated alongside account and network history.

Retries compound quickly.

There is a catch. Risk-based friction adds policy state, observability work, and more cases for support to understand. It is not suitable when the team can't reliably bind a challenge to a single transaction or can't inspect why a decision was made. In that situation, stick with a simpler pre-send challenge for the riskiest event window, plus strict throttling, until the transaction and logging model exists. Predictable friction is better than a sophisticated rule set nobody can operate.

Account recovery needs the same scrutiny. A customer who lost access to a phone will contact support at precisely the moment when phone verification cannot work. Support must not bypass the attempt budget just because the customer can quote an order number. Route recovery through a separately reviewed process, use generic public responses so account existence is not disclosed, and record the security-relevant decision. The goal isn't to make recovery impossible. It is to avoid building an unmetered side door around login.

## Make the risk decision an explicit transaction

A practical implementation separates signal collection, policy evaluation, challenge verification, code issuance, and session creation. Each transition accepts a transaction identifier and returns a small outcome. The browser doesn't decide whether its own CAPTCHA was sufficient; the server does, after checking that the result belongs to the current transaction and intended action.

Here is a focused TypeScript sketch. The numbers are deliberately illustrative starting values for replay tests, not claimed benchmarks:

```ts
type LoginSignals = {
  knownDevice: boolean;
  accountAttempts10m: number;
  networkAttempts1m: number;
  saleStartsWithin5m: boolean;
};

type RiskDecision =
  | { action: "allow"; reasons: string[] }
  | { action: "challenge"; reasons: string[] }
  | { action: "deny_temporarily"; reasons: string[] };

function decidePhoneLogin(signals: LoginSignals): RiskDecision {
  const reasons: string[] = [];
  let score = 0;

  if (!signals.knownDevice) {
    score += 2;
    reasons.push("new_device");
  }
  if (signals.accountAttempts10m >= 4) {
    score += 4;
    reasons.push("account_attempt_budget");
  }
  if (signals.networkAttempts1m >= 20) {
    score += 3;
    reasons.push("network_velocity");
  }
  if (signals.saleStartsWithin5m) {
    score += 1;
    reasons.push("high_demand_window");
  }

  if (score >= 7) return { action: "deny_temporarily", reasons };
  if (score >= 3) return { action: "challenge", reasons };
  return { action: "allow", reasons };
}
```

The code is intentionally boring. Good. The important part is the contract around it: counters must update atomically enough for the chosen abuse model; a passed challenge must be single-use; a sent code must have a bounded verification budget; and successful authentication must rotate the session identifier. The policy reasons belong in protected logs, while the client gets a generic state such as `challenge_required` or `retry_later`. Returning “phone number not found” on one branch and “wrong code” on another creates an enumeration signal even if CAPTCHA is perfectly placed.

A client state machine can now handle `allow`, `challenge`, and `deny_temporarily` without learning the score. On `challenge`, it completes the accessible challenge flow and resumes the same transaction. On `deny_temporarily`, it does not hammer the code-send action. For an authenticated customer entering the ticket queue, the server evaluates freshness separately; prior login is context, not permanent authorization for every later action.

This separation also limits lock-in. The application owns the risk decision and transaction states, while a challenge adapter supplies only a verified result. Changing challenge mechanisms should not rewrite the login policy, session model, or support tooling. It still costs engineering time, but the boundary is small enough to test.

## Measure customer harm before widening the challenge

Do not launch by watching only “CAPTCHAs passed.” That metric can rise while the system makes no security progress. Measure the complete path: code-send requests by decision, challenge completion and abandonment, verification attempts per transaction, successful logins, queue entry, recovery contacts, and the share of decisions attributed to each policy reason. Segment known and new devices, and separate ordinary days from high-demand sale windows.

The failed/simple approach and the risk-based approach should be replayed against the same retained, privacy-reviewed event shape before traffic is shifted. Then roll out in stages and keep a kill switch that changes policy, rather than bypassing rate limits or accepting an unverified transaction. A false positive is not an abstract score here; it is a buyer who cannot reach a time-sensitive sale. A false negative is a retry loop that can consume messaging capacity or create sessions. Both belong on the launch dashboard.

Use support reports as evidence, too. A spike in “code never arrived” can indicate delivery trouble, aggressive throttling, user confusion, or abuse consuming attempt budgets. The report alone doesn't distinguish them. Correlate it with decision reasons and transaction outcomes before relaxing a control.

The final decision rule is plain: challenge before sending a phone code when several independent signals make the attempt uncertain; step up again only when session freshness is inadequate for a sensitive ticket action; temporarily deny when an explicit attempt budget is exhausted. Keep CAPTCHA out of low-risk paths, but never let its success override throttling, transaction binding, or authorization.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
