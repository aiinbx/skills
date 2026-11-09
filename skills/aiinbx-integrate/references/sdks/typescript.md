# TypeScript

Install, configure, and use the aiinbx package — resources, pagination, response metadata, typed errors, React Email, and webhook verification.

From https://docs.aiinbx.com/sdks/typescript

The official TypeScript client. **No dependencies** — it uses `fetch` and Web Crypto, so it runs on Node.js 20+, Bun, Deno, Cloudflare Workers, and Vercel Edge without a polyfill.

```package-install
npm i aiinbx
```
## Getting started

```ts
import AIInbx from "aiinbx"

const aiinbx = new AIInbx() // reads AI_INBX_API_KEY

const email = await aiinbx.emails.send(
  {
    from: { name: "Ada", address: "ada@example.com" },
    to: "grace@example.com",
    subject: "Hello",
    text: "Sent through AI Inbx.",
  },
  { idempotencyKey: "welcome-grace-v1" }
)
```
## Configuration

```ts
const aiinbx = new AIInbx({
  apiKey: process.env.AI_INBX_API_KEY!,
  timeout: 60_000,
  maxRetries: 2,
  // baseURL: "http://localhost:3000/api/v2",
})
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `apiKey?` | `string` | - | Defaults to AI_INBX_API_KEY where the runtime exposes environment variables. |
| `baseURL?` | `string` | `"https://api.aiinbx.com/api/v2"` | Point at a local or self-hosted API. |
| `timeout?` | `number` | `60000` | Per-request timeout in milliseconds. |
| `maxRetries?` | `number` | `2` | Retries: a 429 always; network errors, 408 and 5xx when a retry can't act twice (a GET, PUT or DELETE, or a keyed send, reply or forward). |
| `fetch?` | `typeof fetch` | - | Supply your own fetch — for instrumentation or a proxy agent. |
| `defaultHeaders?` | `HeadersInit` | - | Headers added to every request. |

Every request can override the client defaults. `idempotencyKey` is taken only by the calls the API deduplicates: `emails.send`, `threads.reply` and `threads.forward`.

```ts
await aiinbx.emails.send(payload, {
  timeout: 10_000,
  maxRetries: 0,
  idempotencyKey: "order-8812-receipt",
  signal: controller.signal,
  headers: { "X-Request-ID": traceId },
})
```
## Resources

Each API resource is a property of the client — `aiinbx.emails`, `aiinbx.webhookEndpoints`, `aiinbx.pacingRules` — and every endpoint page in the [API reference](https://docs.aiinbx.com/api.md) shows its TypeScript call.

A method takes, in order, the path IDs, the query filters, the request body, and the request options. Bodies and responses are typed by the API's schema names — `SendEmailRequest`, `Email`, `FullThread`, `PacingRule`:

```ts
const email = await aiinbx.emails.retrieve("eml_123", {
  include: ["attachment_content"],
})

await aiinbx.pacingRules.updateSpread({ spread: 40 })
```
### Replying

No headers to reconstruct — the sender, recipients, subject, and reply headers come from the thread:

```ts
await aiinbx.threads.reply(
  "thr_123",
  { text: "Sounds good — see you then." },
  { idempotencyKey: "reply-thr-123-v1" }
)
```
## React Email

Pass a component as `react` and it's rendered to HTML before the send, the same way Resend's Node SDK does it:

```tsx
import AIInbx from "aiinbx"
import { WelcomeEmail } from "./emails/welcome"

const aiinbx = new AIInbx()

await aiinbx.emails.send({
  from: "Acme <hello@example.com>",
  to: "ada@example.com",
  subject: "Welcome",
  react: <WelcomeEmail name="Ada" />,
})
```
Install `@react-email/render` alongside React; `@react-email/components` alone is not enough. The renderer is imported lazily — only when `react` is present — so it costs nothing if you never use it.

> [!TIP]
> Keep passing `text` alongside `react`. A plain-text alternative renders in clients that won't show your HTML, and its absence is a small deliverability penalty.

## Pagination

A list call is awaitable for one page:

```ts
const page = await aiinbx.threads.list({ limit: 50, query: "invoice" })
```
…and iterable for all of them, carrying your filters onto every request:

```ts
for await (const thread of aiinbx.threads.list({
  mailbox: "team@example.com",
})) {
  console.log(thread.subject)
}

