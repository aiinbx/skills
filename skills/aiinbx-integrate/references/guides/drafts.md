# Drafts

Let an agent write an email and a person decide: create drafts over the API, then review, edit and send them from the console or your own UI.

From https://docs.aiinbx.com/guides/drafts

Pass `draft: true` on a send, a reply or a forward and nothing goes out. The email is stored with `status: "draft"`, filed in its thread, and waits for someone to send it — a person in the console, or your own code once your own reviewer has said yes.

It is the human in the loop for agents that write mail: the agent drafts, a person reads, changes what needs changing, and sends.

```ts TypeScript
const draft = await aiinbx.threads.reply("thr_...", {
  text: "Hi Dana — your refund of €40 is on its way.",
  draft: true,
})

console.log(draft.status) // "draft"
console.log(draft.draft?.review_url) // hand this to the person who decides
```
```python Python
draft = client.threads.reply(
    "thr_...",
    {"text": "Hi Dana — your refund of €40 is on its way.", "draft": True},
)

print(draft["draft"]["review_url"])
```
```bash curl
curl https://api.aiinbx.com/api/v2/threads/thr_.../reply \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "text": "Hi Dana — your refund of €40 is on its way.", "draft": true }'
```
A draft uses none of your plan's emails. It is counted when it is sent, like any other send.

## Reviewing in the console

Drafts show where the rest of your mail does: the **Threads** list's _Drafts_ tab holds every thread with one waiting, counted in the rail, and the **Emails** list has a _Drafts_ tab of its own. Opening one — or following `draft.review_url`, which works for any member of the workspace without your code knowing anything about the console — opens its thread, landed on the draft.

The draft sits under the conversation it answers, and everything in it is editable: recipients, subject, the body — rich text for an HTML email, plain for a text one — and its attachments. **Edit** opens the fields in place; **Save changes** keeps them. A designed email — a layout, not prose — is shown as it will be sent rather than flattened by an editor; it can be rewritten as plain text if it should not go as it is.

**Send** (<kbd>⌘</kbd> <kbd>↵</kbd>) holds the email for twenty seconds with Undo, like every send from the console — Undo makes it a draft again. It can also be sent at a later time, or discarded.

## Editing and sending from code

Build your own review screen on the same three calls. `PATCH /emails/{id}` changes anything the email says — the same call that moves a [scheduled](scheduling.md) email:

```ts TypeScript
await aiinbx.emails.update("eml_...", {
  subject: "Your refund is on its way",
  html: "<p>Hi Dana,</p><p>Your refund of <strong>€40</strong> is on its way.</p>",
})

const sent = await aiinbx.emails.sendDraft("eml_...")
console.log(sent.status) // "scheduled" — it leaves in a moment
```
```python Python
client.emails.update(
    "eml_...",
    {"subject": "Your refund is on its way", "html": "<p>Hi Dana, …</p>"},
)

sent = client.emails.send_draft("eml_...")
```
```bash curl
curl -X PATCH https://api.aiinbx.com/api/v2/emails/eml_... \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "subject": "Your refund is on its way" }'

curl -X POST https://api.aiinbx.com/api/v2/emails/eml_.../send \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
- **Only what you name changes.** Recipients, `subject`, `html`, `text` and `attachments` can each be set. The text part is not rewritten from new HTML — send both to keep them alike. The sender stays as it was written.
- **Attachments are the list it carries from now on.** `{ "id": "att_..." }` keeps one the email has; a new one is given as on send. Left out, they stay; `[]` removes them all. New files may come to 3 MB per request, and everything the email carries to 7 MB.
- **Sending is scheduling.** `POST /emails/{id}/send` turns the draft into a `scheduled` email — at the `scheduled_at` you pass, else the draft's own if it is still ahead, else now — and it takes the path every scheduled email takes: suppressions, threading and [pacing](pacing.md) are decided when it goes.
- **`created_at` moves when it is sent.** The email takes its place in its thread then; `draft.drafted_at` keeps when it was written.
- **Discarding is canceling a draft.** `POST /emails/{id}/cancel` throws a draft away. It keeps its id and its status becomes `discarded`, so whoever wrote it can still read what became of it. It stays among its thread's messages, but never joins the conversation.

Anything but a draft answers `/send` with `409` [`not_draft`](../reference/errors.md#not_draft); an email that has gone out, was canceled or was discarded answers `PATCH` with `409` [`not_scheduled`](../reference/errors.md#not_scheduled).

## Knowing a person approved it

An email written as a draft keeps a `draft` object after it is sent: `drafted_at`, and `edited_at` when anyone changed it before it went.

## Drafts and threads

A draft sits in its thread without being part of the conversation yet: a reply sent meanwhile does not answer it, a forward does not carry it, and the thread's activity time does not move for it. When the draft is sent it takes its place in the thread as of that moment, and answers whatever is newest there by then.

Until then it counts for nothing in the thread: `GET /threads/{id}` lists it among `messages`, but not in `message_count` or `participants`. A draft that starts a new thread makes one that `GET /threads` already lists, with `message_count: 0` and `participants: []`.
