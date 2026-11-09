# Hono

Receive AI Inbx webhooks in Hono on Cloudflare Workers, Bun, Deno and Node: verify the signature over the raw body and hand the event to a worker.

From https://docs.aiinbx.com/integrations/hono

Hono passes the underlying `Request` through as `c.req.raw`, which is exactly what the verifier wants — the body is still unread.

## Receiving webhooks

```ts
import { Hono } from "hono"
import { verifyWebhookRequest, WebhookSignatureError } from "aiinbx/webhooks"

type Env = { AI_INBX_WEBHOOK_SECRET: string }

const app = new Hono<{ Bindings: Env }>()

app.post("/webhooks/aiinbx", async (c) => {
  let event

  try {
    event = await verifyWebhookRequest(c.req.raw, c.env.AI_INBX_WEBHOOK_SECRET)
  } catch (error) {
    if (error instanceof WebhookSignatureError) {
      return c.text("Invalid signature", 400)
    }
    throw error
  }

  // Application-defined durable insert, unique on event.id.
  await enqueueOnce(event.id, event)

  return c.body(null, 204)
})

export default app
```
> [!WARNING]
> Use `c.req.raw`, not `await c.req.json()`. Reading the body as JSON consumes the stream and re-serializes it — the bytes that come back out are not the bytes that were signed, and every verification fails with no obvious cause.

## Durable acknowledgement

`enqueueOnce` is an application function backed by a database or queue. It must atomically deduplicate on `event.id` and resolve only after the job is stored durably. If storage fails, return non-2xx so AI Inbx can retry. Run business logic in a recoverable worker.

`waitUntil` alone does not provide durable processing. If you acknowledge a delivery before storing its work, a later failure is no longer recoverable through automatic webhook retries. See [Going to production](../guides/production.md#receive-events-durably).

## The client on Workers

Workers have no `process.env`, so pass the key explicitly from the binding rather than relying on the constructor's default:

```ts
import AIInbx from "aiinbx"

const aiinbx = new AIInbx({ apiKey: c.env.AI_INBX_API_KEY })
```
The SDK is `fetch`-only with no Node built-ins, so nothing else changes between runtimes.
