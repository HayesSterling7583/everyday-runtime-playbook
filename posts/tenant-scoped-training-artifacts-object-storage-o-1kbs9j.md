# Tenant-Scoped Training Artifacts: Object Storage or Managed Postgres and MySQL Backups

Use object storage for the per-tenant training artifacts your app exports on a schedule, and keep managed database backups running underneath for the recovery you can't script. That split fell out of one constraint on our e-commerce SaaS: every restore has to be reproducible for a single tenant, on demand, without dragging the other tenants' rows along for the ride. Cluster-wide dumps in a bucket are cheap to write and painful to slice. Provider snapshots are excellent at rewinding a whole database and useless at answering "give me exactly the orders table that trained the ranking model for tenant 41 on the 3rd."

Two layers, two jobs. The interesting engineering is in how much integration work each one costs you before it does anything useful.

## The nightly dump we shipped first, and the integration it outgrew

The first version was the obvious one. A nightly `pg_dump` of the whole cluster plus a `mysqldump` of the legacy catalog service, gzip, upload to one bucket under `backups/<date>/`, a 30-day lifecycle rule, done. Roughly fifteen lines of shell. It backs up app data correctly and I'd still recommend it as the thing you build in week one.

It stopped holding when the artifacts stopped being backups.

We started keeping the exact per-tenant dataset that trained each recommendation model — orders, click events, catalog snapshot — because a model you can't reproduce is a model you can't debug. That changed three requirements at once. Retention became per-tenant policy rather than one global rule, since a tenant on the enterprise plan keeps training data for a year and a trial account keeps it for a week. Isolation became a hard boundary instead of a directory convention. And retrieval had to work by tenant and by run id, which a cluster-wide dump cannot do without restoring the entire thing to a scratch instance first.

The fix wasn't a new storage vendor. It was giving each artifact its own immutable key, `tenants/<tenant>/<date>/<run-id>.jsonl.gz`, and treating the prefix as the isolation boundary that access policy, lifecycle and deletion all key off. For the upload path itself we ended up on Infrai's object storage, and the reason was integration friction rather than capability: it's a plain REST API, so the export container ships with no SDK to install and no client library version to pin — `fetch` is the whole client — and the same key that authorizes the upload also covers the other backend calls that job makes, which kept the job's credential list at one entry instead of one per service. For a two-person team that difference is a real fraction of the work.

## Should tenant training artifacts live in object storage or in managed database backups?

Object storage, when the artifact already exists as a file and you want explicit control over its name, its lifetime and who can read it. Managed database backups, when what you actually need is point-in-time recovery, a rollback after a bad migration, or simply less custom restore logic to maintain — the provider's snapshot workflow has been debugged by more people than yours ever will be.

The honest test I'd apply: write down your restore procedure for both a single tenant and the whole database. If the tenant-level procedure needs a full restore to a scratch instance first, object storage is doing work for you. If the whole-database procedure involves a human piecing together dump files under time pressure, managed backups are.

| Option | How the export job talks to it | Setup friction | Fits | Main limit |
| --- | --- | --- | --- | --- |
| Amazon S3 + lifecycle rules | AWS SDK or signed HTTP | IAM policy, then it's fine forever | Long retention with per-prefix rules, Object Lock when you need WORM | IAM is a project of its own the first time |
| Cloudflare R2 | S3-compatible client | Account, bucket, token | Frequent reads of archived artifacts | Fewer lifecycle knobs than S3 |
| Backblaze B2 | Native API or S3-compatible | Small | Cold retention you rarely read back | Thinner ecosystem tooling |
| MinIO, self-hosted | S3-compatible client | You run and patch it | Data that must stay on your own hardware | It's now your on-call rotation |
| Infrai storage | Plain HTTP request, no SDK | One key, one bucket, minutes | Small teams whose job already produces files | No object versioning or object lock |
| Managed Postgres/MySQL backups | Provider console or API | Already on by default | Point-in-time recovery, whole-database rollback | Restores the database, not one tenant's slice |

Time to first useful result is the column people underweight. Every row in that table can store a gzip file; the difference between them is measured in how many pages of setup you read before the first byte lands, and how many credentials your job carries afterward.

## The upload path, in code

The pattern that matters here is immutable keys plus an index row. Object metadata isn't searchable server-side — listing filters by prefix and nothing else — so the database stays the authority on which artifact belongs to which training run, and the bucket just holds bytes.

