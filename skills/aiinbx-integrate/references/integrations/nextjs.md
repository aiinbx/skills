# Next.js

Use AI Inbx in Next.js: a route handler that verifies and receives webhooks, and a server action that sends email with the TypeScript SDK.

From https://docs.aiinbx.com/integrations/nextjs

## Install

```bash
npm install aiinbx
```
```bash .env.local
AI_INBX_API_KEY=aibx_live_...
# The endpoint's secret, or locally: aiinbx listen --print-secret
AI_INBX_WEBHOOK_SECRET=whsec_...
```
## One client

The constructor reads `AI_INBX_API_KEY` from the environment, so a module-scoped client needs no arguments. Keep it in one file — it's cheap to construct but pointless to rebuild per request.

```ts lib/aiinbx.ts
import AIInbx from "aiinbx"

export const aiinbx = new AIInbx()
```
> [!WARNING]
> Only import this from server code — a route handler, a server action, a server component. The key is a bearer token with full access to your workspace; anything that reaches a client bundle is public.

## Receiving webhooks

`verifyWebhookRequest` takes the `Request` itself, reads the raw body, and checks the `aiinbx-signature` header. Hand it the request before anything else parses it.

```ts app/api/aiinbx/route.ts
import { verifyWebhookRequest, WebhookSignatureError } from "aiinbx/webhooks"

export async function POST(request: Request) {
  let event

  try {
    event = await verifyWebhookRequest(request, process.env.AI_INBX_WEBHOOK_SECRET!)
  } catch (error) {
    if (error instanceof WebhookSignatureError) {
      return new Response("Invalid signature", { status: 400 })
    }
    throw error
  }

  // Application-defined: atomically store a durable job by event.id.
  await enqueueOnce(event.id, event)

  return new Response(null, { status: 204 })
}
```
`enqueueOnce` must persist the event before resolving and treat duplicates as successful no-ops. Storage failures must produce a non-2xx response. See [durable event processing](../guides/production.md#receive-events-durably).

Implementation notes:

- **`route.ts`, not `pages/api`.** The App Router hands you a real `Request` with an unread body. The Pages Router parses the body before your handler runs, which changes the bytes and breaks the signature — you'd have to disable `bodyParser` and reassemble the stream.
- **A 400 on a bad signature, not a 500.** A failed check means the request could not be authenticated. A mismatched secret or clock skew can also reject a legitimate delivery. Retrying it would be pointless, and the delivery log should say *rejected*, not *your server crashed*.
- **204 quickly.** The response is an acknowledgement. Deliveries [retry](../webhooks.md#retries) on a non-2xx, so slow work belongs on a queue, not in the handler.

`verifyWebhookRequest` uses Web Crypto, so this route runs on the Edge runtime unchanged.

## Sending

From a server action, resolve the authenticated user on the server. `requireCurrentUser` below is an application-defined authentication function; do not accept an arbitrary recipient and user ID from the browser.

```ts app/actions.ts
"use server"

import { aiinbx } from "@/lib/aiinbx"

export async function sendWelcome() {
  // Application-defined: authenticate the caller and load their verified address.
  const { id: userId, email: address } = await requireCurrentUser()
  await aiinbx.emails.send(
    {
      from: { name: "Acme", address: "hello@acme.com" },
      to: address,
      subject: "Welcome to Acme",
      text: "Your account is ready.",
    },
    { idempotencyKey: `welcome-${userId}` }
  )
}
```
The idempotency key is doing real work here: server actions can be re-invoked by a retry or a double submit, and a key derived from the user ID means the second attempt returns the first email instead of sending another one. Derive it from something stable — not a fresh `crypto.randomUUID()` per call, which defeats the purpose.

**[React Email](react-email.md)**

Pass a component instead of an HTML string.

**[Sending](../guides/sending.md)**

Attachments, headers, scheduling, and the response fields.
