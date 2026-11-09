# email.received

email.received fires for every inbound message, already matched onto its thread, and names the addresses of yours it arrived at. The event most AI Inbx applications are built around.

From https://docs.aiinbx.com/webhooks/events/email-received

Inbound mail. The one most applications are built around — it fires once per received message, and again when that message reaches another address of yours later, with the thread it belongs to already resolved and the addresses of yours it arrived at in `delivered_to`.

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
## Which address it arrived at

`to` and the email's `cc` are what the sender wrote. `delivered_to` is where the mail arrived: the receiving server's recipients on a domain, the mailbox and the aliases the mail is written to on a connected mailbox. Route on `delivered_to` — an address of yours that was only copied or blind-copied is there, and an address the sender merely named in a header is not.

```ts
if (event.type === "email.received") {
  for (const address of event.data.delivered_to) {
    await inboxFor(address).handle(event.data.email_id)
  }
}
```
Every address of yours a message reaches is in exactly one `email.received`. Mail servers can deliver one message in several rounds — a blind copy often travels on its own — so when the same email reaches another address of yours later, the event is sent again with the same `email_id` and only the new addresses in `delivered_to`. A handler that acts once per email deduplicates on `email_id`; one that acts per address handles each event as it comes. The email itself lists them all in its own `delivered_to`.

> [!TIP]
> Gate on `category === "human"` before letting an agent reply. Without it, an out-of-office autoresponder and your bot will happily talk to each other.

Automated mail can still carry what your code needs: `unavailable` on an out-of-office or auto-reply (why its sender can't be reached, the day they're back, who to write to instead), `verification` on a verification email (its code and link). See [What automated mail asks for](../../guides/receiving.md#what-automated-mail-asks-for).

**[Receiving](../../guides/receiving.md)**

Getting mail to arrive in the first place — a domain's MX record, or a connected mailbox.
