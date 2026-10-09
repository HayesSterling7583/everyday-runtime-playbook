# Logistics SaaS App Evidence: SMS API Delivery Polling Across US and EU

TL;DR: For a logistics SaaS that must send compliance notices in the United States and European Union, choose an SMS API by the evidence you can retain with little integration code: a stable message identifier, documented terminal states, explicit failure details, and a read endpoint suitable for bounded polling. Treat provider acceptance and handset delivery as different events. Store every observed transition beside the notice version and recipient snapshot, stop polling at a deadline, and route unresolved cases to review. That is a more useful selection test than counting SDK methods.

The data flow is small on purpose. The application freezes the notice text and destination snapshot, asks an adapter to send it, stores the returned provider reference, and schedules status reads with increasing delays. Each read appends an observation rather than overwriting history. A terminal outcome closes the attempt; a deadline produces an unresolved outcome and a human-review task. The provider remains a transport dependency, while the application owns the audit record.

## How should a Node.js SaaS app poll an SMS alerts API?

Start with one real workflow: a carrier has to receive a changed hazardous-material handling instruction, and an operator later needs to show what was sent, to which normalized destination, and what the transport reported. The proof target is not "the API returned 200." An accepted submission can still be pending, rejected later, or remain unresolved when the review window closes.

This distinction shapes the buying test. A candidate must expose an immutable message reference and let the application retrieve status without requiring an inbound webhook. Its documentation must distinguish transient states from terminal ones. Unknown values must be preservable, because silently mapping a new provider status to "delivered" corrupts the evidence.

Keep those events separate.

The integration should also keep authentication traffic out of this workflow. NIST SP 800-63B treats use of the public switched telephone network for out-of-band authentication as a restricted authenticator and requires verifiers to consider risks such as device swapping and number porting. A logistics notice is not an authentication factor, so model it as a notification with its own policy rather than borrowing OTP assumptions.

## Build the evidence path before comparing dashboards

The following TypeScript keeps provider vocabulary at the adapter boundary. It records raw status alongside the normalized state, uses a deadline, and avoids an endless worker loop. The sample uses an in-memory transport so it runs without a vendor account; replacing that class is the deliberate integration boundary.

```ts
import { createHash, randomUUID } from "node:crypto";

type DeliveryState = "accepted" | "pending" | "delivered" | "failed" | "unknown";

type StatusReading = {
  state: DeliveryState;
  raw: string;
  observedAt: string;
  detail?: string;
};

interface SmsTransport {
  send(input: { to: string; body: string; clientRef: string }): Promise<{ messageRef: string }>;
  read(messageRef: string): Promise<{ state: DeliveryState; raw: string; detail?: string }>;
}

type AuditRecord = {
  noticeId: string;
  clientRef: string;
  messageRef: string;
  destinationHash: string;
  bodyHash: string;
  readings: StatusReading[];
  outcome: "open" | "delivered" | "failed" | "unresolved";
};

const sha256 = (value: string) =>
  createHash("sha256").update(value, "utf8").digest("hex");

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function sendAndObserve(
  transport: SmsTransport,
  input: { noticeId: string; to: string; body: string },
  options: { deadlineMs: number; delaysMs: number[] },
): Promise<AuditRecord> {
  const clientRef = randomUUID();
  const sent = await transport.send({ to: input.to, body: input.body, clientRef });
  const record: AuditRecord = {
    noticeId: input.noticeId,
    clientRef,
    messageRef: sent.messageRef,
    destinationHash: sha256(input.to),
    bodyHash: sha256(input.body),
    readings: [],
    outcome: "open",
  };
  const expiresAt = Date.now() + options.deadlineMs;

  for (const delayMs of options.delaysMs) {
    if (Date.now() + delayMs > expiresAt) break;
    await wait(delayMs);
    const status = await transport.read(record.messageRef);
    record.readings.push({ ...status, observedAt: new Date().toISOString() });

    if (status.state === "delivered" || status.state === "failed") {
      record.outcome = status.state;
      return record;
    }
  }

  record.outcome = "unresolved";
  return record;
}

class DemoTransport implements SmsTransport {
  private reads = 0;

  async send(): Promise<{ messageRef: string }> {
    return { messageRef: "demo-message-001" };
  }

  async read(): Promise<{ state: DeliveryState; raw: string }> {
    this.reads += 1;
    return this.reads < 2
      ? { state: "pending", raw: "queued" }
      : { state: "delivered", raw: "delivered" };
  }
}

const result = await sendAndObserve(
  new DemoTransport(),
  {
    noticeId: "hazmat-rule-42-rev-3",
    to: "+15550100100",
    body: "Handling rule 42 changed. Review revision 3 in the carrier portal.",
  },
  { deadlineMs: 30_000, delaysMs: [500, 1_000, 2_000, 4_000] },
);

console.log(JSON.stringify(result, null, 2));
```

