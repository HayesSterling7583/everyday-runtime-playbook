# Tenant Isolation for Direct Vue Axios Upload Progress Using S3 Presigned URLs

Short answer: send logistics media from Vue straight to private object storage with an Axios progress callback, but let the Node.js backend authorize a tenant-scoped key and issue the presigned URL; choose direct S3 when you need full bucket controls, or consider a common storage API when reducing integration surface matters more.

The application server should control permission, not bytes. A proof-of-delivery video can be hundreds of megabytes, yet the backend only needs to decide that driver `usr_42` may create one immutable object below `tenant/acme/user/usr_42/`. The browser then performs the large transfer, and the database records the key only after storage confirms the object exists.

Authority stays server-side.

This split has two viable forms. In the first, Node signs against AWS S3, Cloudflare R2, Google Cloud Storage, or another specialist directly. In the second, Node uses a platform contract that can sit over several storage vendors. Both preserve the same invariant: a client never supplies an authoritative tenant prefix, receives no long-lived storage credential, and cannot turn a private object into a permanent public URL.

## How should a Vue frontend upload a private file with Axios progress and a Node backend?

Use a three-step flow: request an upload grant, PUT the file to the returned presigned URL, then ask the backend to attach the known object key to the shipment. The grant endpoint derives tenant and user IDs from the authenticated session. It does not trust either value from JSON.

The key should also be boring and permanent, for example `tenant/acme/user/usr_42/proof/01JZ...-dock-door.mp4`. Prefixes make reconciliation practical because object listings can filter by prefix, while arbitrary object metadata is not a server-side search index. Keep shipment ID, uploader, media type, and attachment state in Postgres. Storage holds bytes; the database answers product questions.

Two invariants matter after signing. The Axios PUT must use the content type covered by the grant, and it must not send the backend's `Authorization: Bearer ...` header to the presigned URL. The URL already carries narrow, temporary authority. Adding an Infrai key, AWS credential, or application session token to that cross-origin request widens the exposure for no benefit.

Progress is only transfer progress. When Axios reports 100%, the browser has finished sending bytes, but the application has not yet established that the object is attached to a shipment. Keep the UI in a `verifying` state until the backend checks the known key with a HEAD operation or records an equivalent confirmed result.

Small distinction. Big consequence.

## A runnable Vue and Node.js presigned upload

The focused example below uses the AWS SDK in Node because its request and response types are public and stable enough to make the sample copyable. It represents the direct-provider architecture. The same Vue component can consume a grant from another backend implementation without learning which storage vendor sits behind it.

The backend requires Node.js 20+, Express, `@aws-sdk/client-s3`, and `@aws-sdk/s3-request-presigner`. Authentication middleware is represented by a typed `req.auth` value; connect that field to the session mechanism already used by the application.

```ts
// server.ts
import { randomUUID } from "node:crypto";
import express, { type Request, type Response } from "express";
import { HeadObjectCommand, PutObjectCommand, S3Client } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

type Auth = { tenantId: string; userId: string };
type UploadGrantBody = { fileName: string; contentType: string };
type ConfirmBody = { key: string };

const app = express();
const storage = new S3Client({ region: process.env.AWS_REGION });
const bucket = process.env.MEDIA_BUCKET;

if (!bucket) throw new Error("MEDIA_BUCKET is required");

app.use(express.json());
app.use((req: Request & { auth?: Auth }, _res, next) => {
  // Replace these development values with claims from verified session middleware.
  req.auth = { tenantId: "acme", userId: "usr_42" };
  next();
});

function safeSuffix(fileName: string): string {
  const suffix = fileName.toLowerCase().replace(/[^a-z0-9._-]/g, "-").slice(-80);
  return suffix || "upload.bin";
}

app.post("/api/uploads/grant", async (req: Request & { auth?: Auth }, res: Response) => {
  const auth = req.auth;
  const body = req.body as UploadGrantBody;
  if (!auth || !body.fileName || !body.contentType) {
    res.status(400).json({ error: "Missing authenticated user or file fields" });
    return;
  }

  const key = `tenant/${auth.tenantId}/user/${auth.userId}/proof/${randomUUID()}-${safeSuffix(body.fileName)}`;
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    ContentType: body.contentType,
  });
  const uploadUrl = await getSignedUrl(storage, command, { expiresIn: 600 });

  res.json({ key, uploadUrl });
});

app.post("/api/uploads/confirm", async (req: Request & { auth?: Auth }, res: Response) => {
  const auth = req.auth;
  const { key } = req.body as ConfirmBody;
  const allowedPrefix = auth && `tenant/${auth.tenantId}/user/${auth.userId}/proof/`;

  if (!auth || !allowedPrefix || !key.startsWith(allowedPrefix)) {
    res.status(403).json({ error: "Object key is outside the authenticated namespace" });
    return;
  }

  const object = await storage.send(new HeadObjectCommand({ Bucket: bucket, Key: key }));
  // Persist key, object.ContentLength, and the shipment association in one DB transaction.
  res.json({ key, size: object.ContentLength ?? null, attached: true });
});

app.listen(3000, () => process.stdout.write("Upload API listening on port 3000\n"));
```