// A known-small collection, in full:
const domains = await aiinbx.domains.list().all()

// Page by page, when you want to checkpoint:
for await (const page of aiinbx.emails.list().iterPages()) {
  await saveCheckpoint(page.next_cursor)
}
```
A thread's messages paginate separately:

```ts
for await (const message of aiinbx.threads.iterateMessages("thr_123", {
  message_limit: 100,
})) {
  console.log(message.subject)
}
```
`emails.follow` and `events.follow` yield every item added to the list as it arrives, reconnecting until you stop them — see [Following a list live](../reference/conventions.md#live).

## Response metadata

Every call is awaitable. `.withResponse()` gets you the headers and request ID too:

```ts
const { data, response, requestId } = await aiinbx.emails
  .retrieve("eml_123")
  .withResponse()

console.log(response.status, requestId, data.subject)
```
## Errors

Non-2xx responses throw `APIError` with `status`, `code`, `requestId`, `headers`, and the parsed `body`. Subclasses let you branch without comparing numbers:

```ts
import { APIError, NotFoundError, RateLimitError } from "aiinbx"

try {
  await aiinbx.domains.retrieve("dom_missing")
} catch (error) {
  if (error instanceof NotFoundError) return null
  if (error instanceof RateLimitError) return scheduleRetry()
  if (error instanceof APIError) console.error(error.status, error.code)
  throw error
}
```
| Class                      | Status |
| -------------------------- | ------ |
| `BadRequestError`          | 400    |
| `AuthenticationError`      | 401    |
| `PermissionDeniedError`    | 403    |
| `NotFoundError`            | 404    |
| `RequestTimeoutError`      | 408    |
| `ConflictError`            | 409    |
| `UnprocessableEntityError` | 422    |
| `RateLimitError`           | 429    |
| `InternalServerError`      | 5xx    |

Codes and what to do about each are in [Errors](../reference/errors.md).

## Webhooks

`verifyWebhookRequest` handles the raw body, header parsing, timing-safe comparison, and the five-minute replay window:

```ts app/webhooks/aiinbx/route.ts
import { verifyWebhookRequest } from "aiinbx/webhooks"

export async function POST(request: Request) {
  const event = await verifyWebhookRequest(
    request,
    process.env.AI_INBX_WEBHOOK_SECRET!
  )

  // `type` is a discriminant — TypeScript knows this event's exact data.
  if (event.type === "email.received") {
    console.log(event.data.thread_id, event.data.category)
  }

  return new Response(null, { status: 204 })
}
```
When your framework hands you the body and signature separately:

```ts
import { verifyWebhook } from "aiinbx/webhooks"

const event = await verifyWebhook(rawBody, signature, secret)
```
Both throw `WebhookSignatureError`. More, including per-framework raw-body recipes, in [Verifying](../webhooks/verifying.md).

## Types

Every request and response type is exported from the package root:

```ts
import type {
  Email,
  FullThread,
  SendEmailParams,
  WebhookEvent,
  WebhookEventType,
} from "aiinbx"

function summarize(thread: FullThread): string {
  return thread.messages.map((m) => m.snippet).join("\n")
}
```
`WebhookEvent` is a discriminated union over `type`, so a `switch` narrows `data` in each branch with no casts.

## Runtimes

| Runtime            | Notes                                                                  |
| ------------------ | ---------------------------------------------------------------------- |
| Node.js 20+        | `fetch` and Web Crypto are built in.                                   |
| Bun                | Works as-is.                                                           |
| Deno               | Works as-is.                                                           |
| Cloudflare Workers | Works. `apiKey` must be passed explicitly — there's no `process.env`.  |
| Vercel Edge        | Same.                                                                  |
| Browsers           | Technically works; don't. It would ship your API key to every visitor. |