The hashes in this example are correlation aids, not a complete privacy or compliance design. A hash of a predictable phone number can still be guessed, so the production record should use an approved keyed transformation or protected lookup design when reversibility is unnecessary. Retention, access, and deletion rules belong in the application policy. The important architectural point is narrower: status history and notice identity should survive independently of a provider dashboard.

Keep the adapter strict. If the remote system returns a state your mapping does not recognize, store the raw value and normalize it to `unknown`. Do not throw the observation away. Do not promote it to success.

Unknown means unknown.

## Compare integration effort with a proof matrix

A short proof matrix exposes hidden work better than a feature checklist. Fill it from candidate documentation and a sandbox test; do not award credit for an ambiguous marketing label.

| Question | Acceptable evidence | Integration cost if absent |
|---|---|---|
| Can a sent message be read by its stable reference? | Documented lookup plus a repeatable test | A webhook receiver, signature validation, replay handling, and public ingress |
| Are terminal and nonterminal states documented? | A finite state table with failure meanings | Guesswork in retry and escalation logic |
| Can the client submit its own correlation value? | The value returns in later reads or exports | A separate mapping table and harder reconciliation |
| Are duplicate submissions addressed? | A documented idempotency mechanism or duplicate behavior | Application-side deduplication around uncertain send results |
| Can raw outcomes be exported? | Machine-readable retrieval under a stated retention policy | Periodic capture jobs or an incomplete audit trail |
| Are US and EU destinations supported under the required sender rules? | Country-specific documentation for the intended traffic | Separate transports or manual exceptions |

Run the same fixture set against every serious candidate: one valid destination, one invalid destination, one repeated client reference, one status that remains pending until the local deadline, and one simulated read failure. Five cases are enough to expose a surprising amount of adapter complexity. They do not prove carrier delivery performance, but they do show whether the contract can support the evidence model.

My integration preference is blunt: I would accept a little more polling code to keep public ingress out of a small SaaS app, but I would reject any API whose polling contract hides terminal-state meanings. That trade-off reduces setup surface without weakening the record.

Integration effort includes operations. Polling creates read traffic, so the schedule must be bounded and observable. Batch due reads where the API permits it, apply jitter across workers, cap concurrency, and respect documented throttling signals. A worker restart must resume from persisted `nextCheckAt` data rather than restart every message at the shortest delay. This is where a supposedly tiny integration can become expensive in engineering attention and background jobs.

Do not optimize around a published per-message price before this path works. A lower transport charge cannot compensate for an ambiguous final state or weeks spent building reconciliation. Measure cost per closed notice attempt, including status reads, storage, exception handling, and operator review.

## Failure semantics decide whether the record is credible

There are three separate questions in every attempt: did the application submit a request, did the transport accept responsibility for it, and what delivery outcome did the transport later report? Preserve the answer to each. Collapsing them into one `sent` boolean makes retries dangerous because a timeout after submission may leave the application unsure whether another send would duplicate the notice.

Retry reads freely within a controlled policy because they do not create another message. Retry sends only under a documented idempotency contract or after reconciliation shows that the first attempt did not create a message. This is an explicit trade-off: a slower manual exception is often preferable to sending the same compliance notice twice and then defending an audit trail that cannot explain why.

A reported delivery outcome is still transport evidence, not proof that a person read or understood the notice. Phrase user interfaces and exports accordingly. `Delivered`, `failed`, and `unresolved` are different outcomes; unresolved is not a softer spelling of failed.

Words matter here.

Keep email evidence separate too. RFC 7489 defines DMARC for email authentication, policy, and reporting through alignment with SPF and DKIM. It does not validate an SMS sender or establish handset delivery. If the workflow falls back to email, record that as a new channel attempt with its own identifiers and controls rather than attaching an email result to the SMS attempt.

## Operate the notice trail as a small ledger

Deployment begins with adapter contract tests and a migration for append-only observations. Release one destination region at a time, verify that unknown raw states trigger review, and alert on growing age in `open` records rather than on worker errors alone. The useful service-level signal is the distribution of time from acceptance to a terminal or unresolved outcome. Keep transport latency separate from queue delay inside the application.

The on-call view should answer a concrete question without opening a vendor console: which notices are still open, when is each next read due, what was the last raw response, and which deadline will expire first? Log correlation references, not message bodies or full phone numbers. Limit who can retrieve the underlying notice and destination, and test deletion against the retention policy.

Before production, rehearse an adapter outage, an unknown status value, a worker restart, and a destination that never reaches a terminal state. Confirm that no rehearsal produces a second send by accident. Then inspect an exported record as if the provider account were unavailable: it should still identify the notice revision, the submission reference, every status observation, and the rule that closed the attempt.

That is the decision rule. Choose the contract that produces this record with the least custom machinery while meeting the required regional sender rules. Everything else is secondary.

## References

The standards below define useful boundaries: authentication guidance for PSTN-based out-of-band mechanisms, and email-domain authentication that must not be mistaken for SMS evidence.

## Sources

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
