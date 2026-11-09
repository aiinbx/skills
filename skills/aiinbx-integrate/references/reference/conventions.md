# Conventions

The rules that hold across every AI Inbx endpoint: prefixed IDs, cursor pagination, timestamps, request IDs, and idempotent retries.

From https://docs.aiinbx.com/reference/conventions

Things that are true everywhere in the API, gathered so the guides don't repeat them.

## IDs

Every resource has a prefixed ID: a short type tag, an underscore, and 32 hex characters.

```
eml_9f2c4a1b8e7d6c5b4a3f2e1d0c9b8a76
```
| Prefix  | Resource         |
| ------- | ---------------- |
| `key_`  | API key          |
| `spc_`  | Space            |
| `inb_`  | Inbox            |
| `dom_`  | Domain           |
| `dns_`  | DNS record       |
| `mbx_`  | Mailbox          |
| `app_`  | OAuth app        |
| `thr_`  | Thread           |
| `eml_`  | Email            |
| `att_`  | Attachment       |
| `whk_`  | Webhook endpoint |
| `evt_`  | Event            |
| `dlv_`  | Webhook delivery |
| `atp_`  | Delivery attempt |
| `sup_`  | Suppression      |
| `pace_` | Pacing rule      |

The prefix is part of the ID, not decoration — pass `eml_9f2c…` verbatim, never the bare hex. An ID of the wrong type never silently matches another resource: in the path it is a `404` [`not_found`](errors.md#not_found), in a body or query a `400`.

Treat IDs as opaque strings. They're stable and safe to store.

## Pagination

List endpoints are cursor-paginated and answer the same shape:

```json
{
  "data": [],
  "next_cursor": "eml_..."
}
```
| Parameter | Default | Range                                          |
| --------- | ------- | ---------------------------------------------- |
| `limit`   | 40      | 1–100                                          |
| `cursor`  | —       | An opaque cursor from a previous `next_cursor` |

`next_cursor` is `null` on the last page. Four short lists come whole, with no cursor: workspaces, members, invitations, and the pacing queue, which answers its first `limit` held emails and a `total`.

Paging manually:

```ts
let cursor: string | undefined

do {
  const page = await aiinbx.emails.list({ limit: 100, cursor })
  await process(page.data)
  cursor = page.next_cursor ?? undefined
} while (cursor)
```
Both SDKs wrap that up. A list call is awaitable for one page, iterable for all of them, and keeps your filters on every request:

```ts TypeScript
// One page.
const page = await aiinbx.threads.list({ limit: 50 })

// Every result, transparently paginated.
for await (const thread of aiinbx.threads.list({
  mailbox: "team@example.com",
})) {
  console.log(thread.subject)
}

// A small collection, in full.
const domains = await aiinbx.domains.list().all()

// Page by page, when you want to checkpoint.
for await (const page of aiinbx.emails.list().iterPages()) {
  await saveCheckpoint(page.next_cursor)
}
```
```python Python
# One page.
page = client.threads.list(limit=50)

# Every result, transparently paginated.
for thread in client.threads.iter(mailbox="team@example.com"):
    print(thread["subject"])

# Async works the same way.
async for thread in async_client.threads.iter():
    print(thread["subject"])
```
A cursor from one endpoint is meaningless on another, and a malformed one is rejected with [`invalid_cursor`](errors.md#invalid_cursor) rather than silently starting from the top.

> [!NOTE]
> A thread's messages paginate separately from the thread list, with `message_limit` and `message_cursor`.

### Syncing a list [#syncing]

Emails and [events](../webhooks.md#catching-up) are things that happened, so their lists also take a window of time and a direction:

| Parameter | Default | Meaning                                                        |
| --------- | ------- | -------------------------------------------------------------- |
| `after`   | —       | Only items with `created_at` later than this instant           |
| `before`  | —       | Only items with `created_at` earlier than this instant         |
| `order`   | `desc`  | `desc` is newest first; `asc` is oldest first, by `created_at` |

Both bounds are exclusive and take an ISO 8601 timestamp with an offset — `2026-09-01T10:30:00Z`. A cursor is a position, not a direction, so send the same `order` and filters with it.

A sync reads forward from where it stopped, rather than paging back from the newest item until it meets something it knows:

```ts TypeScript
let after = await loadCheckpoint() // the last created_at you stored

for await (const email of aiinbx.emails.list({
  order: "asc",
  after: overlap(after),
  direction: "inbound",
})) {
  await upsertEmail(email) // keyed by email.id
  after = email.created_at
}

await saveCheckpoint(after)
```
```python Python
after = load_checkpoint()  # the last created_at you stored

for email in client.emails.iter(order="asc", after=overlap(after), direction="inbound"):
    upsert_email(email)  # keyed by email["id"]
    after = email["created_at"]

save_checkpoint(after)
```
Have `overlap` step back a minute and write by id. `created_at` is stamped when a write begins and the item is readable when it commits, so one item can appear just behind another you have already read; re-reading the last minute picks it up, and the id makes the repeat harmless.

Threads take no window: they sort by their latest message, which moves as mail arrives. Sync emails and group them by `thread_id`.

### Following a list live [#live]

`GET /emails` and `GET /events` asked for `Accept: text/event-stream` answer with a [server-sent event](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) stream instead of a page. Each item added to the list goes out once, oldest first, as a message whose `data` is the item as the list returns it and whose `id` is where you stand once you have it. A message with an `id` and no `data` moves you past events that added nothing to your list, and a comment every 20 seconds keeps a quiet connection open.

The list's filters apply — `direction`, `thread_id`, `query`, `space` on emails; `type`, `email_id`, `space` on events. `after` says where to start; without it the stream starts now. `cursor`, `before`, and `status` are refused with `400` [`invalid_query`](errors.md#invalid_query), and `limit` and `order` do not apply. An email is sent as it is when the stream reads it, so its status may already have moved on — follow events for what happens to it next.

A connection lasts about four minutes and then ends. Connect again with `Last-Event-ID` set to the last `id` you read, and the stream resumes exactly there, so nothing is missed between connections. A workspace can follow up to 25 lists at once; one more is answered `429` [`too_many_requests`](errors.md#too_many_requests).

The TypeScript SDK does all of this in `follow`, reconnecting until you stop it:

```ts
const controller = new AbortController()

for await (const event of aiinbx.events.follow(
  { type: ["email.received"] },
  { since: new Date(Date.now() - 60_000), signal: controller.signal }
)) {
  await handle(event)
}
```
`aiinbx.emails.follow` takes the email list's filters the same way. On the command line, [`aiinbx listen`](https://docs.aiinbx.com/cli.md#listen-webhooks-on-localhost) and [`aiinbx wait`](https://docs.aiinbx.com/cli.md#wait-block-until-an-email-arrives) are built on it.

## Spaces

Every resource that can live in a [space](../guides/spaces.md) carries `space_id`, and it is never omitted: a `spc_…` id, or `null`.

`null` is the workspace itself, not "unassigned". A resource created without a space belongs to the workspace and stays there — there is no move. A workspace that never creates a space reads `null` everywhere and can ignore the field.

Mail inherits rather than declares: an email's `space_id` is that of the domain or mailbox it went through, and a thread's is its messages'. Nothing on a send names one.

Lists of resources that live in a space take `space`:

| Parameter | Default | Range                                                            |
| --------- | ------- | ---------------------------------------------------------------- |
| `space`   | —       | A `spc_…` id, or `none` for only resources with `space_id: null` |

Omitted, the list includes the workspace itself and every space. Pass `space=none` for the workspace's own resources only. Naming a space that is not the workspace's is `404`.

## Timestamps

Every timestamp the API returns is RFC 3339 in UTC:

```
2026-09-01T10:31:04Z
```
Timestamps you send — `scheduled_at` — must carry an offset. `2026-09-14T09:00:00Z` and `2026-09-14T11:00:00+02:00` are both fine; `2026-09-14T09:00:00` is rejected, because a wall-clock time without a zone isn't an instant.

Durations carry their unit in the name: `per_seconds`, `grace_seconds`, `open_early_minutes`, `backfill_days`, `duration_ms`.

## Request IDs

Every response carries `X-Request-ID`, and every error body repeats it as `request_id`. Quote it when you report a problem — it's how a specific request gets found.

Send your own and it's echoed back instead of a generated one, which is what you want when you already have a trace ID:

```bash
curl https://api.aiinbx.com/api/v2/emails \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "X-Request-ID: trace-8f2c4a1b"
```
Up to 128 characters of `A-Za-z0-9._:-`; anything else is ignored in favor of a generated ID.

Both SDKs surface it without unwrapping the response:

```ts TypeScript
const { data, response, requestId } = await aiinbx.emails
  .retrieve("eml_...")
  .withResponse()
```
```python Python
response = client.with_raw_response.emails.retrieve("eml_...")
print(response.request_id)

# Or, after any call:
print(client.last_request_id)
```
## Idempotency

`POST /emails`, `POST /threads/{id}/reply`, and `POST /threads/{id}/forward` accept an `Idempotency-Key` header. It makes a retry safe: the same key with the same body returns the original result instead of sending twice.

```ts TypeScript
await aiinbx.emails.send(payload, { idempotencyKey: "order-8812-receipt" })
```
```python Python
client.emails.send(payload, idempotency_key="order-8812-receipt")
```
```bash curl
curl https://api.aiinbx.com/api/v2/emails \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Idempotency-Key: order-8812-receipt" \
  -H "Content-Type: application/json" \
  -d @email.json
```
| Case                        | Result                                                                     |
| --------------------------- | -------------------------------------------------------------------------- |
| First request with this key | `201`, an email is created; inspect its status                             |
| Same key, same body         | `200` with `Idempotent-Replayed: true` and the existing email              |
| Same key, different body    | `409` [`idempotency_key_reused`](errors.md#idempotency_key_reused) |

Keys are 1–255 characters and scoped to the workspace and the resolved space, including `null` for workspace-level mail. Use a different key for each logical send, reply, or forward within that scope.

A replay returns the existing email without sending again. It does not repeat the original suppression or pacing evaluation: the replay’s `suppressed` is empty and `pacing` is null. Persist the first response if you need those original decisions. The key is stored with the email; do not rely on it as a permanent deduplication record after that email is deleted.

**Reuse one key for every attempt of the same operation.** Derive it from a stable application identifier, such as `order-8812-receipt`, or generate a UUID once and persist it. Generating a new key on every retry creates separate operations. When you're answering an email from a webhook, its `email_id` is already the stable thing:

```ts
await aiinbx.threads.reply(
  threadId,
  { text },
  { idempotencyKey: `reply-${event.data.email_id}` }
)
```
## Requests and responses

- **Request bodies are JSON.** For operations that accept a JSON body, send `Content-Type: application/json`; another content type returns `415` [`unsupported_media_type`](errors.md#unsupported_media_type). Bodyless operations, such as canceling an email, do not require a JSON body.
- **`PATCH` is a partial update.** Send only the fields you're changing; at least one is required.
- **Empty successes return `204`** with no body — most deletions and revocations. Disconnecting a mailbox answers `200` with the mailbox, now `disconnected`.
- **Redirects are real.** Attachment downloads answer `307` with a `Location` header; follow it (`curl -L`). Both SDKs do this for you.
- **Nothing is cached.** Every response carries `Cache-Control: private, no-store`.
- **Unknown fields are rejected**, not ignored. A typo in a request body fails validation with an `issues` array naming the field — much better than a message that silently didn't do what you meant.

## Errors

Every failure is `application/problem+json` ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) with a machine-readable `code`. Full list and handling patterns in [Errors](errors.md).

```json
{
  "type": "https://docs.aiinbx.com/problems/invalid_request",
  "title": "Unprocessable Content",
  "status": 422,
  "detail": "Request body is invalid",
  "code": "invalid_request",
  "request_id": "...",
  "issues": [{ "path": "to.0", "message": "Invalid email address" }]
}
```
## Limits

Worth knowing before you hit them:

|                                                      | Limit                 |
| ---------------------------------------------------- | --------------------- |
| Recipients per field (`to`, `cc`, `bcc`, `reply_to`) | 100                   |
| Attachments per message                              | 20, 3 MB in total     |
| `html` or `text` body                                | 2,000,000 characters each |
| Subject                                              | 998 characters        |
| Header name / value                                  | 200 / 8000 characters |
| `scheduled_at` horizon                               | 30 days               |
| Page size                                            | 100                   |
| Addresses per suppression call                       | 1000                  |
| Email IDs per pacing release                         | 500                   |
| Delivery IDs per webhook retry                       | 100                   |
| Webhook routing rules per endpoint                   | 100                   |
| Webhook handler response time                        | 10 seconds            |

## Request budgets

Every workspace has a request budget, and every request is admitted against it before it runs. A budget is counted in **cost units** rather than requests, because a search over a workspace's mail costs a few hundred times what a read by id does:

| Class       | Units | Operations                                                                                                                |
| ----------- | ----- | ------------------------------------------------------------------------------------------------------------------------- |
| `read`      | 1     | Reads by id, lists without a search term, small writes                                                                    |
| `send`      | 2     | Sending, replying, forwarding                                                                                             |
| `query`     | 5     | Lists with `query`, suppression batches, pacing queue reads and releases, delivery retries                                |
| `provision` | 10    | Creating, verifying and diagnosing domains, mailbox syncs, webhook tests — anything that talks to DNS, a mail provider or your server |

The default budget refills at 50 units per second and holds 10 seconds of that — 500 units — for a burst. At most 32 units may be in flight at once. A request that names a space (`?space=`, `/spaces/{id}`) is held to half the workspace's budget besides, so one busy customer leaves the rest of the platform its share. Larger budgets are agreed per workspace; ask.

Every answer to an admitted request says where the budget stands — the ones that worked and the ones that failed alike, so an error is as good to pace from as a `200`:

```http
RateLimit-Limit: 500
RateLimit-Remaining: 497
RateLimit-Reset: 1
```
`RateLimit-Remaining` is in units, and `RateLimit-Reset` is the seconds until the budget is full again. Past the budget the answer is `429` with [`too_many_requests`](errors.md#too_many_requests) and `Retry-After` in seconds; both SDKs wait that long and retry on their own. A `429` with [`rate_limited`](errors.md#rate_limited) is different — an upstream mail provider throttling — and carries `Retry-After` the same way.

The platform also has a ceiling on requests in flight across every workspace. A request refused under it is a `429` too, with a `Retry-After` of a second: the budget was yours to spend, the capacity was not there to spend it on.
