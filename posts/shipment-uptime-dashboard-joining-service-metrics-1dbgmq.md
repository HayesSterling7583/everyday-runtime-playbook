# Shipment Uptime Dashboard: Joining Service Metrics and Logs for Incident Evidence

An internal uptime page should optimize for incident reconstruction, not for a wall of green boxes. The useful design is a small read model that joins a bounded service-health sample with structured operational events by time, service, and correlation ID. **Short answer:** keep the dashboard cheap and boring, but preserve enough context to answer which shipment was affected, what the service reported, and what happened immediately before and after the failure.

For a logistics operation, "dispatch-api is down" is only the start. An operator needs to know whether a carrier-label request failed, whether a retry created a second booking attempt, and which shipment IDs need review. Averages cannot answer those questions. A compact event trail can.

## What should an internal uptime dashboard preserve from metrics and logs?

Start from the questions an on-call engineer or support operator will ask. Which service changed state? When was the last successful check? Which shipment operation was in flight? Can its related log entries be found without searching every byte retained by the company?

I would store two deliberately different records. A health sample is small and numeric: timestamp, service, availability, latency, and recent error count. An operational event is sparse but descriptive: timestamp, service, event name, shipment ID, correlation ID, and outcome. The page joins them for reading; it does not pretend one record can do both jobs.

This follows the monitoring distinction described in Google's SRE guidance. Metrics are suited to trends and alerting, while logs retain details about individual events. The same guidance organizes monitoring around latency, traffic, errors, and saturation. A tiny admin page does not need to reproduce an entire monitoring platform, but its read model should leave room for those signals rather than reducing health to a process-is-running boolean.

Here is the decision rule I use for the page:

| Visible state | Evidence behind it | Operator action |
| --- | --- | --- |
| Healthy | Recent successful probe and no recent failed shipment event | Keep watching |
| Degraded | Probe succeeds, but latency or recent errors cross the team's configured boundary | Inspect correlated events |
| Unavailable | Most recent probe fails | Check affected shipments and dependency state |
| Unknown | No sample inside the configured freshness window | Treat telemetry as missing, not as success |

"Unknown" matters.

Stale green is worse than honest uncertainty because it turns a telemetry gap into a false operational claim.

## Build the read model first

The data flow is plain: each service emits structured shipment events and a collector records periodic health samples; a small Node.js process keeps a bounded window, calculates current state, and serves an internal JSON view. The browser can poll that view and render rows. Storage, authentication, and the probe scheduler sit outside this example because they depend on the deployment boundary, but the status calculation and correlation contract do not.

The following TypeScript file runs on current Node.js releases with TypeScript execution configured. It uses only built-in server primitives, seeds illustrative logistics data, and exposes one internal endpoint. The in-memory arrays make the example copyable; production retention belongs in durable storage. This is a conscious trade-off: a memory-backed example reveals the state calculation without hiding it behind infrastructure, but it cannot preserve incident evidence across a restart or coordinate several application instances. For those requirements, keep the same record shapes and put the bounded window in durable shared storage instead.

```ts
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";

type HealthSample = {
  at: string;
  service: string;
  ok: boolean;
  latencyMs: number;
  recentErrors: number;
};

type ShipmentEvent = {
  at: string;
  service: string;
  event: "label.requested" | "label.created" | "dispatch.failed";
  shipmentId: string;
  correlationId: string;
  outcome: "ok" | "error";
};

type Status = "healthy" | "degraded" | "unavailable" | "unknown";

const freshnessMs = 90_000;
const degradedLatencyMs = 800;

const health: HealthSample[] = [
  {
    at: new Date(Date.now() - 18_000).toISOString(),
    service: "dispatch-api",
    ok: true,
    latencyMs: 940,
    recentErrors: 2,
  },
  {
    at: new Date(Date.now() - 12_000).toISOString(),
    service: "label-worker",
    ok: false,
    latencyMs: 1_420,
    recentErrors: 4,
  },
];

const events: ShipmentEvent[] = [
  {
    at: new Date(Date.now() - 15_000).toISOString(),
    service: "dispatch-api",
    event: "label.requested",
    shipmentId: "shp_demo_1042",
    correlationId: "corr_demo_a7f3",
    outcome: "ok",
  },
  {
    at: new Date(Date.now() - 11_000).toISOString(),
    service: "label-worker",
    event: "dispatch.failed",
    shipmentId: "shp_demo_1042",
    correlationId: "corr_demo_a7f3",
    outcome: "error",
  },
];

function calculateStatus(sample: HealthSample | undefined, now: number): Status {
  if (!sample || now - Date.parse(sample.at) > freshnessMs) return "unknown";
  if (!sample.ok) return "unavailable";
  if (sample.latencyMs > degradedLatencyMs || sample.recentErrors > 0) {
    return "degraded";
  }
  return "healthy";
}

function buildDashboard(now = Date.now()) {
  const latestByService = new Map<string, HealthSample>();

  for (const sample of health) {
    const previous = latestByService.get(sample.service);
    if (!previous || Date.parse(sample.at) > Date.parse(previous.at)) {
      latestByService.set(sample.service, sample);
    }
  }

  return [...latestByService.values()].map((sample) => ({
    service: sample.service,
    status: calculateStatus(sample, now),
    checkedAt: sample.at,
    latencyMs: sample.latencyMs,
    recentErrors: sample.recentErrors,
    recentEvents: events
      .filter((event) => event.service === sample.service)
      .sort((a, b) => Date.parse(b.at) - Date.parse(a.at))
      .slice(0, 5),
  }));
}

function sendJson(response: ServerResponse, status: number, value: unknown): void {
  response.writeHead(status, {
    "content-type": "application/json; charset=utf-8",
    "cache-control": "no-store",
  });
  response.end(JSON.stringify(value));
}

createServer((request: IncomingMessage, response: ServerResponse) => {
  if (request.method === "GET" && request.url === "/internal/status") {
    sendJson(response, 200, { generatedAt: new Date().toISOString(), services: buildDashboard() });
    return;
  }

  sendJson(response, 404, { error: "not_found" });
}).listen(3000);
```

