# Sending

Addresses, HTML and text bodies, attachments, custom headers, idempotency, and what the send response tells you.

From https://docs.aiinbx.com/guides/sending

`POST /emails` takes a message and returns it with an ID, a thread, and a status. Everything else on this page is a detail of that one call.

```ts TypeScript
const email = await aiinbx.emails.send({
  from: { name: "Ada", address: "ada@example.com" },
  to: "grace@example.com",
  subject: "Quick question",
  text: "Does Thursday still work for the review?",
})
```
```python Python
email = client.emails.send(
    {
        "from_": {"name": "Ada", "address": "ada@example.com"},
        "to": ["grace@example.com"],
        "subject": "Quick question",
        "text": "Does Thursday still work for the review?",
    }
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/emails \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Ada <ada@example.com>",
    "to": "grace@example.com",
    "subject": "Quick question",
    "text": "Does Thursday still work for the review?"
  }'
```
> [!NOTE]
> Sending requires a key with the `sending` or `full` [scope](https://docs.aiinbx.com/authentication.md#scopes), and a `from` address on a [verified domain](domains.md) or a [connected mailbox](mailboxes.md). A connection made an [inbox](https://docs.aiinbx.com/guides/agent-inbox.md) may leave `from` out: it sends as its inbox.

## Addresses

`from` takes either a formatted string or an object — pick whichever your code already has:

```json
{ "from": "Ada Lovelace <ada@example.com>" }
{ "from": { "name": "Ada Lovelace", "address": "ada@example.com" } }
{ "from": { "address": "ada@example.com" } }
```
A display name is 1–200 characters; surrounding quotes are stripped.

`to`, `cc`, `bcc`, and `reply_to` each take a single address or an array, so the single-recipient case stays terse:

```json
{ "to": "grace@example.com" }
{ "to": ["grace@example.com", "charles@example.com"], "cc": "ops@example.com" }
```
Each field holds at most 100 addresses, and addresses are lowercased. A send with an empty `to` fails with `422` [`invalid_request`](../reference/errors.md#invalid_request); one whose every `to` address is suppressed fails with [`recipients_suppressed`](../reference/errors.md#recipients_suppressed).

## Content

`subject` is required (up to 998 characters, the RFC line limit). At least one of `html` or `text` must be present; sending both is the right default, because it gives every client something to render:

```ts
await aiinbx.emails.send({
  from: "Ada <ada@example.com>",
  to: "grace@example.com",
  subject: "Your invoice",
  html: "<p>Your invoice is <a href='https://example.com/i/42'>ready</a>.</p>",
  text: "Your invoice is ready: https://example.com/i/42",
})
```
`html` and `text` may each be up to 2,000,000 characters.

## Attachments

Up to 20 attachments per message, together at most 3 MB once decoded, each as base64 `content` with a `filename` and `content_type`:

```ts TypeScript
await aiinbx.emails.send({
  from: "Ada <ada@example.com>",
  to: "grace@example.com",
  subject: "The report",
  text: "Attached.",
  attachments: [
    {
      filename: "report.pdf",
      content_type: "application/pdf",
      content: pdfBuffer.toString("base64"),
    },
  ],
})
```
```python Python
import base64

client.emails.send(
    {
        "from_": "Ada <ada@example.com>",
        "to": ["grace@example.com"],
        "subject": "The report",
        "text": "Attached.",
        "attachments": [
            {
                "filename": "report.pdf",
                "content_type": "application/pdf",
                "content": base64.b64encode(pdf_bytes).decode(),
            }
        ],
    }
)
```
Set `cid` to reference an attachment from your HTML instead of listing it at the bottom of the message:

```json
{
  "html": "<img src=\"cid:logo\" alt=\"Acme\">",
  "attachments": [
    {
      "filename": "logo.png",
      "content_type": "image/png",
      "cid": "logo",
      "content": "iVBORw0KG..."
    }
  ]
}
```
More than 20 fails with `422` [`invalid_request`](../reference/errors.md#invalid_request); more than 3 MB in total fails with `413` [`attachments_too_large`](../reference/errors.md#attachments_too_large). Inbound attachments work differently — see [Attachments](https://docs.aiinbx.com/guides/attachments.md), which also covers the prepared-text extraction that turns a received PDF into Markdown.

## Custom headers

`headers` sets additional RFC 5322 headers — a campaign tag, a correlation ID, anything your downstream tooling reads:

```json
{
  "headers": {
    "X-Campaign-ID": "spring-2026",
    "X-Correlation-ID": "ord_8812"
  }
}
```
Names are limited to 200 characters and values to 8000. `Message-ID`, `In-Reply-To` and `References` are the API's to write — from the thread, or from [`in_reply_to` and `references`](#replying-to-mail-ai-inbx-hasnt-seen) — and a send, reply or forward that sets one here fails with `422` [`invalid_request`](../reference/errors.md#invalid_request).

## Idempotency

Network timeouts are the reason double-sends happen. Pass an `Idempotency-Key` and a retry with the same body returns the original email instead of sending a second one:

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
A replay returns `200` with `Idempotent-Replayed: true`, where the original returned `201`. Reusing a key with a *different* body is a mistake rather than a retry, and fails with [`idempotency_key_reused`](../reference/errors.md#idempotency_key_reused). Derive keys from something stable in your domain — `order-8812-receipt`, not a fresh UUID per attempt. Full semantics in [Conventions](../reference/conventions.md#idempotency).

## Reading the response

The send response is the email plus two fields worth checking on every call:

| Prop | Type | Default | Description |
| - | - | - | - |
| `id` | `string` | - | The email's ID (eml_…). |
| `thread_id` | `string` | - | The thread this message opened or joined. |
| `status` | `string` | - | queued, sending, sent, or scheduled at this point — delivery outcomes arrive later, by webhook. |
| `suppressed` | `string[]` | - | Recipients dropped before sending because they're on a suppression list. Not an error — read it. |
| `pacing` | `object \| null` | - | Non-null when a pacing rule is holding the message: held_by names the rule, estimated_send_at is the projection. |

```ts
const email = await aiinbx.emails.send(payload)

if (email.suppressed.length) {
  console.warn("dropped:", email.suppressed)
}

if (email.pacing && email.pacing.held_by.kind !== "ready") {
  console.log("held until", email.pacing?.estimated_send_at)
}
```
`status` describes the send lifecycle, and moves on to `delivered`, `bounced`, `complained` or `failed` as the outcome comes in. Each outcome also arrives as a [webhook](../webhooks/events.md) and in the email’s `events` array when you retrieve it. A complaint can arrive after delivery, and a soft bounce leaves `status` as it was.

### Bypassing pacing

A password reset shouldn't wait behind a marketing queue. `pacing.skip` sends immediately; `pacing.count` controls whether the send still consumes rule budget:

```json
{ "pacing": { "skip": true, "count": false } }
```
See [Pacing](pacing.md) for what the rules do in the first place.

## Suppression and unsubscribe

`suppression_key` names the list this send is checked against — a campaign, a product, a tenant. Omit it and the send is checked against the workspace’s `*` list and, when sending from a space, that space’s `*` list. A named key adds the matching lists at both levels.

`unsubscribe: true` marks the message as optional mail: it adds one-click unsubscribe headers, and lets a recipient opt out of *this* list without blocking your transactional mail. Place the link yourself with `{{unsubscribe_url}}` in the body, or one is appended at the end. The link is personal, so such a send takes exactly one recipient across `to`, `cc` and `bcc`; more fails with `400` [`unsubscribe_single_recipient`](../reference/errors.md#unsubscribe_single_recipient). [Suppressions](suppressions.md) covers both.

```json
{ "suppression_key": "product-updates", "unsubscribe": true }
```
## Tracking

Whether opens and clicks are tracked is a [setting on the sending domain](domains.md#tracking-settings). `tracking` overrides it for one email, either way and one half at a time — what you leave out follows the domain:

```json
{ "tracking": { "opens": false, "clicks": false } }
```
That is how a password reset leaves a domain that tracks everything else with its links untouched, or how one campaign is measured on a domain that otherwise isn't. The override is kept with the message, so a [scheduled](scheduling.md) send and an idempotent retry carry it too; the half left to the domain follows whatever the domain says when the message goes out.

Only mail sent from a domain is tracked. Through a [connected mailbox](mailboxes.md) the provider sends the email as written, so asking for `opens: true` or `clicks: true` there fails with `400` [`tracking_unavailable`](../reference/errors.md#tracking_unavailable). `false` is accepted: it is what happens anyway.

## Scheduling

`scheduled_at` (RFC 3339, with an offset, up to 30 days out) defers the send. The message comes back with `status: "scheduled"` and can be rescheduled, edited or canceled while it's still in that status — see [Scheduling](scheduling.md). `draft: true` holds it for a person to review and send instead — see [Drafts](drafts.md).

```json
{ "scheduled_at": "2026-09-15T09:00:00Z" }
```
## Which space a send is in

Nothing on the request names a [space](spaces.md). The email is in the space `from` is [assigned to](spaces.md#single-addresses-of-one-domain), else in the space of the domain or mailbox it is on, and its thread with it — `space_id` on the response says which, `null` for the workspace itself. A workspace that has no spaces never sees anything else there.

Lists take `space` to narrow to one space:

```ts
for await (const email of aiinbx.emails.list({ space: "spc_...", direction: "inbound" })) {
  console.log(email.from.address, email.subject)
}
```
## Continuing a conversation

Pass `thread_id` to attach a new message to an existing thread. For an actual reply, use [`threads.reply`](threads.md#replying) instead — it infers the sender, recipients, subject, and reply headers, which is the part that's easy to get wrong by hand.

### Replying to mail AI Inbx hasn't seen

Both of those need a thread, and a thread needs the message you're answering to have passed through AI Inbx. Some mail hasn't: a conversation that started before you migrated, or one that lives at another provider while you build one API over several. For that, name the message yourself:

```json
{
  "from": "Ada <ada@example.com>",
  "to": "grace@example.com",
  "subject": "Re: Quick question",
  "text": "Thursday at 10 works.",
  "in_reply_to": "CAF=abc123@mail.gmail.com",
  "references": ["CAF=first@mail.gmail.com", "CAF=abc123@mail.gmail.com"]
}
```
`in_reply_to` is the `Message-ID` of the message being answered and `references` the chain before it, oldest first — what the original's own `References` held, plus its `Message-ID`. Both are written to the wire, so the reply threads in the recipient's mail client; a chain that leaves `in_reply_to` out has it added at the end, where a mail client looks for it. Pass them with or without their angle brackets — `<CAF=abc123@mail.gmail.com>` reads the same — and the API gives them back without, as it does every Message-ID. Leave `references` out and it is `in_reply_to` alone. `references` without `in_reply_to` is accepted too, and writes no `In-Reply-To`.

The email is then threaded the way an inbound message with those headers would be: if one of the IDs is the `message_id` of an email in the same [space](spaces.md), the send joins that thread; if none is, it opens a new one — and the recipient's answer lands on it, because it names the `message_id` of what you sent.

`thread_id` already says what a send answers, so it can't be combined with either field: that is a `422` [`invalid_request`](../reference/errors.md#invalid_request). `threads.reply` and `threads.forward` don't take them at all. A [scheduled](scheduling.md) send keeps the headers you named, where one into a `thread_id` answers whatever is latest on the thread by the time it leaves.

## Next

**[Threads](threads.md)**

Replying without rebuilding headers.

**[Scheduling](scheduling.md)**

Deferred sends, rescheduling, cancellation.

**[Suppressions](suppressions.md)**

Why a recipient came back in `suppressed`.

**[Pacing](pacing.md)**

Sending windows and rate ceilings.
