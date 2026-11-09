# email.received

email.received fires once per inbound message, already matched onto its thread. The event most AI Inbx applications are built around.

From https://docs.aiinbx.com/webhooks/events/email-received

Inbound mail. The one most applications are built around — it fires once per received message, with the thread it belongs to already resolved.

## Payload

Every field, with an example body, is in the [`email.received` reference](https://docs.aiinbx.com/api/endpoints/webhook-events/email-received.md), generated from the API contract.

## Handling it

The payload carries a snippet, not the body — fetch the email when you need the full content.

```ts
if (event.type === "email.received" && event.data.category === "human") {
  const email = await aiinbx.emails.retrieve(event.data.email_id)
  await queueReply(event.data.thread_id, email.text)
}
```
Or have it sent along. An endpoint created with [`payload: "full"`](../../webhooks.md#full-payloads) gets the email as `data.email` — bodies, stripped text, headers, and every attachment with its signed URLs and prepared text — read when the delivery is sent, so a retry carries fresh URLs. The summary fields stay as they are. Fall back to the retrieve when it is `null`:

```ts
if (event.type === "email.received" && event.data.category === "human") {
  const email =
    event.data.email ?? (await aiinbx.emails.retrieve(event.data.email_id))
  await queueReply(event.data.thread_id, email.stripped_text)
}
```
> [!TIP]
> Gate on `category === "human"` before letting an agent reply. Without it, an out-of-office autoresponder and your bot will happily talk to each other.

Automated mail can still carry what your code needs: `unavailable` on an out-of-office or auto-reply (why its sender can't be reached, the day they're back, who to write to instead), `verification` on a verification email (its code and link). See [What automated mail asks for](../../guides/receiving.md#what-automated-mail-asks-for).

**[Receiving](../../guides/receiving.md)**

Getting mail to arrive in the first place — a domain's MX record, or a connected mailbox.
