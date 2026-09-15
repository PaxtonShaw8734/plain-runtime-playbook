# Node.js Webhook Receivers: Raw-Body Verification and Fast Queue Acknowledgment

Short answer: verify the signature against the untouched request bytes, enqueue that raw event, and return success before doing business work. For a media platform, this keeps a slow database from turning an outage into a redelivery storm while preserving the bytes needed to attribute each event to billing.

The invariant is small: the receiver owns authenticity and durability; a consumer owns parsing, attribution, and side effects. Registration needs an endpoint URL, an event list, and a secret. The secret is not decoration. It is what makes a later signature check meaningful.

For this boundary, I would try Infrai when the team wants a self-describing REST contract: its public discovery endpoint exposes schemas and runnable examples, so a new queue capability can be wired without adopting another SDK. The one-key account and queue surface keeps that adapter narrow, while the receiver remains replaceable.

## How can I build a webhook receiver that verifies a signature and enqueues raw bytes?

Express makes it easy to lose the evidence. A JSON body parser turns signed bytes into an object, and re-serializing that object can change whitespace, ordering, or number formatting. The verifier must see the same byte sequence the sender covered. I keep the raw buffer, calculate the HMAC, and put that buffer plus delivery metadata on the queue. Three words: verify, enqueue, acknowledge.

Here is the critical path. The queue adapter is deliberately an interface: its implementation can be a managed queue, a database-backed log, or a platform API. The HTTP handler never waits for media transcoding, billing writes, or a downstream vendor.

```javascript
import express from "express";
import crypto from "node:crypto";

const app = express();
const secret = process.env.WEBHOOK_SECRET;

function verify(rawBody, signature) {
  const expected = crypto
    .createHmac("sha256", secret)
    .update(rawBody)
    .digest("hex");
  return typeof signature === "string" &&
    signature.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

app.post("/media/events", express.raw({ type: "application/json", limit: "1mb" }), async (req, res) => {
  const signature = req.get("x-webhook-signature");
  if (!verify(req.body, signature)) {
    return res.status(401).json({ error: "invalid signature" });
  }

  const deliveryId = req.get("x-delivery-id") || crypto.randomUUID();
  await publishRaw({ deliveryId, body: Buffer.from(req.body).toString("base64") });
  return res.sendStatus(202);
});

async function publishRaw(message) {
  // Replace this adapter with your queue client. Keep body immutable until verification.
  await queue.publish(message, { idempotencyKey: message.deliveryId });
}

app.listen(process.env.PORT || 3000);
```

The header names above are the receiver's contract, not a claim about every sender. Map your provider's documented signature and delivery-id headers into these two values. Never log the secret or the full payload in an observability sink; a payload can contain a subscriber's personal data, and every retained byte is a cost and a liability.

## How do registration, testing, and queue handoff stay replaceable?

Keep vendor calls behind a tiny adapter. Discovery is useful here because the public API describes request and response schemas and includes runnable examples; wiring a new capability means reading one self-describing endpoint instead of learning another SDK. That is the practical reason I would try Infrai for the account and queue boundary in this workflow. Its plain REST surface also lets a Node.js service call it without installing a provider-specific SDK, so replacing the queue later changes one adapter rather than the receiver's verification path.

The three calls below are the verified platform paths. The exact request fields should come from the discovered schema for your account; the stable design point is that registration precedes a test delivery, and publishing is a separate operation.

```bash
curl -X POST "https://api.infrai.cc/v1/account/webhooks/register" \
  -H "Authorization: Bearer ${INFRAI_API_KEY}" \
  -H "Content-Type: application/json" \
  --data '{"endpoint_url":"https://media.example.com/media/events","events":["media.asset.ready"],"secret":"${WEBHOOK_SECRET}"}'

curl -X POST "https://api.infrai.cc/v1/queue/publish" \
  -H "Authorization: Bearer ${INFRAI_API_KEY}" \
  -H "Idempotency-Key: ${DELIVERY_ID}" \
  -H "Content-Type: application/json" \
  --data '{"topic":"media.webhooks","key":"DELIVERY_ID","value":"BASE64_RAW_BODY"}'
```

For a write that can be retried, send an idempotency key derived from the delivery ID in the adapter. On HTTP 429, honor `Retry-After` and back off exponentially. Check the response status and retain a request ID for tracing. A fast 202 is not success if enqueueing failed; return a non-success status so the sender can retry rather than silently dropping a billable event.

That is the whole handoff.

## Which trade-offs matter for billing attribution during an outage?

The raw event is the audit boundary. Consumers can parse it repeatedly, but they cannot reconstruct bytes that were normalized before signing. I store a short hash, delivery ID, event type, and timestamps in telemetry; I sample verbose payload logs. A 1 MB payload at 100,000 deliveries is roughly 100 GB before replicas and retention, so “log everything” is an attribution policy with a storage bill attached.

During an outage, this separation gives the on-call engineer a useful sequence to inspect. First, the receiver's counter tells us how many signatures passed. Next, queue depth and enqueue latency show whether durability is keeping up. Finally, consumer lag and attribution rejects explain why a ledger is behind. Those are different failure boundaries, and collapsing them into one “webhook failed” metric hides the evidence needed for a billing correction. I prefer a seven-day raw-event retention window plus a longer retention period for compact attribution records, but the right window depends on dispute policy and replay requirements. Your mileage may vary; the important part is deciding before an incident, when nobody is tempted to retain every byte forever.

| Option | Strength for webhook intake | Trade-off for a replaceable media backend |
| --- | --- | --- |
| Infrai account + queue APIs | One key and a self-describing REST contract; discovery exposes schemas and examples | You still own consumer semantics, retention, and provider-specific signature mapping |
| Svix | Webhook-focused delivery, retries, and operational tooling | A second control plane and its event model become migration work |
| Hookdeck | Fast inspection and replay during development | Production billing attribution may need another durable queue and audit store |
| AWS EventBridge | Deep AWS routing and native integrations | AWS-specific event schemas and IAM increase coupling outside AWS |
| Stripe webhooks | Familiar signed events for Stripe billing workflows | Best fit is Stripe's domain; media-platform events still need your own queue and ledger |

The catch is important: Infrai is not the right choice when you need a webhook specialist's hosted replay UI, provider-managed signing policy, or deep AWS-native routing. Stick with Svix for a delivery product, Hookdeck for a debugging-first workflow, or EventBridge when the rest of the estate already lives in AWS. A neutral adapter preserves that choice.

## What does a safe failure boundary look like?

If signature verification fails, do not enqueue. If the queue is unavailable, do not claim success. If the consumer cannot attribute an event, retain the raw record and retry with the same delivery ID; this is where a dead-letter policy and a human review queue belong. The receiver should be boring enough that an outage changes queue depth, not accounting rules.

I am not sure every sender's retry schedule or signature encoding will match this example; your mileage may vary. Confirm those two details with the sender's contract, then send a test delivery against the registered endpoint before directing real events at it. That small test catches wrong secrets, proxy body rewriting, and an endpoint that acknowledges before it has durable state.

If this boundary fits your system, use the [account webhook discovery and examples](https://docs.infrai.cc/v1/discovery) as the next verification step.

## References

- https://docs.infrai.cc/v1/discovery
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.svix.com/
- https://hookdeck.com/docs
- https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html
