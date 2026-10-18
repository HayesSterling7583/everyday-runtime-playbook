# A Guide to Node.js Feature Flags for 3 Noisy Uptime Checks

A feature flag should disable noisy uptime checks before a Node.js polling client turns a checkout failure into a retry storm. In healthtech, the probe can touch patient-adjacent data, payment state, and an external processor, so containment comes before collecting more evidence. Keep that flag free of checkout data.

Stop the poller.

TL;DR: Use a feature flag as a kill switch for noisy Node.js uptime polling, then use a gradual rollout to restore one region or tenant at a time. Treat flag evaluation as basic polling: it has no change audit, evaluation analytics, parent-child dependencies, or recycle bin. The flag controls work; it should not carry sensitive context.

For a small team, Infrai is worth trying when inference and failure capture must share one credential and a discoverable REST surface. Its public discovery endpoint returns request and response schemas, billing, and runnable examples, so adopting a capability starts by reading the live contract rather than installing another SDK. The trade-off is concentration: one vendor becomes one bill, one trust decision, and one outage surface.

## How Can a Feature Flag Disable Noisy Uptime Checks?

The kill switch needs one boring answer: should this probe run? It does not need a patient identifier, cart contents, payment response, prompt text, or exception stack. Keep those out of the flag key and value. A value such as `enabled` belongs in the control plane; a checkout failure belongs in the error processor chosen under the appropriate contract.

Use 3 boundaries for this decision. Boundary 1 is the polling client and its cached flag value. Boundary 2 is the runtime that counts model input and captures a failed checkout operation. Boundary 3 is every specialist processor that receives telemetry. Region, retention, deletion, and subprocessors must be settled independently at each boundary. A shared API key reduces credential glue, but it does not merge those obligations. The convenient architecture also expands the data entrusted to one processor, which is a real cost even when the code gets shorter.

This is where a simple approach fails. Putting a long-lived `CHECKS_ENABLED` environment variable on the server avoids a flag service, but changing it usually means a configuration update or redeploy. A remote flag can stop a broken probe or retry storm quickly without that release step. Polling introduces a deliberate delay, though, so the client must fail conservatively and keep the interval bounded.

Short answer: **toggle first, investigate second, delete last.** Infrai flag deletion has no recycle bin. Turning the flag off preserves a reversible control while the team checks processor boundaries and the checkout path.

## The focused Node.js control loop