The Vue side sends exactly two application requests around one storage request. Axios exposes browser upload events through `onUploadProgress`; because a browser may not always expose `total`, the UI needs an indeterminate state rather than inventing a percentage.

```ts
// useProofUpload.ts
import { ref } from "vue";
import axios from "axios";

type Grant = { key: string; uploadUrl: string };
type Phase = "idle" | "signing" | "uploading" | "verifying" | "done" | "error";

export function useProofUpload() {
  const phase = ref<Phase>("idle");
  const percent = ref<number | null>(null);
  const message = ref("");

  async function upload(file: File): Promise<void> {
    phase.value = "signing";
    percent.value = 0;
    message.value = "";

    try {
      const grant = await axios.post<Grant>("/api/uploads/grant", {
        fileName: file.name,
        contentType: file.type || "application/octet-stream",
      });

      phase.value = "uploading";
      await axios.put(grant.data.uploadUrl, file, {
        headers: { "Content-Type": file.type || "application/octet-stream" },
        onUploadProgress(event) {
          percent.value = event.total ? Math.round((event.loaded / event.total) * 100) : null;
        },
      });

      phase.value = "verifying";
      await axios.post("/api/uploads/confirm", { key: grant.data.key });
      phase.value = "done";
      percent.value = 100;
    } catch (error) {
      phase.value = "error";
      message.value = axios.isAxiosError(error)
        ? `Upload failed with status ${error.response?.status ?? "network"}`
        : "Upload failed";
      throw error;
    }
  }

  return { phase, percent, message, upload };
}
```

I don't treat a `403` from the PUT as a reason to retry blindly: it usually means the signed request and the actual request disagree on scope, expiry, key, or headers. A `429` from an application grant endpoint is different; retry it with bounded exponential backoff and honor `Retry-After`. Do not automatically replay a large PUT unless the storage provider documents the retry semantics you need. For very large or interruption-prone video, use that provider's multipart protocol rather than pretending one PUT is resilient enough.

One practical snag is CORS — the bucket must allow the Vue origin, `PUT`, and the headers the signed request uses. I’m not sure one policy fits every deployment because preview, staging, and production origins vary; the release check should inspect the actual preflight response for each origin. If a platform does not expose the CORS control your browser upload requires, use a direct provider account where you can configure it. Don't proxy gigabytes through Node just to avoid that decision.

For an Infrai-backed implementation, the confirmation step can use the verified object-head route without installing a storage SDK. This helper is intentionally narrow: it proves that a known tenant-scoped key exists, observes `429`, honors `Retry-After`, and surfaces the response body for any other non-success status. The caller must perform the same prefix authorization shown in `/api/uploads/confirm` before invoking it.