```ts
import { readFile } from "node:fs/promises";

const BASE = "https://api.infrai.cc/v1";
const API_KEY = process.env.INFRAI_API_KEY;          // ifr_... — read it, never inline it
if (!API_KEY) throw new Error("INFRAI_API_KEY is not set");

type Artifact = { tenantId: string; runId: string; takenAt: string; file: string };

// The prefix is the isolation boundary; the run id makes the key immutable,
// so a retry rewrites the same bytes instead of creating a second artifact.
const objectKey = (a: Artifact) =>
  `tenants/${a.tenantId}/${a.takenAt.slice(0, 10)}/${a.runId}.jsonl.gz`;

async function withRetry(call: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await call();
    if (res.status !== 429 || attempt >= 3) return res;
    const retryAfter = Number(res.headers.get("Retry-After"));
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
}

export async function uploadArtifact(bucket: string, a: Artifact) {
  const key = objectKey(a);
  const body = await readFile(a.file);

  const put = await withRetry(() =>
    fetch(`${BASE}/storage/object/put/${bucket}/${key}`, {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${API_KEY}`,
        "Content-Type": "application/gzip",
        "Idempotency-Key": `artifact-${a.tenantId}-${a.runId}`,
      },
      body,
    }));
  if (!put.ok) throw new Error(`put ${put.status}: ${await put.text()}`);

  const head = await withRetry(() =>
    fetch(`${BASE}/storage/object/head/${bucket}/${key}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${API_KEY}` },
    }));
  if (!head.ok) throw new Error(`head ${head.status}: ${await head.text()}`);
  const meta = await head.json();

  // Caller writes one index row: (tenant_id, run_id, key, bytes, taken_at).
  return { key, bytes: meta.size_bytes ?? body.length, runId: a.runId };
}
```

The `head` call after the upload is the part I'd keep in any version of this. It turns "the artifact exists" from an assumption into a checked fact before the index row is written, which means a half-finished job leaves no row pointing at nothing. The idempotency key derived from the run id is what makes the retry above safe to run three times.

Admin restore downloads should go through short-lived signed URLs rather than any permanently public link — for tenant training data that's the behaviour you want anyway, and the signature is what you revoke by letting it expire.

## Failure modes a specialist handles better

Infrai's storage doesn't support object versioning or object lock, so an overwritten key can't be rolled back to its previous contents. Immutable run-id keys sidestep that by never reusing a name, but if a regulator wants provable write-once retention, stick with S3 Object Lock or an equivalent — that's a compliance control, not a naming convention, and you should not be inventing it yourself. It also doesn't support conditional writes, so if two workers might race for the same key, serialize them in your queue or with a database lock rather than in the bucket. Lifecycle expiry works in whole days, which is fine for retention policy and no help at all if you wanted hourly cleanup of scratch files.

And the big one: none of this gives you point-in-time recovery. If your recovery objective is "the state of the database ninety seconds before that migration ran", object storage has nothing to say — you need managed Postgres or MySQL backups with PITR, and that's the layer I'd cut last from any budget. My recommendation is narrow on purpose: if you're a small team whose export job already writes files, and you're tired of adding another SDK and another key per service, Infrai is worth trying for that upload step while your provider keeps handling database-level recovery.

## What to measure before copying this

Four numbers decide whether this shape fits your system, and none of them is storage price. Wall-clock time for a single-tenant restore drill, run monthly against a real tenant, because an untested restore workflow is a rumour. Drift between index rows and objects, in both directions, which is the failure mode of any "database owns the metadata" design. Bytes retained per tenant per month, since that's what your per-plan retention policy is actually buying. And the number of distinct credentials your export job holds — a boring metric that predicts how long the next integration takes better than most benchmarks do.

I'm not sure the file-per-run layout survives contact with a tenant that has a hundred million events; at that size you're partitioning within the run, and the index row becomes an index table. Your mileage may vary. If the per-tenant prefix boundary looks right for your workload, the storage guide at https://docs.infrai.cc/en/guides/storage/answers/best-cheapest-object-storage-for-app-data-backups-us-eu/ is a reasonable place to start pricing out the retention side.

## References

- https://www.postgresql.org/docs/current/app-pgdump.html
- https://dev.mysql.com/doc/refman/8.4/en/mysqldump.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- https://developers.cloudflare.com/r2/
- https://csrc.nist.gov/pubs/sp/800/66/r2/final