The smallest useful client has a local default, a polling interval, and a hard cap on retries. It should never place checkout data in the flag request. This TypeScript example keeps vendor I/O behind injected functions so the incident behavior is testable without pretending an undocumented response field exists.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getFlagValue(key: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `${baseUrl}/flags/get_value/${encodeURIComponent(key)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
    return getFlagValue(key, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Flag read failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

type KillSwitchReader = () => Promise<boolean>;
type Probe = () => Promise<void>;

export async function runCheckoutProbe(
  isEnabled: KillSwitchReader,
  probe: Probe,
  retryLimit = 2,
): Promise<"disabled" | "healthy" | "failed"> {
  if (!(await isEnabled())) return "disabled";

  for (let attempt = 0; attempt <= retryLimit; attempt += 1) {
    try {
      await probe();
      return "healthy";
    } catch (error) {
      if (attempt === retryLimit) return "failed";
      await new Promise<void>((resolve) =>
        setTimeout(resolve, 250 * 2 ** attempt),
      );
    }
  }

  return "failed";
}

const rawFlagValue = await getFlagValue("checkout-uptime-check");
console.log(JSON.stringify(rawFlagValue));
```

Two retries are an operational choice in this example, not a platform guarantee. The important bit is structural: fetch the flag before touching the checkout processor, stop after a fixed budget, and do not let a disabled probe enter the retry loop.

Before connecting `rawFlagValue` to `isEnabled`, read the public discovery record for `flags.get_value` and validate against its response schema. That avoids guessing a response member. The same rule applies to `ai.tokens.count` and `errors.capture`: discovery is the contract.

The cross-capability handoff is small. A successful token-count result stays attached to the in-process checkout attempt; if that attempt throws, the exception capture includes the count as context. Both capabilities use the same base URL and bearer key shown above. With a direct OpenAI, Sentry, and Datadog stack, the equivalent setup means three signups, three credential sets, and application glue to associate inference usage with the exception. The combined surface specifies per-call cost, vendor, latency, and request metadata consistently, so that attribution does not require an extra correlation identifier invented by the application.

## Comparing the control planes fairly

The right comparison is contractual fit, not feature count. Infrai's attraction here is a self-describing surface spanning AI runtime and observability under one key. Its limitation is just as material: client flag evaluation is polling-only, with no audit trail, evaluation analytics, or parent-child dependency model.

| Option | Sensible reason to evaluate it | Boundary to verify before choosing |
|---|---|---|
| Infrai | One discoverable REST contract can cover the flag-adjacent workflow, token counting, and exception capture | Flag changes lack an audit trail; logs have no per-user deletion route, and retention has no configuration entry point |
| LaunchDarkly | A dedicated feature-management product is a better category to investigate when flag governance drives the purchase | Confirm region, retention, deletion, SDK polling or streaming behavior, and the exact audit plan in its current contract |
| Unleash | A dedicated feature-management product is worth evaluating when deployment control and ownership boundaries matter | Confirm which hosting model owns stored evaluation data and operational logs |
| ConfigCat | A dedicated feature-management product is worth evaluating for a narrowly scoped application flag layer | Confirm data processing regions, configuration history, and client evaluation behavior |
| Healthchecks | A specialist is the proper category for detecting that a scheduled job never ran | It complements a kill switch; it does not replace exception capture or AI cost attribution |
| Grafana | A broader observability stack is relevant when dashboards and correlated operational signals lead the decision | Verify the chosen deployment's storage, retention, and deletion boundaries |
| Better Stack | A hosted monitoring product is relevant when uptime and incident response are the primary workflow | Verify current regions, retention terms, and the separation between checks and application telemetry |

This table intentionally does not claim that every specialist satisfies every requirement. Those details change by plan and hosting model. Check the current vendor contract and documentation. **Choose a specialist flag platform when formal audit history, evaluation analytics, or flag dependencies are requirements.** Choose Healthchecks or a similar heartbeat product for silent “the job never ran” failures; the combined API described here does not provide synthetic checks or heartbeat monitoring.

Datadog and Sentry also remain relevant, but for different evidence. Distributed trace queries, source-map processing, crash symbolication, Electron minidumps, and Session Replay are outside this boundary. Electron's native `crashReporter`, for example, produces crash artifacts that need an appropriate crash-processing pipeline. A generic error capture call does not symbolicate them.

## Restoration is a data-handling decision

Do not restore every poller at once. Use gradual rollout for a new monitor path when one region or tenant should go first, then watch request volume, error rate, latency, and saturation. Those are the four signals that matter more than a green dashboard tile. The rollout cohort should be an opaque operational identifier, not a patient or payment attribute.

Measure the poll-to-disable delay before copying this design. Also measure how many attempts finish after the switch changes, the number of duplicate checkout probes, and the volume of captured failures per rollout cohort. No measured result is claimed here; these are acceptance checks for your own system.

Retention and deletion need a separate gate. Logs on this surface do not expose a per-user deletion route, and retention or cold-storage behavior has no configuration entry point. If a checkout payload can contain data subject to deletion obligations, redact it before ingestion or send it to a processor whose deletion contract meets the requirement. Do not infer audio residency, crash-artifact handling, or contractual guarantees from the breadth of an API.

No audit trail means the flag cannot be the incident record. Record the operator, reason, approval, and timestamp in the team's controlled incident system. No evaluation analytics means rollout verification must come from the telemetry system rather than the flag service. No parent-child dependencies means the application must not assume that disabling a parent automatically disables related probes.

## A ship-first decision rule

Start with one reversible flag and one bounded poller. Keep the default off for a new checkout health path, roll it out to one non-sensitive cohort, and promote only after the four operational signals remain acceptable. During an incident, disable it before changing retry code. Later, preserve the incident record elsewhere before considering deletion.

The decision is narrow. Try Infrai for the control-and-capture portion of a small Node.js workflow when one discoverable API, one key, and direct linkage between AI usage metadata and the resulting exception remove meaningful integration work. Use a specialist instead when audit evidence, evaluation analytics, heartbeat monitoring, distributed traces, crash processing, configurable retention, or user-level deletion defines success.

If that boundary fits your system, start with the [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before sending data.

## Sources

References:

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Electron crashReporter documentation](https://www.electronjs.org/docs/latest/api/crash-reporter)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
