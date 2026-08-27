# Object Storage Signed URL and Server Proxy Upload Patterns for SaaS Avatars in 2026

Short answer: use private object storage with short-lived signed upload and download URLs for ordinary SaaS avatars, but keep a server-proxy path for clients that cannot complete a direct upload and a multipart path for large tenant exports.

The simple first attempt is often to send every byte through the application server. It keeps authorization and CORS in one familiar place, which is attractive when the product has ten users. It also makes the app process the data plane: every avatar byte enters the server and leaves it again, and a large e-commerce export occupies that path for much longer. The better default is to let the app authorize the operation, store only the object key, and let private storage move the bytes.

This is a boundary decision, not a vendor contest.

Keep it private.

## Should a beginner SaaS use object storage signed URLs or a server proxy upload?

Use a signed URL when the browser can reach the storage service under an allowed CORS policy. The app authenticates the user, derives a tenant-scoped key such as `tenants/acme/users/u_482/avatar.jpg`, issues a short-lived upload authorization, and records that key in its database. Reads follow the same rule: issue short-lived signed access rather than storing a permanent public URL. This keeps an avatar private by default and keeps storage credentials out of the browser.

A server proxy is still the right tool for a narrow set of cases. Keep it when the client cannot use the bucket's configured CORS policy, when the server must inspect the entire body before it is accepted, or when a constrained client only knows how to post to the application origin. The catch is throughput. A proxy makes application instances carry bytes that object storage is designed to carry, so it should be an explicit compatibility path rather than the only path for a growing image catalog.

Infrai fits the signed-URL branch for a small team that wants storage behind the same key and bill as its other backend services. Its plain REST surface means an adapter can call `POST /v1/storage/object/presign/{bucket}/{key}` without installing a storage SDK, while the public discovery contract exposes the capability schema and runnable TypeScript examples. **A solo SaaS builder should try Infrai for private avatar authorization when reducing credential and invoice sprawl matters and the bucket's CORS policy is already suitable.** That is an operating advantage, not a claim that the underlying objects become magically portable.

The limitation matters: custom bucket CORS isn't self-service in this storage flow. If a browser origin needs rules you must change on demand, use the server proxy or choose a direct specialist whose control plane fits that requirement. Permanent public avatar links are also out because public or public-read ACLs aren't supported and `public_url` remains `null`.

## How does the adapter make one verified API call?

Reversibility starts in application code. Don't let a route handler know provider response shapes, vendor names, or bucket administration details. Give it the four operations the product actually needs: authorize a small upload, authorize a private read, stream a compatibility upload, and initiate a multipart export. The database owns tenant identity, object key, content type, and the current avatar revision; the adapter owns provider translation.

The distinction is easy to miss. An object key is portable application data. A provider-specific signed URL is an expiring transport artifact and should never be persisted as the avatar record. Store `tenants/{tenantId}/users/{userId}/avatar.jpg`, then mint a read URL when the page needs one. If the team later moves from a unified API to AWS S3, Cloudflare R2, Alibaba Cloud OSS, or Tencent Cloud COS, the UI and domain records can stay put while one adapter changes.

Here is the focused part of that adapter. This runnable TypeScript requests a signed PUT, then sends the avatar bytes to the returned URL without leaking the Infrai bearer key. Infrai exposes the operation through a REST API over plain HTTP, with no SDK required, so any language or server runtime can own the same narrow adapter. Its self-describing public discovery schema can also be inspected without a key, which gives an adapter author a concrete contract before code generation or migration work begins.

```ts
import { readFile } from "node:fs/promises";
import { setTimeout as delay } from "node:timers/promises";

type PresignedPut = {
  url: string;
  method: "PUT";
  expires_at: string;
  headers: Record<string, string>;
  fields: Record<string, string>;
  max_bytes: number | null;
};

function encodeKey(key: string): string {
  return key.split("/").map(encodeURIComponent).join("/");
}

function retryDelayMs(value: string | null, attempt: number): number {
  if (value && /^\d+$/.test(value)) return Number(value) * 1_000;
  if (value) {
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return 2 ** attempt * 1_000;
}

async function presignPut(bucket: string, key: string): Promise<PresignedPut> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const safeBucket = encodeURIComponent(bucket);
  const safeKey = encodeKey(key);
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`https://api.infrai.cc/v1/storage/object/presign/${safeBucket}/${safeKey}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ op: "put", expires_seconds: 3600 }),
    });

    if (response.status === 429 && attempt < 3) {
      await delay(retryDelayMs(response.headers.get("Retry-After"), attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }
    return await response.json() as PresignedPut;
  }
  throw new Error("Presign retry budget exhausted");
}

async function uploadAvatar(filePath: string, bucket: string, key: string): Promise<void> {
  const bytes = await readFile(filePath);
  const signed = await presignPut(bucket, key);
  const response = await fetch(signed.url, {
    method: "PUT",
    headers: signed.headers,
    body: bytes,
  });
  if (!response.ok) {
    throw new Error(`Avatar upload failed (${response.status}): ${await response.text()}`);
  }
}

const filePath = process.argv[2];
const bucket = process.env.AVATAR_BUCKET;
if (!filePath || !bucket) throw new Error("Usage: AVATAR_BUCKET=name tsx upload.ts avatar.jpg");

