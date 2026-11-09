# Threads

Read a conversation, page through a long one, and reply or forward within a thread without reconstructing In-Reply-To or References headers.

From https://docs.aiinbx.com/guides/threads

A thread is the conversation, not a folder you put messages in. Every send opens one or joins one, every inbound reply is matched onto one, and `thread_id` is the handle your code should hold onto.

## Listing threads

```ts TypeScript
const page = await aiinbx.threads.list({ limit: 50, query: "invoice" })

for (const thread of page.data) {
  console.log(thread.subject, thread.message_count, thread.last_message_at)
}
```
```python Python
page = client.threads.list(limit=50, query="invoice")

for thread in page["data"]:
    print(thread["subject"], thread["message_count"], thread["last_message_at"])
```
```bash curl
curl "https://api.aiinbx.com/api/v2/threads?limit=50&query=invoice" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Every word of `query` must appear in the subject, the mailbox or a participant's address; `mailbox` narrows to one address. Both SDKs also expose the list as an iterator that keeps your filters across every page:

```ts TypeScript
for await (const thread of aiinbx.threads.list({
  mailbox: "team@example.com",
})) {
  console.log(thread.subject)
}
```
```python Python
for thread in client.threads.iter(mailbox="team@example.com"):
    print(thread["subject"])
```
`query` is a literal subject/address filter for finding an operational thread, not a knowledge-base search. Persist any email data your product needs long-term in your own database.

## Retrieving one

`threads.retrieve` returns the conversation with its messages inline, each a full email: bodies, headers, attachments, delivery events, and engagements.

```ts TypeScript
const thread = await aiinbx.threads.retrieve("thr_...")

console.log(thread.participants, thread.message_count)

for (const message of thread.messages) {
  console.log(message.direction, message.from.address, message.text)
}
```
```python Python
thread = client.threads.retrieve("thr_...")

for message in thread["messages"]:
    print(message["direction"], message["from"]["address"], message["text"])
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `id` | `string` | - | thr_… |
| `subject` | `string` | - | The conversation's subject. |
| `mailbox` | `string` | - | The address on your side of the conversation. |
| `participants` | `string[]` | - | Every address that has appeared on the thread. |
| `message_count` | `number` | - | Total messages, not just the page. |
| `last_message_at` | `string` | - | Timestamp of the newest message. |
| `messages` | `FullEmail[]` | - | Messages, oldest first. |
| `messages_next_cursor` | `string \| null` | - | Pass as message_cursor to fetch the next page of messages. |

### Long threads

Messages paginate independently of the thread list — `message_limit` (default 40, max 100) and `message_cursor`. Both SDKs wrap that in an iterator:

```ts TypeScript
for await (const message of aiinbx.threads.iterateMessages("thr_...", {
  message_limit: 100,
})) {
  console.log(message.subject)
}
```
```python Python
for message in client.threads.iter_messages("thr_..."):
    print(message["snippet"])
```
## Replying

This is the call the API exists for. Give it the thread and the content; the sender, recipients, subject, and the `In-Reply-To` and `References` headers are derived from the conversation:

```ts TypeScript
await aiinbx.threads.reply(
  "thr_...",
  { text: "Sounds good — see you Thursday." },
  { idempotencyKey: "reply-thr-123-v1" }
)
```
```python Python
client.threads.reply(
    "thr_...",
    {"text": "Sounds good — see you Thursday."},
    idempotency_key="reply-thr-123-v1",
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/threads/thr_.../reply \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: reply-thr-123-v1" \
  -d '{ "text": "Sounds good — see you Thursday." }'
```
As with a send, at least one of `html` or `text` is required, and everything else is optional. The reply answers the thread's latest email and goes where a mail client would send it: to that email's Reply-To when it sets one — a helpdesk ticket address, a mailing list, the person behind a form — else to its sender, or, when the email is yours, to whom it was sent.

