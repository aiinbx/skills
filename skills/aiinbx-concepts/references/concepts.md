# Core concepts

The building blocks of AI Inbx: workspaces, sending identities, emails, threads, events, and the spaces that keep each customer's mail apart.

From https://docs.aiinbx.com/concepts

A workspace contains your API keys, sending identities, messages, and configuration. Use the resources below to configure where mail comes from, work with conversations, and handle delivery outcomes.

## The two halves

Everything here divides cleanly. On one side are the objects that establish **who you can send as**: a domain you publish DNS for, or a mailbox a customer authorized. On the other are the objects that carry **what you sent and what came back**: emails, threads, and the events they emit.

You need exactly one thing from the first half before anything in the second half works.

**Identity**

**Domain** and **Mailbox** — the two ways an address becomes yours to send
from. Pick one; most products end up with both.

**Traffic**

**Email**, **Thread**, **Attachment** — the messages themselves, and the
conversation each one belongs to.

## Identity

### Domain

A domain you control the DNS for. Creating one returns the records to publish — SPF, DKIM, DMARC, a return path, and an MX record if you want to receive on it — and the domain can send once its DKIM identity is verified. Receiving also requires the inbound MX record.

A domain does not have to be _yours_. Publishing the records at a customer's registrar makes their domain send through you, which is how a product sends as `support@theircompany.com`.

```
dom_… → verified_at, region, records[]
```
**[Domains](guides/domains.md)**

Each record, what `diagnostics` reports when one won't resolve, and regions.

### Mailbox

A Gmail or Outlook account a customer authorized over OAuth. No DNS, no verification — they click through a consent screen and the mailbox is live. In exchange you get one address rather than a whole domain, and the authorization can expire.

```
mbx_… → address, provider, app_id, state
```
**[Mailboxes](guides/mailboxes.md)**

The connect flow, `ref` for matching a mailbox to your user, and reauth.

> [!NOTE]
> Domain and mailbox aren't "yours" versus "your customer's" — they're two different permissions. A domain needs DNS access; a mailbox needs an OAuth grant. Which one you can get is usually decided by the customer, not by you.

## Traffic

### Email

One message, inbound or outbound, with its addresses, bodies, headers, and attachments. Sending returns one; receiving one fires a webhook that names it.

An email's `status` follows it out — `draft`, `scheduled`, `queued`, `sending`, `sent` — and then records what the far end said: `delivered`, `bounced`, `complained` or `failed`. Inbound mail is `received`. Each outcome also arrives as an event.

```
eml_… → thread_id, status, suppressed[], pacing
```
### Thread

The conversation an email belongs to, and the reason this API exists. A reply lands on its thread even when the client dropped the reply headers, a colleague answers from another address, or the subject changed along the way. A shared subject alone never merges two conversations, so unrelated "Re: Hi" messages don't collide, and mail that could belong to more than one starts a thread of its own rather than joining the wrong one.

Because the thread is a real object, replying takes the thread ID and your text. The sender, the recipients, the subject, and the `In-Reply-To`/`References` headers are inferred from what's already on the thread.

```
thr_… → subject, mailbox, messages[]
```
**[Threads](guides/threads.md)**

Reading conversations, and what `threads.reply` fills in for you.

### Attachment

A file on an email. Outbound, you pass base64 content. Inbound, you get metadata plus a `preparation` field — AI Inbx extracts a received PDF to Markdown so an agent can read it without you running a parser.

```
att_… → filename, content_type, size, preparation
```
## Delivery outcomes

### Event and webhook endpoint

Everything that happens after acceptance is an **event**: delivered, bounced, complained, opened, clicked, unsubscribed, plus the inbound `email.received` that starts most applications. A **webhook endpoint** is a URL subscribed to some of them.

Subscribe to webhooks for delivery outcomes as they occur. You can also retrieve an email and inspect its `events` array. A provider accepting a message does not establish that it reached the recipient’s inbox.

```
whk_… → url, subscriptions[], secret
evt_… → type, created_at, data
```
**[Webhooks](webhooks.md)**

Endpoints, subscriptions, retries, and replay.

**[Event types](webhooks/events.md)**

Each event type and its payload fields.

## Sending controls

Two objects exist only to stop a send that shouldn't happen.

### Suppression

An address that must not be mailed, on a named list. Bounces that show an address is dead, and complaints, create these automatically; unsubscribes and your own calls add more. Every send is checked against the list named by `suppression_key`, plus the org-wide `*` list.

Suppression is why a send can succeed with recipients missing — read `suppressed` on the response.

```
sup_… → address, key, scope, reason
```
### Pacing rule

A rule that holds the mail it `match`es: an `hours` rule keeps it inside weekly windows, a `limit` rule caps how many go per rolling window. A held message comes back with a non-null `pacing` object naming the rule and projecting when it will go.

Transactional mail shouldn't queue behind a campaign, so a send can set `pacing.skip`.

```
pace_… → kind, match, schedule | limit
```
**[Suppressions](guides/suppressions.md)**

Lists, scopes, and one-click unsubscribe.

**[Pacing](guides/pacing.md)**

Sending hours, rate limits, and releasing held mail by hand.

## Grouping by customer

### Space

Optional. A group inside the workspace — one per customer, typically — that owns domains and mailboxes, and with them every email and thread that went through those. Pacing rules and suppression lists can be put in a space too, and then reach that space alone. Sending reputation is kept per space as well, so one customer's bounces never pause another's mail. API keys and webhook endpoints are workspace-wide.

Mail is never told its space: an email is in the space of the domain or mailbox it went through, and a reply is in its thread's. Any space-aware resource created without a space belongs to the workspace itself and reads `space_id: null`.

```
spc_… → name
```
**[Spaces](guides/spaces.md)**

When you need one, and a platform walkthrough end to end.

## A message, through all of it

1. **You send**

    `POST /emails` checks the `from` address against a verified **domain** or an
    active **mailbox**, checks each recipient against the **suppression** lists,
    and asks the **pacing** rules whether it can go now.

2. **An email and a thread exist**

    You get an **email** back with a `thread_id` — a new **thread** if nothing
    matched, an existing one if `thread_id` was passed. `suppressed` lists who was
    dropped; `pacing` is non-null if the message is held.

3. **Events arrive**

    `email.sent`, then `email.delivered` — or `email.bounced`. A bounce that shows
    the address is dead writes a **suppression** so the next send to it never
    leaves.

4. **They reply**

    Their message is matched onto the same **thread** and arrives as
    `email.received`, carrying the `thread_id` and a `category` that tells you
    how the message was classified. Classification is a signal for your application, not proof of sender identity. An out-of-office or auto-reply that says its sender can't be reached carries `unavailable` (why, the day they're back, who to write to instead); see [What automated mail asks for](guides/receiving.md#what-automated-mail-asks-for).

5. **You reply**

    `threads.reply` with the thread ID and your text. Every header that makes it a
    real reply is filled in from the thread.

## Everything has an ID

Prefixed, opaque, stable, safe to store. The prefix is part of the ID — an ID of the wrong type is refused (`404` in the path, `400` in a body or query), never a silent match against another resource.

**[Conventions](https://docs.aiinbx.com/reference/conventions.md)**

The full prefix table, pagination, timestamps, request IDs, and idempotency.