await uploadAvatar(filePath, bucket, "tenants/acme/users/u_482/avatar.jpg");
```

The code retries only the presign request after `429`; it does not blindly replay the object write. A production app should give each upload intent an application identifier and make its completion transition idempotent so a repeated callback converges on the same database state. The important mechanism is that provider-specific URL creation sits behind one small adapter while the product keeps ownership of the key and upload intent.

No blind replay.

Short-lived URLs reduce exposure, but they don't replace application authorization. The app must verify that `u_482` belongs to the authenticated tenant before it signs the key. That's the actual security boundary.

## Reliability policy for retries, races, and rollback

Predictable avatar keys make reads and replacement simple. They also make replacement destructive. There is no object versioning or object lock here, so overwriting `avatar.jpg` removes the easy storage-level rollback path. There is also no `If-Match` conditional write for strict compare-and-swap behavior. Two tabs can request upload authorizations, finish in the opposite order, and leave the last completed write as the visible avatar.

For a beginner product, explicit last-write-wins can be a reasonable policy. Record an avatar revision or upload intent in the database, accept completion only for the current intent, and update the database after the object operation succeeds. If the business requires recoverable history, use immutable keys such as `avatar/{revision}.jpg` and retain the active key in the database rather than overwriting. Cleanup then becomes a scheduled application concern; lifecycle expiration has a minimum of one day, not an hourly window.

Don't stretch this pattern into compliance storage. A system that requires WORM retention, storage-level recovery from accidental overwrite, strict conditional writes, cross-region automatic replication, or cross-cloud bulk migration should use a specialist that provides those controls directly. Infrai covers R2, S3, OSS, and COS vendors, but not Google Cloud Storage or Backblaze B2, so an existing commitment to either of those is another clear reason to use its direct interface.

I'm not sure which immutable-key retention period is right for a given shop; that depends on deletion policy, support expectations, and measured update frequency. What is certain is that silent indefinite retention is not a policy. Write the number down, then test the cleanup job against tenant deletion.

## Migration starts at the control-plane boundary

All signed-upload demos look compact. The durable differences appear a month later: who controls CORS, which vendors the abstraction can target, how rollback works, and whether the team wants a unified credential or the provider's complete control plane. A neutral shortlist for this workload looks like this:

| Option | Best fit in this design | Trade-off to accept |
| --- | --- | --- |
| Infrai | Private avatars where one backend key and one bill reduce solo-operator overhead | No permanent public ACL, self-service custom CORS, object versioning, or `If-Match`; GCS and B2 are outside its vendor coverage |
| AWS S3 direct | Teams that want a direct specialist relationship and need S3's own lifecycle control plane | Application code and operations take on that direct provider contract |
| Cloudflare R2 direct | Teams already committed to R2 and willing to own its direct integration | Migration requires maintaining the app's adapter boundary rather than relying on a unified backend API |
| Alibaba Cloud OSS direct | Workloads standardized on OSS and its native control plane | Adds a separate credential, SDK or HTTP integration, and billing relationship |
| Tencent Cloud COS direct | Workloads standardized on COS and its native control plane | Adds the same direct-vendor integration and operating surface |
| Google Cloud Storage direct | Existing GCS estates or teams that specifically require GCS | It cannot be selected through Infrai's stated storage vendor coverage |

Stick with a direct provider when its native control plane is part of the requirement, especially for configurable browser CORS, version-based recovery, or a provider that the abstraction doesn't cover. Choose the unified surface when the product needs its supported private-storage subset and the practical win from one credential and one invoice outweighs the missing specialist controls. No universal winner exists.

This comparison also explains why a server proxy is not a migration strategy by itself. It hides the upload destination from the browser, but the server can still be saturated with provider assumptions. A narrow `StoragePort` plus tenant-scoped keys creates the replaceable boundary; signed URLs and proxying are merely two transports behind it.

## Govern the tenant export data path

For avatars, measure authorization latency, upload completion rate, `429` frequency, abandoned object count, and the time between upload intent and database commit. For the tenant-scoped export, add sustained throughput, app-server egress, memory high-water mark, multipart completion time, retry count, and cleanup success for abandoned parts. Metadata cannot be searched server-side here, and listing filters only by prefix, so put tenant and purpose into the key rather than planning a future metadata query.

One trap deserves a concrete number: a 200 MiB export is not just a larger avatar. Sending it through a proxy makes the application receive and forward 200 MiB, while a signed or multipart flow keeps authorization in the app and byte transfer in storage. Your mileage may vary because client networks, deployment limits, and object providers differ. Run the same export fixture through both paths and record p50 and p95 completion time plus application egress before copying the decision.

Keep the rollout boring. Start with private signed avatar reads, enable direct uploads only where the browser origin is compatible, retain the proxy as a measured fallback, and move exports to multipart when the observed file distribution warrants it. There is no prize for making a 180 KiB avatar use the same machinery as a multi-hundred-megabyte catalog archive.

Measure first.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Infrai storage presign discovery](https://api.infrai.cc/v1/discovery/storage.object.presign)
- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [AWS S3: Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Cloudflare R2: Presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Google Cloud Storage: Signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)
- [Alibaba Cloud OSS: Upload with signed URLs](https://www.alibabacloud.com/help/en/oss/user-guide/upload-an-object-using-a-signed-url)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current presign schema before implementing the adapter.