When the Reply-To names only your own addresses — a receipt that asks answers to go to your support desk, say — the reply has nobody to go to and fails with [`no_recipients`](https://docs.aiinbx.com/reference/errors.md#no_recipients). AI Inbx does not guess another recipient in its place; pass `to`.

### Answering a specific email

Pass `email_id` to answer one email rather than whatever is latest when the call lands. Recipients, subject, sender and `In-Reply-To` all come from that email, and a draft or scheduled reply keeps answering it when it goes out. An agent working from a webhook should pass the id it was handed, so an out-of-office or a colleague's message arriving while it writes cannot change whom it answers:

```ts
if (event.type === "email.received") {
  await aiinbx.threads.reply(
    event.data.thread_id,
    { email_id: event.data.email_id, text: await draftAnswer(event) },
    { idempotencyKey: `reply-${event.data.email_id}` }
  )
}
```
An `Idempotency-Key` on a reply is bound to the request, not to the recipients it worked out, so a retry made after the thread has moved on replays the first reply.

To see whom a reply would go to before anything leaves, pass `draft: true`. The [draft](https://docs.aiinbx.com/guides/drafts.md) holds the recipients, subject and `in_reply_to` as worked out, and `emails.sendDraft` sends it unchanged.

A reply needs a thread, so the message being answered has to be one AI Inbx holds. To answer one it has never seen — mail from before a migration, or from another provider — [send with `in_reply_to` and `references`](https://docs.aiinbx.com/guides/sending.md#replying-to-mail-ai-inbx-hasnt-seen) instead.

### Overriding what's inferred

Each inferred field can be set explicitly when the default isn't what you want:

| Prop | Type | Default | Description |
| - | - | - | - |
| `email_id?` | `string` | - | The email in the thread to answer. Defaults to the thread's latest. |
| `reply_all?` | `boolean` | `false` | Reply all, as a mail client does: the answered email's Reply-To (else its sender) and `to` go in `to`, its `cc` in `cc`. With a Reply-To set, the sender is left out. Only the address you reply as and its mailbox's aliases are left off; colleagues stay. When the email is yours, its `to` and `cc` again. |
| `to?` | `string \| string[]` | - | Replace the inferred recipients. |
| `cc?` | `string \| string[]` | - | Carbon copies for this reply, added to the ones `reply_all` brings. |
| `bcc?` | `string \| string[]` | - | Blind copies for this reply. |
| `from?` | `string \| object` | - | Send as a different address on the thread — must still be a verified domain or connected mailbox. |
| `reply_to?` | `string \| string[]` | - | Where answers to this reply should go. |
| `subject?` | `string` | - | Override the inherited subject. |
| `attachments?` | `object[]` | - | Same shape as on a send — up to 20, 3 MB in total. |
| `headers?` | `object` | - | Additional RFC 5322 headers. |
| `scheduled_at?` | `string` | - | Defer the reply; up to 30 days out. |
| `suppression_key?` | `string` | - | List this reply is checked against. |
| `unsubscribe?` | `boolean` | - | Mark the reply as optional mail. |
| `pacing?` | `object` | - | { skip, count } — bypass or discount pacing rules. |
| `tracking?` | `object` | - | { opens, clicks } — override the sending domain's tracking for this email. |

```ts
await aiinbx.threads.reply("thr_...", {
  reply_all: true,
  cc: "manager@example.com",
  text: "Looping in Dana.",
})
```
The reply returns the same shape a send does — including `suppressed` and `pacing` — so the checks from [Sending](https://docs.aiinbx.com/guides/sending.md#reading-the-response) apply here too.

## Forwarding

`threads.forward` sends a whole thread to addresses that were not on it. Pass the thread and a recipient; the API renders the conversation into one email and sends it:

```ts TypeScript
await aiinbx.threads.forward(
  "thr_...",
  { to: "dana@example.com", note: "See the thread below." },
  { idempotencyKey: "forward-thr-123-v1" }
)
```
```python Python
client.threads.forward(
    "thr_...",
    {"to": "dana@example.com", "note": "See the thread below."},
    idempotency_key="forward-thr-123-v1",
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/threads/thr_.../forward \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: forward-thr-123-v1" \
  -d '{ "to": "dana@example.com", "note": "See the thread below." }'
```
Only `to` is required. The recipient gets one email: the note, if any, then every message in the thread, oldest first — who wrote it, when, to whom, and what they wrote. Each turn is the author's own text; quoted tails and repeated signatures are removed, so a ten-message thread reads as ten messages rather than ten copies of the chain. The thread's attachments ride along in order, up to about 15 MB in total; anything past that is named in the transcript and stays on the thread, where the API serves it.

| Prop | Type | Default | Description |
| - | - | - | - |
| `to` | `string \| string[]` | - | Recipients of the forward. |
| `note?` | `string` | - | Text placed above the transcript. Up to 10,000 characters. |
| `from?` | `string \| object` | - | The thread's own mailbox unless set; must be a verified domain or connected mailbox. |
| `subject?` | `string` | - | `Fwd:` and the thread's subject unless set. |
| `include_attachments?` | `boolean` | `true` | Whether the thread's files ride along. They are listed in the transcript either way. |
| `cc?` | `string \| string[]` | - | Carbon copies. |
| `bcc?` | `string \| string[]` | - | Blind copies. |
| `reply_to?` | `string \| string[]` | - | Where replies to the forward should go. Defaults to `from`. |
| `attachments?` | `object[]` | - | Files of your own, on top of the thread's — same shape as on a send. |
| `headers?` | `object` | - | Additional RFC 5322 headers. |
| `scheduled_at?` | `string` | - | Defer the forward; up to 30 days out. The transcript is rendered at request time. |
| `suppression_key?` | `string` | - | List this forward is checked against. |
| `pacing?` | `object` | - | { skip, count } — bypass or discount pacing rules. |
| `tracking?` | `object` | - | { opens, clicks } — override the sending domain's tracking for this email. |

### The forward is its own thread

A forward opens a new thread rather than joining the one it carries; the response's `thread_id` is that new thread. Its participants are the forward's recipients, not the original thread's, so the two stay apart: `threads.reply` on the original still infers the original participants, and a reply to the forward lands on the forward thread.

The two are linked. The new thread's `forward_of` names the original — on `threads.retrieve`, `threads.list`, and the [`thread.created`](https://docs.aiinbx.com/webhooks/events/thread-created.md) event it fires. A reply to the forward arrives as an [`email.received`](https://docs.aiinbx.com/webhooks/events/email-received.md) on a thread whose `forward_of` is set:

```ts
const thread = await aiinbx.threads.retrieve(event.data.thread_id)

if (thread.forward_of) {
  // A reply to a forward. thread.forward_of is the thread it carried.
}
```
The forward returns the same shape a send does, and suppression, pacing, scheduling, and idempotency behave exactly as on one. An idempotency key on a forward is bound to the request — recipients, note, options — rather than to the transcript it rendered, so a retry made after the thread has grown still replays the first forward.

## Automated replies

Use a durable worker to read the conversation, decide whether to respond, and send a reply with the `email_id` it answers and a stable idempotency key. Thread retrieval returns one page of messages; use `iterateMessages` or `iter_messages` when you need the full available conversation.

Classification alone does not make an agent safe to run unattended. Account for duplicate and out-of-order events, concurrent replies, and human handoff. The [email agent guide](https://docs.aiinbx.com/integrations/agents.md) describes the workflow and the application functions it requires.

## Next

**[Receiving](receiving.md)**

Handle new messages as they arrive.

**[Attachments](attachments.md)**

Reading what came in as a PDF.