Run it behind the same authentication boundary as the rest of the internal admin surface. The important part is not the HTTP route. It is the shape of `buildDashboard()`: freshness is explicit, state is derived in one place, and the raw evidence remains available beside the label.

One trap is visible in the sample. A successful probe can still produce a degraded state because real shipment work is slow or failing. If the page only checks whether `/health` returns success, it can stay green while customers cannot create labels.

## Correlation beats a larger log pile

The `correlationId` is the cheapest useful join key in this design. Pass it through a shipment operation, attach it to each structured event, and show it on the incident row. An operator can then follow one attempt across the dispatch API and label worker without guessing from message text.

Do not use the shipment ID as the only join key. One shipment may have several attempts, and those attempts can have different outcomes. Keep both identifiers: the shipment ID represents the business object; the correlation ID represents one execution path. This is a data-model choice, not a dashboard decoration.

Be strict about payloads too. Status evidence should include identifiers and outcomes, not addresses, customer notes, carrier credentials, or raw model prompts. The dashboard needs enough evidence to reconstruct service behavior, and no more. Retention should follow the incident-review window defined by the team, while access should follow the internal admin boundary.

There is a direct cost reason for being selective. Log services commonly charge for ingestion volume; Amazon's public CloudWatch pricing page, for example, describes log ingestion charges by data volume. That does not make logs undesirable. It means high-cardinality evidence should be intentional: structured fields that close an investigation are valuable, while repeated blobs and duplicate stack traces are expensive noise.

Keep the raw stream bounded.

Keep the useful keys.

Tests should freeze time and exercise the state function directly. Cover a fresh success, a slow success, an explicit failure, and a missing or stale sample. Then test correlation separately: two events with the same shipment ID but different correlation IDs must remain distinguishable.

```ts
import assert from "node:assert/strict";
import { test } from "node:test";

type HealthSample = {
  at: string;
  service: string;
  ok: boolean;
  latencyMs: number;
  recentErrors: number;
};

type Status = "healthy" | "degraded" | "unavailable" | "unknown";

const freshnessMs = 90_000;
const degradedLatencyMs = 800;

function calculateStatus(sample: HealthSample | undefined, now: number): Status {
  if (!sample || now - Date.parse(sample.at) > freshnessMs) return "unknown";
  if (!sample.ok) return "unavailable";
  if (sample.latencyMs > degradedLatencyMs || sample.recentErrors > 0) return "degraded";
  return "healthy";
}

test("marks stale evidence unknown", () => {
  const now = Date.parse("2026-10-05T12:00:00.000Z");
  const sample: HealthSample = {
    at: "2026-10-05T11:58:00.000Z",
    service: "dispatch-api",
    ok: true,
    latencyMs: 120,
    recentErrors: 0,
  };

  assert.equal(calculateStatus(sample, now), "unknown");
});

test("marks a successful but slow service degraded", () => {
  const now = Date.parse("2026-10-05T12:00:00.000Z");
  const sample: HealthSample = {
    at: "2026-10-05T11:59:45.000Z",
    service: "label-worker",
    ok: true,
    latencyMs: 901,
    recentErrors: 0,
  };

  assert.equal(calculateStatus(sample, now), "degraded");
});
```

Those numbers are example policy, not universal thresholds. Pick them from the service's own objective and normal traffic, then keep them in configuration so a deployment does not silently change the meaning of green. The test's fixed 2026 timestamp is there for determinism, not as an assertion about current production behavior.

Deployment deserves one awkward test as well: stop the collector while the monitored service remains healthy. The row must become unknown after the freshness window. That single case catches the most misleading failure mode in a homegrown status page. The limitation is deliberate: this page reconstructs recent application incidents, but it is not suitable as the sole external availability monitor because it shares an operational boundary with the system it describes. Use an independently hosted probe when the question is whether users can reach the service from outside that boundary; keep this internal view for the richer shipment evidence.

## Operate the page as evidence infrastructure

Before release, walk through one synthetic shipment attempt from event creation to the admin view. Confirm that timestamps use one unambiguous format, identifiers survive each service boundary, failed and successful outcomes remain distinct, and a stale sample cannot display as healthy. Check that the page is protected by internal authentication and that sensitive shipment fields never enter its event payload.

After release, review the event schema when an incident cannot be reconstructed. Add a field only when it answers a concrete question, and remove fields that no investigation uses. Watch ingestion volume alongside request traffic so a retry storm does not multiply duplicate logs without bound. This is the ship-first compromise: begin with the smallest evidence contract that closes a real investigation, then evolve it from incident reviews rather than from dashboard aesthetics.

The final check is human. Give an operator a shipment ID and ask them to explain the sequence without shell access. If they can identify the affected service, the execution attempt, the last trustworthy health sample, and the failure outcome, the page is doing its job. If they can only report a color, it is an uptime ornament.

## Sources / References

- Google SRE Book, "Monitoring Distributed Systems": https://sre.google/sre-book/monitoring-distributed-systems/
- Amazon CloudWatch pricing: https://aws.amazon.com/cloudwatch/pricing/
