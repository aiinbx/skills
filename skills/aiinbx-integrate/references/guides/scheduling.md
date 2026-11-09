# Scheduling

Schedule a send up to 30 days ahead, move it, or cancel it, and learn what stays editable on a scheduled email until the moment it goes out.

From https://docs.aiinbx.com/guides/scheduling

Pass `scheduled_at` on a send, a reply or a forward and the message is held until then. The API returns it immediately with `status: "scheduled"`, and it stays yours to move or cancel for as long as it's in that status.

```ts TypeScript
const email = await aiinbx.emails.send({
  from: "Ada <ada@example.com>",
  to: "grace@example.com",
  subject: "Monday reminder",
  text: "The review is at 10.",
  scheduled_at: "2026-09-14T09:00:00Z",
})

console.log(email.status) // "scheduled"
```
```python Python
email = client.emails.send(
    {
        "from_": "Ada <ada@example.com>",
        "to": ["grace@example.com"],
        "subject": "Monday reminder",
        "text": "The review is at 10.",
        "scheduled_at": "2026-09-14T09:00:00Z",
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
    "subject": "Monday reminder",
    "text": "The review is at 10.",
    "scheduled_at": "2026-09-14T09:00:00Z"
  }'
```
`scheduled_at` is an RFC 3339 timestamp **with an offset** (`2026-09-14T09:00:00Z` or `2026-09-14T11:00:00+02:00`), at most 30 days out. A naive timestamp is rejected with [`invalid_request`](../reference/errors.md#invalid_request) — the offset is required precisely because "9am" without one means six different instants.

> [!NOTE]
> Scheduling works identically on [`threads.reply`](threads.md#replying) and [`threads.forward`](threads.md#forwarding). A scheduled reply's recipients are fixed when you make the call — a participant who joins the thread in the meantime is not added. Only its `In-Reply-To` and `References` are worked out again at release, so it answers whatever is newest in the thread by then.

## Rescheduling

`PATCH /emails/{id}` moves a scheduled message. The same 30-day bound applies from the moment of the call. The same call changes what it says — recipients, subject, bodies — until it leaves; it is composed again when it goes, so the edit is what is sent.

```ts TypeScript
const moved = await aiinbx.emails.update("eml_...", {
  scheduled_at: "2026-09-15T09:00:00Z",
})
```
```python Python
moved = client.emails.update("eml_...", {"scheduled_at": "2026-09-15T09:00:00Z"})
```
```bash curl
curl -X PATCH https://api.aiinbx.com/api/v2/emails/eml_... \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-09-15T09:00:00Z" }'
```
Only a message still in `scheduled` status — or a [draft](drafts.md) — can change. Once it has been released — including while [pacing](pacing.md) holds it afterwards — the call, like `cancel`, fails with `409` [`not_scheduled`](../reference/errors.md#not_scheduled).

## Canceling

```ts TypeScript
const canceled = await aiinbx.emails.cancel("eml_...")
console.log(canceled.status) // "canceled"
```
```python Python
canceled = client.emails.cancel("eml_...")
```
```bash curl
curl -X POST https://api.aiinbx.com/api/v2/emails/eml_.../cancel \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Cancellation is terminal — a canceled message can't be rescheduled back into flight. Send a new one instead.

Both rescheduling and cancellation need a key with the `sending` or `full` [scope](https://docs.aiinbx.com/authentication.md#scopes).

## Finding what's scheduled

`status: "scheduled"` on the email list is the queue of everything still pending:

```ts TypeScript
for await (const email of aiinbx.emails.list({ status: "scheduled" })) {
  console.log(email.scheduled_at, email.subject, email.to)
}
```
```python Python
for email in client.emails.iter(status="scheduled"):
    print(email["scheduled_at"], email["subject"], email["to"])
```
```bash curl
curl "https://api.aiinbx.com/api/v2/emails?status=scheduled&limit=100" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
## Recomposed at release

A scheduled message is composed and stored when you make the call, then composed again at the instant it's released. Three consequences worth designing around:

- **Suppressions are applied then, too.** A recipient suppressed between scheduling and release is removed from the message's `to`, `cc` or `bcc` — no event is recorded for them. If every `to` recipient is suppressed by then, the message fails instead.
- **Pacing rules are evaluated then, too.** A message scheduled for 3am under an [hours rule](pacing.md#sending-hours) that forbids 3am is held until the window opens; `scheduled_at` sets the earliest time it may go, not a guarantee of the exact instant.
- **Tracking follows the domain then.** Opens and clicks are tracked as the [domain's setting](domains.md#tracking-settings) says when the message leaves. What the send said itself in [`tracking`](sending.md#tracking) is kept with it and still wins.

If you need a message to ignore the pacing rules entirely, set `pacing.skip` on it:

```json
{
  "scheduled_at": "2026-09-14T09:00:00Z",
  "pacing": { "skip": true }
}
```
## Scheduling versus pacing

They solve different problems and compose fine:

| | [Scheduling](scheduling.md) | [Pacing](pacing.md) |
| --- | --- | --- |
| Set on | One message | Workspace-wide rules |
| Answers | "Not before this instant" | "Not faster than this, and not outside these hours" |
| Changed by | `update` / `cancel` | Rule edits, or releasing from the queue |

Use `scheduled_at` for "send this Monday morning". Use pacing for "never more than 200 an hour from this domain".
