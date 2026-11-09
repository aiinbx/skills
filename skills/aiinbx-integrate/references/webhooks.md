# Webhooks

Register an endpoint, choose events, route them by address, and understand what happens when a delivery fails.

From https://docs.aiinbx.com/webhooks

Webhooks are how anything that happens after a send reaches your code — a reply arriving, a bounce, a complaint, a mailbox losing its authorization. Everything asynchronous shows up here.

**[Verifying](webhooks/verifying.md)**

Check the signature before you parse the body.

**[Events](webhooks/events.md)**

Every event type and the shape of its `data`.

**[Receiving](guides/receiving.md)**

How inbound mail gets here in the first place.

## Creating an endpoint

```ts TypeScript
const endpoint = await aiinbx.webhookEndpoints.create({
  url: "https://app.example.com/webhooks/aiinbx",
  subscriptions: [
    "email.received",
    "email.bounced",
    "email.complained",
    "email.unsubscribed",
  ],
})

console.log(endpoint.secret) // returned once — store it now
```
```python Python
endpoint = client.webhook_endpoints.create(
    {
        "url": "https://app.example.com/webhooks/aiinbx",
        "subscriptions": ["email.received", "email.bounced", "email.complained"],
    }
)

print(endpoint["secret"])
```
```bash curl
curl https://api.aiinbx.com/api/v2/webhook-endpoints \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://app.example.com/webhooks/aiinbx",
    "subscriptions": ["email.received", "email.bounced", "email.complained"]
  }'
```
`subscriptions` is required and must name at least one [event type](webhooks/events.md) — an endpoint subscribed to nothing would never fire. Subscribe to what you handle; you can always widen later with `update`. `max_concurrency` sets how many deliveries may be in flight at once; see [Throughput](#throughput). `payload` decides whether a message event brings the email with it; see [Full payloads](#full-payloads).

Managing endpoints requires a `full` [scope](https://docs.aiinbx.com/authentication.md#scopes) key.

> [!WARNING]
> The signing secret is returned once, on creation — no later read of the endpoint shows it. If you lose it, [rotate](#rotating-the-secret): `POST /webhook-endpoints/{id}/rotate-secret` returns a new one and keeps the old one verifying for a grace period.

## The request

Every delivery is a `POST` with a JSON envelope:

```json
{
  "id": "evt_...",
  "type": "email.received",
  "created_at": "2026-09-01T10:31:04Z",
  "space_id": null,
  "data": {}
}
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `id` | `string` | - | evt_… — stable across retries and replays. |
| `type` | `string` | - | An event type listed in the event reference. |
| `created_at` | `string` | - | When the event happened, RFC 3339. |
| `space_id` | `string \| null` | - | The space the resource is in — the email's, thread's, domain's or mailbox's. null is the workspace itself, and all a workspace without spaces ever sees. |
| `data` | `object` | - | Varies by type. |

| Header             | Value                                                               |
| ------------------ | ------------------------------------------------------------------- |
| `Content-Type`     | `application/json`                                                  |
| `AIInbx-Signature` | `t=<unix seconds>,v1=<hmac>` — see [Verifying](webhooks/verifying.md) |
| `AIInbx-Event-ID`  | The event ID, also `id` in the body                                 |
| `User-Agent`       | `aiinbx-webhooks/1`                                                 |

`id` is stable across retries of the same event, which makes it the right idempotency key on your side.

### Responding

Answer `2xx` and answer fast. Anything else — or no answer within **10 seconds** — counts as a failure and is retried.

```ts
export async function POST(request: Request) {
  const event = await verifyWebhookRequest(request, secret)

  await enqueueOnce(event.id, event) // durable, atomic insert keyed by event ID

  return new Response(null, { status: 204 })
}
```
`enqueueOnce` is an application-defined operation: it must durably store the event and atomically deduplicate by `event.id`. Return `2xx` only after storage succeeds. If storage fails, return a non-2xx response so delivery can retry. Process the event in a worker with its own retry policy. See the [production guide](guides/production.md#receive-events-durably).

## Full payloads

By default a message event is a summary — addresses, subject, snippet, attachment metadata — and the handler fetches the email when it needs the body. An endpoint created with `payload: "full"` is sent the email along with it, so there is no second call to make:

```ts TypeScript
await aiinbx.webhookEndpoints.create({
  url: "https://app.example.com/webhooks/aiinbx",
  subscriptions: ["email.received"],
  payload: "full",
})
```
```python Python
client.webhook_endpoints.create(
    {
        "url": "https://app.example.com/webhooks/aiinbx",
        "subscriptions": ["email.received"],
        "payload": "full",
    }
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/webhook-endpoints \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://app.example.com/webhooks/aiinbx",
    "subscriptions": ["email.received"],
    "payload": "full"
  }'
```
[`email.received`](webhooks/events/email-received.md) and [`email.sent`](https://docs.aiinbx.com/webhooks/events/email-sent.md) then carry `data.email`: exactly the object [`GET /emails/{id}?include=attachment_content`](https://docs.aiinbx.com/api/endpoints/emails/retrieve-email.md) returns — both bodies and their stripped forms, segments, headers, every attachment with its signed `download_url`, and the [prepared text](https://docs.aiinbx.com/guides/attachments.md) of each document inline as `preparation.text`. The summary fields stay where they are, so one handler reads both kinds of endpoint. Every other event is the same either way. `update` switches an existing endpoint, and the change applies from the next attempt — retries of earlier events included.

```ts
if (event.type === "email.received") {
  const email =
    event.data.email ?? (await aiinbx.emails.retrieve(event.data.email_id))
  await queueReply(event.data.thread_id, email.stripped_text)
}
```
Keep that fallback. `data.email` is `null` when the email could not be included:

- **The message is gone** by the time the delivery is sent — its workspace or space is being deleted. The retrieve answers `404` too.
- **The body would pass 4 MB.** No delivery is larger than that, so a server that accepts 4 MB accepts every one; an email too big for it is left out whole rather than cut, and the retrieve returns it. Attachment bytes are never inlined, only their prepared text, so this takes a very long message or a very long document. Check your framework's limit: Express parses 100 KB by default, and serverless platforms cap a request at around 4.5–6 MB.

The email is read when each attempt is sent, not stored with the event. A retry an hour later therefore carries freshly signed URLs — they are good for an hour from the attempt — and the email as it is then, `status` and delivery `events` included. It also means the email is not part of the record: a delivery's `body` under [Inspecting deliveries](#inspecting-deliveries) and an event read from [`GET /events`](#catching-up) show the summary.

Signed URLs open without your API key, so a `full` body is as sensitive as the mail it carries — [verify the signature](webhooks/verifying.md) before trusting it, and keep it out of request logs. An endpoint carried over from the previous aiinbx.com (`format: "v1"`) cannot be `full`.

## Routing

An endpoint can narrow what it receives by sender or recipient — useful when one workspace serves several products or tenants:

```ts
await aiinbx.webhookEndpoints.create({
  url: "https://app.example.com/webhooks/support",
  subscriptions: ["email.received"],
  routing: [
    { effect: "allow", field: "to", pattern: "support@example.com" },
    { effect: "block", field: "from", pattern: "*@spam-source.example" },
  ],
})
```
`field` is `from`, `to`, or `either`. A `from` rule matches the sender and any Reply-To address; a `to` rule matches every recipient — To, Cc and Bcc, and on received mail the addresses of yours it arrived at (`delivered_to`) — so a pattern like `*@acme.dev` follows that company's people whether they were addressed, copied or blind-copied. Two rules decide everything:

1. **Block wins**

    If any `block` rule matches, the event isn't delivered — regardless of what
    the allow rules say.

2. **An empty allow list means no opinion**

    With no `allow` rules, everything not blocked comes through. Add one and the
    endpoint receives only what matches it.

Mailbox events use the mailbox address for both `from` and `to`, so address routing also applies to `mailbox.*` events. Domain events have no addresses to match and bypass address routing; the endpoint must still subscribe to their event type.

Up to 100 rules per endpoint.

## Spaces

Every endpoint receives every [space](guides/spaces.md)'s events, each stamped with its `space_id` on the envelope — `null` for the workspace itself. Branch on it to find the customer:

```ts
if (event.space_id) {
  const customer = await customerBySpace(event.space_id)
  await queueForCustomer(customer, event)
}
```
A customer who needs events delivered to a system of their own gets them from your handler, not from an endpoint of their own. Routing rules apply across spaces the same way.

## Retries

A failed delivery is retried on a fixed schedule, with ±20% jitter, inside a **24-hour window**:

| Attempt | Roughly after the previous |
| ------- | -------------------------- |
| 2       | 1 minute                   |
| 3       | 5 minutes                  |
| 4       | 30 minutes                 |
| 5       | 2 hours                    |
| 6       | 6 hours                    |
| 7       | 24 hours, capped at the delivery deadline |

Each next attempt is capped at 24 hours after the delivery was created, so the final interval may be shorter than the table shows. After the attempt budget is exhausted, or an attempt fails at or after that deadline, the delivery is marked `failed`. Inspect and replay failed deliveries after resolving the cause.

If your endpoint returns a `Retry-After` header, it's honored instead of the standard backoff — the way to say "I'm degraded, come back in ten minutes" without being hammered.

When events to an endpoint stop getting through — one has waited an hour on its retries, or a busy endpoint with 10 or more waiting has not answered once in 15 minutes — the workspace's owners and admins get an email, and another if events start being dropped after their 24 hours of retries: once per outage, not per delivery, and only those who have not turned these mails off under Settings → Notifications. The outage is over once the endpoint has answered for an hour without a failure; everything still waiting is then sent at once, and the mail says how many events were held up and how many were dropped.

## Throughput

Up to `max_concurrency` deliveries are in flight to an endpoint at once — 16 by default, anywhere from 1 to 128. Throughput is that number divided by your handler's latency: at 200 ms a default endpoint takes about 80 events a second, and one set to 128 about 640. A platform fronting thousands of tenants from one endpoint should raise it.

```ts
await aiinbx.webhookEndpoints.update("whk_...", { max_concurrency: 64 })
```
Nothing about order is promised at any setting, including 1 — a retry lands after events that came later, and two events about one thread can arrive in either order. Order by `created_at` on the envelope, or by the data's own timestamps, and never by arrival. Setting it to 1 is a way to go easy on a small handler, not a way to sequence events.

## Inspecting deliveries

```ts TypeScript
const page = await aiinbx.webhookEndpoints.listDeliveries("whk_...", {
  state: "failed",
})

for (const delivery of page.data) {
  console.log(delivery.event_type, delivery.state, delivery.next_attempt_at)

  for (const attempt of delivery.attempts) {
    console.log(" ", attempt.attempted_at, attempt.status_code, attempt.error)
  }
}
```
```python Python
page = client.webhook_endpoints.list_deliveries("whk_...", state="failed")

for delivery in page["data"]:
    print(delivery["event_type"], delivery["state"])
```
```bash curl
curl "https://api.aiinbx.com/api/v2/webhook-endpoints/whk_.../deliveries?state=failed" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Each delivery carries the `body` the endpoint receives and every `attempt` — timestamp, status code, duration, error, and the first 2 KB of your response. That last field is usually what tells you why your handler rejected the request.

`body` is what is stored. For a [`full`](#full-payloads) endpoint that is the envelope without `data.email`, which is read from the message each time a delivery is sent and not kept — the console rebuilds it for the delivery you open. `body` is null in two cases: for an endpoint carried over from the previous aiinbx.com (`format: "v1"`), whose legacy body is built from the message at send time rather than stored — the console rebuilds it for the delivery you open — and for a delivery nothing was sent for, which such an endpoint settles as `failed` with no attempts when the message the event was about no longer exists.

Filter by `state`: `pending`, `delivered`, or `failed`.

### Replaying

After you've fixed the handler, push the failures back through:

```ts
const { data } = await aiinbx.webhookEndpoints.retryDeliveries("whk_...", {
  delivery_ids: failed.map((d) => d.id),
})
```
Up to 100 per call. A replay carries the same event `id` as the original, so a handler that deduplicates on it stays correct.

## Catching up

A replay needs a delivery to replay. An endpoint that was paused, not yet subscribed to a type, or not yet created has none — but the event is on record all the same. Every event is recorded when it happens, whether or not anything was listening, and [`GET /events`](https://docs.aiinbx.com/api/endpoints/events/list-events.md) reads the record back: the same envelopes with the same `id`, for the whole workspace.

To catch up, walk forward from the last event you processed:

```ts TypeScript
let after = await loadCheckpoint() // the last created_at you processed

for await (const event of aiinbx.events.list({
  order: "asc",
  after: overlap(after),
  type: ["email.delivered", "email.bounced", "email.complained", "email.failed"],
})) {
  await enqueueOnce(event.id, event) // the same dedupe your webhook handler uses
  after = event.created_at
}

await saveCheckpoint(after)
```
```python Python
after = load_checkpoint()  # the last created_at you processed

for event in client.events.iter(
    order="asc",
    after=overlap(after),
    type=["email.delivered", "email.bounced", "email.complained", "email.failed"],
):
    enqueue_once(event["id"], event)
    after = event["created_at"]

save_checkpoint(after)
```
```bash curl
curl -G https://api.aiinbx.com/api/v2/events \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -d order=asc \
  --data-urlencode "after=2026-09-01T10:30:00Z" \
  -d type=email.bounced -d type=email.failed
```
`order=asc` lists oldest first, `after` leaves out everything up to and including that instant, and `next_cursor` continues the walk in the same order until it is `null` — you have caught up. `before` closes the window from the other side; the [same three parameters](reference/conventions.md#syncing) sync the email list. `type` repeats for any of several; `email_id` narrows to what happened to one email — all but `email.unsubscribed`, which is about the address that left; `space` to one [space](guides/spaces.md).

`overlap` is yours to write, and it should step back a minute. `created_at` is when the event happened, and an event recorded inside a longer write — a send, an inbound message being filed — becomes readable when that write commits, which can be after a later event already was. Reading a minute you have already seen costs nothing when `enqueueOnce` deduplicates by `event.id`, and it is what makes the checkpoint safe.

Three things differ from a delivery:

- `data` is always the summary. The `data.email` an endpoint with `payload: "full"` is sent is read when the delivery goes out and is not kept; retrieve the email by its `email_id`.
- Routing rules belong to endpoints, so nothing is filtered by address here.
- [Test events](#testing-an-endpoint) are not listed: nothing happened.

One event by id is [`GET /events/{event_id}`](https://docs.aiinbx.com/api/endpoints/events/retrieve-event.md) — the `evt_…` from the envelope, the `AIInbx-Event-ID` header, or a delivery's `event_id`. Any key can read events; `read` [scope](https://docs.aiinbx.com/authentication.md#scopes) is enough.

Used on its own, the same loop on a timer is a way to take events without exposing an endpoint at all. To have each event pushed as it happens instead, [follow the list live](reference/conventions.md#live).

## Testing an endpoint

Fire a synthetic event without waiting for real mail — the fastest way to check a new handler or a tunnel:

```ts TypeScript
const result = await aiinbx.webhookEndpoints.test("whk_...", {
  event_type: "email.received",
  resource_id: "eml_test",
  payload: { thread_id: "thr_test", subject: "Test", from: "test@example.com" },
})

console.log(result.state, result.status_code, result.error, result.response)
```
```python Python
result = client.webhook_endpoints.test("whk_...", {"event_type": "email.received"})
print(result["state"], result["status_code"])
```
```bash curl
curl https://api.aiinbx.com/api/v2/webhook-endpoints/whk_.../test \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "event_type": "email.received" }'
```
The response tells you what your endpoint actually did — status code, duration, and the body it returned. A test event is signed exactly like a real one, so it exercises your verification code too.

> [!NOTE]
> `payload` is whatever you put in it. A test event's `data` won't match the real shape unless you make it, which is deliberate — it lets you probe how your handler behaves on unexpected input.

A test event is about no message. On a [`full`](#full-payloads) endpoint, the `data.email` of an `email.received` or `email.sent` test is the `email` you put in `payload`, or a sample email when you put none — so a test shows the whole shape either way.

## Rotating the secret

The secret is shown once, when the endpoint is created, and never read back. Rotation is how you get another — because it was lost, because it leaked, or on a schedule. It issues a new secret while the old one keeps verifying for a grace period, so you can deploy without dropping events:

```ts
const { secret, previous_secret_expires_at } =
  await aiinbx.webhookEndpoints.rotateSecret("whk_...", {
    grace_seconds: 86_400,
  })
```
The new `secret` is in the response, once, like the first. During the grace period every delivery is signed with both secrets — two `v1=` values in `AIInbx-Signature` — so a verifier holding either one passes, and the endpoint's `previous_secret_expires_at` says when that ends. `grace_seconds` defaults to 24 hours and can be up to 7 days. Set it to `0` to cut over immediately — only when a secret has leaked and the gap matters less than the exposure.

1. **Rotate with a grace period**

    Both secrets verify.

2. **Deploy the new secret**

    Your handler switches over.

3. **Let the grace period lapse**

    The old secret stops working on its own.

## Pausing an endpoint

```ts
await aiinbx.webhookEndpoints.update("whk_...", { enabled: false })
```
A disabled endpoint receives nothing and accrues nothing — events that occur while it's off are not queued for later. Prefer it to deleting when you're taking a service down for maintenance, and [catch up](#catching-up) from the events API when it is back.

## Endpoint URLs

Endpoint URLs must be publicly reachable over HTTPS. Requests to private and loopback addresses are refused, so a URL that resolves inward can't be used to reach into infrastructure. Redirects are not followed: a `3xx` answer is a failed delivery, retried like any other.

In development, run [`aiinbx listen`](https://docs.aiinbx.com/cli.md#listen-webhooks-on-localhost) instead. It forwards your workspace's events to `localhost`, signed with the same scheme, with no endpoint to register and no tunnel.
