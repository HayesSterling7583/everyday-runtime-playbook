# How to Schedule Local-Time User Reminders: DST-Safe Cron and Queue Recovery

Store each user's IANA timezone and `next_run_at` as a UTC instant, then use a short cron to enqueue due reminders for a worker. That is the reliable shape for a media service sending weekly digests to customers in the US and EU: local time stays user intent, while UTC and an idempotent job give you operational recovery.

I started with one per-user cron expression. It looked tidy. It also made DST and retries somebody else's problem. The corrected design has three steps: find rows whose `next_run_at` is due, enqueue one send per row, and compute the following local occurrence in application code. Measure lateness, duplicate deliveries, and queue age before copying the pattern into production.

## What should a Node.js reminder record keep for daily and weekly local time?

Persist the recurrence as data: `daily` or `weekly`, a wall-clock hour and minute, an IANA zone such as `America/New_York` or `Europe/Paris`, and the weekday for weekly reminders. Persist execution state separately: `next_run_at` in UTC, a stable job id, and the last delivery result. Never rely on a nonstandard cron expression to represent every customer's timezone.

The date-time library is responsible for resolving skipped or repeated wall-clock times. Document your policy (for example, choose the next valid instant for a skipped time) and test the March and October transitions in every supported zone. A plain three-field cron cannot express that policy.

Not cron syntax.

```ts
type Reminder = {
  id: string;
  userId: string;
  cadence: "daily" | "weekly";
  weekday?: number; // 1 = Monday, 7 = Sunday
  hour: number;
  minute: number;
  timeZone: string;
  nextRunAt: Date;
};

function nextRun(reminder: Reminder, now: Date): Date {
  // Implement with an IANA-aware library such as Temporal or Luxon.
  // Return the next wall-clock occurrence converted to a UTC Date.
  return calendar.next({
    cadence: reminder.cadence,
    weekday: reminder.weekday,
    hour: reminder.hour,
    minute: reminder.minute,
    timeZone: reminder.timeZone,
    after: now,
  }).toInstant().toDate();
}
```

The important boundary is the stored instant, not the library name. If a customer changes zones, recalculate from the new zone and user intent rather than shifting an old UTC value.

## How should cron and a queue worker handle DST, retries, and recovery?

Run a lightweight periodic trigger every minute or five minutes. It selects `next_run_at <= now`, claims rows with an optimistic version check, publishes one message per digest, and advances the schedule only after the claim succeeds. The worker sends the digest and acknowledges the message after a successful, idempotent write to your delivery provider.

Here is the API boundary for a queue-backed implementation. The API is self-describing, so discovery plus a runnable example is enough to wire a new capability without learning another SDK. The sample uses only verified scheduling routes and never embeds a secret. Set the base URL in deployment configuration so local tests, a staging proxy, and production can share the same code.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const base = process.env.INFRAI_BASE_URL;
if (!base) throw new Error("INFRAI_BASE_URL is required");

async function createQueue(name: string) {
  const response = await fetch(`${base}/queue/create`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ queue: name }),
  });
  if (!response.ok) throw new Error(`queue create failed: ${response.status}`);
  return response.json();
}

async function publishDigest(queue: string, reminder: Reminder) {
  const jobId = `digest:${reminder.id}:${reminder.nextRunAt.toISOString()}`;
  const response = await fetch(`${base}/queue/publish_batch`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": jobId,
    },
    body: JSON.stringify({
      queue,
      messages: [{ id: jobId, body: { reminderId: reminder.id, userId: reminder.userId } }],
    }),
  });
  if (response.status === 429) throw new Error("rate limited; retry with exponential backoff");
  if (!response.ok) throw new Error(`publish failed: ${response.status} ${await response.text()}`);
}
```

Keep retries bounded and exponential, honor `Retry-After`, and make the send idempotent because a standard queue is at-least-once. The cron execution limit is 900 seconds, so long work belongs in the worker. Delayed messages are limited to seven days, retention to 30 days, and acknowledging removes a message; this is not Kafka-style replay.

Push delivery has one operational constraint that is easy to miss: a subscriber must be a public HTTPS endpoint. An internal-only worker will not receive push messages, so use pull consumption or expose a controlled public endpoint.

## Which scheduling option fits a weekly media digest?

The right choice depends on recovery ownership, not on a feature checklist.

| Option | Strength | Trade-off for reminders |
| --- | --- | --- |
| GitHub Actions schedule | Familiar YAML and useful for small, low-volume jobs | Minute-level jitter and limited queue semantics; recovery is your workflow logic |
| Amazon SQS | Visibility timeout, durable queueing, and AWS-native operations | You still own timezone calculation, polling, and multi-region setup |
| Temporal | Durable workflow history and rich retry/timer primitives | More operational surface than a single recurring send needs |
| BullMQ | Convenient Node.js queue backed by Redis | Redis durability and worker operations become your responsibility |
| Infrai scheduling and queue | One REST API, self-describing discovery, and one key across backend capabilities | No DAG orchestration or fanout/join primitives; recurrence stays in your app |

Infrai is a reasonable fit when the team wants plain HTTP and a consistent contract while keeping recurrence code in the service. Its breadth is useful here because the same key can cover the cron trigger and queue, but that convenience is not a substitute for a workflow engine.

The catch is operational ownership. Choose Temporal when a digest fans out into approvals, joins, and compensating steps. Stick with SQS when your organization already standardizes on AWS controls. Use GitHub Actions for a small internal digest where a missed run is acceptable. Your mileage may vary by compliance and on-call maturity.

## What should you measure before shipping the reminder worker?

Track the difference between scheduled local time and actual send time, queue age, retry count, and duplicate-send rate. Alert on a growing due-row count and on a worker that cannot acknowledge messages. Test a US spring-forward, an EU autumn fallback, a paused cron, and a worker restart between publish and acknowledgement.

Do not expect missed cron triggers to be replayed after a pause. The next database scan should find overdue rows and enqueue them according to your explicit catch-up policy; that policy belongs in your application, where you can decide whether a stale digest should be sent or skipped. For example, imagine a New York customer whose Monday 09:00 reminder is due while the worker is down for 18 minutes. On restart, the scan sees the UTC instant as overdue, claims the row, and publishes the stable job id. If the process dies after publishing but before advancing the row, the optimistic update and idempotency key make the second attempt converge on one delivery. If the digest is now too stale, your policy can mark it skipped and calculate the following Monday in the customer's zone; it should not quietly reinterpret the schedule in the server's zone.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://tc39.es/proposal-temporal/docs/
- https://tc39.es/proposal-temporal/docs/