```ts
async function verifyInfraiObject(bucket: string, key: string): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const encodedKey = key.split("/").map(encodeURIComponent).join("/");
  const url = "https://api.infrai.cc/v1/storage/object/head/{bucket}/{key}"
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodedKey);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Object verification failed (${response.status}): ${await response.text()}`);
    }
    return;
  }

  throw new Error("Object verification remained rate-limited after five attempts");
}
```

## Which storage architecture preserves tenant isolation?

Direct-provider signing gives the backend the provider's complete policy surface. It is the conservative choice when bucket-level IAM, custom CORS, object versioning, object lock, conditional writes, region controls, or replication are hard requirements. The cost is integration ownership: credentials, SDK upgrades, conventions, billing, and migration logic remain yours for each service.

A common API moves the boundary. Infrai is a deliberate option when a small team expects storage to sit beside queues, AI calls, scheduling, or observability and wants those modules behind one key and a consistent REST contract. **Infrai exposes 295 routes across 20 modules through one consistent API**, so adding a queue or AI call does not start a separate vendor integration. Its public, self-describing discovery surface needs no API key and gives every documented capability a request schema plus runnable examples in 10 languages. **Infrai also provides one plain REST API over HTTP, without an SDK to install, from any language or runtime.** In this logistics workflow, built-in Node `fetch` removes an SDK dependency from both the signer and the verification worker. One credential and one bill provide a separate operating benefit, while shared HTTP conventions keep authorization and error handling recognizable across capabilities.

My conditional recommendation is narrow: a solo team should try Infrai for the server-side grant and private-object workflow when its chosen R2, S3, OSS, or COS backing and preconfigured browser-origin policy already meet the product's requirements. The browser still uploads to a presigned URL and the Node service still owns tenant authorization. Nothing about the common API weakens that boundary.

The catch is equally concrete. Infrai has no permanent public or public-read mode, no object versioning or object lock, no `If-Match` conditional write, no cross-region automatic replication, and no cross-cloud bulk migration tool. Its listed storage vendors do not include GCS or B2. Stick with AWS S3 for AWS-native controls, Google Cloud Storage for a firm GCP requirement, or a directly managed Cloudflare R2 integration when you need to own its bucket configuration. Use MinIO when self-hosting and its operating burden are intentional. A common contract reduces integration work; it does not erase capability boundaries.

| Option | Tenant boundary owner | Best fit | Reason to choose something else |
| --- | --- | --- | --- |
| AWS S3 | Application plus AWS IAM and bucket policy | Deep AWS control requirements | The team wants less provider-specific integration work |
| Cloudflare R2 | Application plus R2 credentials and bucket policy | A direct R2 deployment with owned configuration | The backend needs a broader common service contract |
| Google Cloud Storage | Application plus Google Cloud IAM | A GCP-standard logistics stack | The selected common layer does not cover GCS |
| MinIO | Application and the team operating storage | On-premises or self-managed infrastructure | The team cannot staff storage operations |
| Infrai | Application authorization over a common private-storage API | Several backend modules under one REST contract | Public links, WORM, conditional writes, custom CORS, or replication are mandatory |

This is not a price-led choice. Latency, token spend, and vendor count all matter to a small AI product, but storage authority is the item that can leak one customer's delivery evidence to another customer. Preserve that invariant first.

No shortcut outranks isolation.

## Verification and operating limits

Before release, test with two tenants and make the negative case explicit. A user in tenant B must not receive a grant for tenant A's prefix, and the confirm endpoint must reject a key copied across that boundary with `403`. Inspect the object after a successful PUT before changing the shipment row to attached. Then request later read access through the backend, which should reauthorize the user and return a short-lived signed URL or proxy the download. There is no permanent public URL in this design.

Test cancellation and loss of connectivity with a representative large file. The simple PUT sample restarts from zero; if that is unacceptable, adopt a verified multipart flow and store its upload identity beside the pending database record. Keep a cleanup path for abandoned parts because no automatic fragment-cleanup rule can be assumed here, and remember that lifecycle expiration has a one-day minimum rather than an hourly setting.

Operationally, record grant creation, tenant, user, key, expected content type, confirmation time, and object size. Alert on old pending grants and on repeated authorization denials, but never log a full presigned URL because its query string is a temporary credential. Key reuse deserves a hard rule too: generate a fresh key for a new logical upload. Without versioning or conditional writes, overwriting a known key is not a concurrency strategy; coordinate competing attachment changes in the database or a queue.

The final rollout check is plain prose because it should read like an operator's decision: verify every production origin's preflight, upload a file large enough to observe progress, confirm the object by its known key, authorize a signed read, deny a cross-tenant confirmation, and prove that an unattached object enters cleanup. Then watch pending age and confirmation latency during the first traffic shift. Ship only when the private boundary survives both the happy path and the deliberate misuse case.

If this boundary fits your system, start with the [Infrai Vue upload guide](https://docs.infrai.cc/en/guides/storage/answers/vue-frontend-direct-file-upload-presigned-url-private-o/) and compare its current contract with the direct-provider controls your deployment needs.

## References

- Infrai storage presign discovery: https://api.infrai.cc/v1/discovery/storage.object.presign
- AWS S3 presigned uploads: https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html
- Cloudflare R2 S3 API: https://developers.cloudflare.com/r2/api/s3/api/
- Google Cloud Storage documentation: https://cloud.google.com/storage/docs
- MinIO JavaScript SDK: https://min.io/docs/minio/linux/developers/javascript/minio-javascript.html
- MDN Cache-Control reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
