# Going to production

Go to production with AI Inbx: reliable, idempotent sends, durable webhook processing, customer boundaries, and monitoring of delivery outcomes.

From https://docs.aiinbx.com/guides/production

A production integration needs to recover from uncertain send results, duplicate events, and provider failures. This guide describes the application responsibilities around the [API conventions](https://docs.aiinbx.com/reference/conventions.md).

## Understand what success means

| Signal | Meaning | Application action |
| --- | --- | --- |
| Successful send response | An email resource was created or an existing request was replayed | Store the email and thread IDs; inspect `status`, `suppressed`, and `pacing` |
| `email.sent` | The sending provider accepted the message | Record provider acceptance; do not mark inbox delivery |
| `email.delivered` | A delivery confirmation was received | Record the reported delivery; it does not prove the recipient read the message |
| `email.bounced` | A delivery failure was reported | Inspect each recipient's cause and whether it was suppressed |
| `email.complained` | A recipient complaint was reported | Update your application state and inspect suppression handling |
| HTTP timeout | Your application did not receive a complete result | Retry the same operation with the same key and payload |

Available outcomes depend on the sending provider. Check the [event reference](https://docs.aiinbx.com/webhooks/events.md) for provider-specific behavior. Events can arrive more than once and out of order.

## Make sends recoverable

Persist a logical operation ID, its idempotency key, and its request body before sending. Reuse them across HTTP retries, worker retries, and process restarts.

```ts
// `operation` is a persisted application record, not a new object per attempt.
const email = await aiinbx.emails.send(operation.payload, {
  idempotencyKey: operation.idempotencyKey,
})
await saveSendResult(operation.id, email.id, email.thread_id)
```
The SDKs retry some failures automatically. A key still matters: neither client generates a persistent application key for you. An uncertain response is not a reason to generate a new key.

If you generate message content with a model, save the selected content before sending. Regenerating it on a retry can produce a different body and a `409 idempotency_key_reused` response. See [idempotency scope and replay behavior](https://docs.aiinbx.com/reference/conventions.md#idempotency).

Use bounded retries with backoff, and inspect structured [error codes](https://docs.aiinbx.com/reference/errors.md). Do not retry a validation error or a key/body conflict unchanged.

## Receive events durably

The webhook handler has **10 seconds** to respond. Verify the signature, store the event durably, and acknowledge it. Run model calls and other business logic in a worker.

```ts
import { verifyWebhookRequest, WebhookSignatureError } from "aiinbx/webhooks"

export async function POST(request: Request) {
  let event
  try {
    event = await verifyWebhookRequest(
      request,
      process.env.AI_INBX_WEBHOOK_SECRET!
    )
  } catch (error) {
    if (error instanceof WebhookSignatureError) {
      return new Response("Invalid signature", { status: 400 })
    }
    throw error
  }

  // Application-defined: commit a durable job with a unique event ID.
  // An existing ID is a no-op. Storage failures must reject this call.
  await enqueueOnce(event.id, event)
  return new Response(null, { status: 204 })
}
```
`enqueueOnce` is an integration point for your database or queue, not an SDK method. It must:

1. Atomically store the event with a unique constraint on `event.id` or equivalent deduplication.
2. Treat duplicate deliveries as success without creating another job.
3. Return only after durable storage succeeds. On failure, let the handler return non-2xx so AI Inbx retries.

A worker should mark a job complete only after its work succeeds. Use a recoverable claim or lease so a worker crash does not leave a job permanently in progress. If you use a database plus a separate queue, use an outbox or another recovery mechanism for the gap between committing the event and publishing the job.

Returning `204` and starting untracked background work is not durable acceptance. Once acknowledged, AI Inbx will not retry that successful delivery if your later work fails.

## Handle concurrency and ordering

Deduplicate by `event.id`, not by the signature or delivery timestamp. Signatures change across attempts; the event ID stays the same across retries and manual replays.

No webhook concurrency setting guarantees event order. Use timestamps for your event history and retrieve the resource when you need current state. Do not overwrite newer state merely because an older event arrived later.

For an automated conversation, serialize jobs per thread or use an application-level version check. Two different inbound events have different IDs and can otherwise produce overlapping replies. Before sending, check whether a person or another worker has already answered.

## Enforce customer boundaries

An API key reaches every space in its workspace within the key's operation scope. A `sending` key is not a credential restricted to one customer.

Keep keys on your servers. Resolve the authenticated customer's space in your application, validate ownership of requested senders and resources, and apply `space` filters to list operations. A resource ID supplied by a browser is not proof of ownership.

Webhook endpoints also receive events across spaces. Resolve `event.space_id` against your own customer records after verifying the signature. Treat `null` explicitly as workspace-level mail. See [Spaces](https://docs.aiinbx.com/guides/spaces.md).

## Monitor and recover

Record operation IDs, email IDs, thread IDs, event IDs, and `X-Request-ID` alongside your application traces. Avoid logging API keys, webhook secrets, message bodies, or signed attachment URLs by default.

Monitor these conditions:

| Condition | Where to investigate |
| --- | --- |
| Failed webhook deliveries | [Delivery attempts and replay](https://docs.aiinbx.com/webhooks.md#inspecting-deliveries) |
| Mail held longer than expected | [Pacing queue](https://docs.aiinbx.com/guides/pacing.md#the-queue) and scheduled time |
| Sending domain's DKIM record goes missing | [`domain.at_risk`](https://docs.aiinbx.com/webhooks/events/domain-at-risk.md), 72 hours before it stops sending |
| Sending domain loses verification | [`domain.lost`](https://docs.aiinbx.com/webhooks/events/domain-lost.md) and domain diagnostics |
| Connected account needs authorization | [`mailbox.needs_reauth`](https://docs.aiinbx.com/webhooks/events/mailbox-needs-reauth.md) |
| Recipients excluded or rejected | `suppressed`, bounce events, and [suppression lists](https://docs.aiinbx.com/guides/suppressions.md) |
| Repeated API failures | Structured error code and request ID |

Automatic webhook retries have a bounded window. After fixing your handler, inspect failed deliveries and replay them. Disabling an endpoint does not queue events created while it is disabled — but every event is recorded whether or not an endpoint heard it, so a handler that was down, paused, or not yet subscribed [catches up from `GET /events`](https://docs.aiinbx.com/webhooks.md#catching-up).

Store the data your application needs for its own history and recovery. Attachment URLs expire; retain attachment IDs and fetch fresh URLs as needed. Confirm retention and residency requirements before relying on the service as an archive.

## Verify the integration before launch

Review [Plans and billing](https://docs.aiinbx.com/guides/billing.md) for email and attachment-processing allowances, spending caps, and the API responses your application should handle when it reaches a limit.

Exercise a complete send, delivery event, inbound reply, and thread reply with addresses you control. Also check that:

- An invalid webhook signature is rejected without creating work.
- Concurrent deliveries of one event create one durable job.
- A worker can resume after crashing without generating a different send payload.
- Failed storage causes a non-2xx webhook response.
- A customer cannot read or send through another customer's resources.
- Operators can find a failed delivery and replay it after a fix.

A synthetic webhook tests your endpoint and signature handling. It does not establish that DNS, provider delivery, or inbound routing works; test those with an actual message.
