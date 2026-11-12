# Receiving

Two ways mail reaches your code — an MX record on a domain, or a connected Gmail or Outlook mailbox — and what arrives when it does.

From https://docs.aiinbx.com/guides/receiving

There are two ways inbound mail reaches your application. Both work equally well for a domain or account you run and for one belonging to a customer who connected it through your app — what differs is what the setup needs and how much it covers.

**A domain**

Point an MX record at AI Inbx and *every* address on the domain becomes an
inbox your code can read. Needs DNS access, and takes over the domain's
mail.

**A connected mailbox**

An existing Gmail or Outlook account, authorized over OAuth and synced in.
Needs no DNS, changes nothing about the account, and covers one address.

Both land in the same place: a thread, and an [`email.received`](../webhooks/events/email-received.md) webhook.

## Receiving on a domain

Add the `INBOUND` MX record from the domain's `records` array and verify the domain — that's the whole setup. Whose domain it is doesn't matter to the API; if it's a customer's, you show them the records and they publish them. See [Domains](domains.md) for the record set and how verification works.

Once the MX record resolves, mail to any address on the domain is accepted, parsed, threaded, and delivered to your [webhook endpoints](../webhooks.md). It lands in the domain's [space](spaces.md), or in the space of the address it was sent to when that address is [assigned to one of its own](spaces.md#single-addresses-of-one-domain).

> [!TIP]
> **Plus addressing**
>
> Every address on the domain is live, so you can encode routing into the local part — `reply+ord_8812@yourapp.com` — and read it back off `data.delivered_to` in the webhook. Nothing to register per address.

## Receiving through a connected mailbox

When the address already has an inbox at Google or Microsoft — so pointing an MX record at AI Inbx would take mail away from it — connect the mailbox instead. AI Inbx creates a one-time authorization URL, whoever holds the account approves it, and their mail syncs in — the last seven days by default, then everything new:

```ts
const { url } = await aiinbx.mailboxes.connect({
  provider: "google",
  app_id: "app_...", // your own OAuth app
  return_to: "https://app.example.com/settings/mailboxes",
  ref: "user_8812",
})

// Send the customer to `url`.
```
`ref` is your own identifier, kept on the mailbox and carried on [`mailbox.connected`](https://docs.aiinbx.com/webhooks/events/mailbox-connected.md) — and every mailbox event after it — so you know which of your users it is about; [`metadata`](mailboxes.md#ref-and-metadata) holds anything more. [Mailboxes](mailboxes.md) covers the full flow — including a hosted page and a plain connect link that need no server call at all — and registering the OAuth app of your own that every connection goes through.

## What arrives

A received message is delivered as an `email.received` event whose `data` carries the routing you need without a follow-up fetch:

```json
{
  "id": "evt_...",
  "type": "email.received",
  "created_at": "2026-09-01T10:31:04Z",
  "space_id": null,
  "data": {
    "email_id": "eml_...",
    "message_id": "CAF7k2xQ@mail.gmail.com",
    "thread_id": "thr_...",
    "domain_id": "dom_...",
    "mailbox_id": null,
    "from": "grace@example.com",
    "to": ["support@yourapp.com"],
    "delivered_to": ["support@yourapp.com"],
    "subject": "Re: Quick question",
    "snippet": "Thursday at 10 works for me…",
    "category": "human",
    "verdicts": { "spam": "PASS", "spf": "PASS", "dkim": "PASS", "dmarc": "PASS" },
    "verification": null,
    "unavailable": null,
    "attachments": []
  }
}
```
`to` is the To header as the sender wrote it. `delivered_to` is the addresses of yours the mail arrived at, so an address that was only in Cc or Bcc is there. Route on it; [Which address it arrived at](../webhooks/events/email-received.md#which-address-it-arrived-at) has the rules.

Retrieve the full message — bodies, headers, attachments, delivery events — with `emails.retrieve(data.email_id)`, or fetch the whole conversation with `threads.retrieve(data.thread_id)`. To skip that call, create the endpoint with [`payload: "full"`](../webhooks.md#full-payloads): the same event then carries the email as `data.email`, prepared attachment text included.

### What was written

A reply carries the whole conversation underneath it, and people often type their answers inside that quote. `stripped_text` is the part the sender actually wrote: the quoted conversation and the signature removed, and an answer typed into the quote kept under the line it answers. The signature stays the first time a sender uses it in a thread — that is where their name, role and phone number are — and goes once a message of theirs that this one answers already carries it.

```json
{
  "text": "Hi Lena, answers inline.\n\nOn Mon, 1 Sep 2026 at 10:44, Lena Bergmann <lena@bergmann-metall.de> wrote:\n> 1. How many units per month?\n\n500, ramping to 800 in Q4.\n\n> 2. Which finish?\n\nAnodised black.\n\nBest\nTom",
  "stripped_text": "Hi Lena, answers inline.\n\n> 1. How many units per month?\n500, ramping to 800 in Q4.\n\n> 2. Which finish?\nAnodised black.\n\nBest\nTom",
  "segments": [
    { "kind": "written", "text": "Hi Lena, answers inline." },
    { "kind": "quoted", "text": "On Mon, 1 Sep 2026 at 10:44, Lena Bergmann <lena@bergmann-metall.de> wrote:\n1. How many units per month?" },
    { "kind": "written", "text": "500, ramping to 800 in Q4." },
    { "kind": "quoted", "text": "2. Which finish?" },
    { "kind": "written", "text": "Anodised black." },
    { "kind": "signature", "text": "Best\nTom" }
  ]
}
```
Answers typed into the quote are found even when the mail client leaves the quote unmarked, as Outlook does, as long as the message being answered is on the thread. `segments` is the whole cut in order, for a view that folds the quote or a model that wants the context too.

A message that answers nothing on file — a forwarded conversation, a reply to a mail sent before the mailbox was connected — keeps its quote in `stripped_text`: nothing underneath has been seen, so it is the content. `segments` still marks it as quoted.

`stripped_html` is the HTML part with the same cut: the quoted conversation and a repeated signature removed, the line an answer sits under kept above it as a `<blockquote>`. It is null when the message has no HTML part, or when its markup can't be cut cleanly — then render `html` whole rather than half of it.

Give a model `stripped_text` for what was said and the thread for what was said before it.

### Categories

Every inbound message is classified, so an autoresponder doesn't get treated as a human answer. `category` is one of the values below, or `null` when classification failed — the email is still stored and delivered, just without a category:

| Category | Meaning |
| --- | --- |
| `human` | Classified as a message written by a person. |
| `out_of_office` | An away or vacation autoreply. |
| `auto_reply` | Another kind of automatic response — ticket acknowledgements, "we got your message". |
| `bounce` | A delivery failure notification that arrived as mail. A delivery report about a message you sent isn't stored as an email at all: it lands on that message as [`email.bounced`](https://docs.aiinbx.com/webhooks/events/email-bounced.md) (or a `delivered` or `deferred` delivery event, whichever it reports). |
| `verification` | A confirmation or code email. |
| `transactional` | Receipts, invoices, account notices. |
| `notification` | Alerts and system mail. |
| `marketing` | Bulk or promotional mail. |
| `spam` | Junk, scams or phishing, or failed the receiving side's spam check (`verdicts.spam: "FAIL"`). Unsolicited sales pitches are `marketing`. |

Gate your agent on it:

```ts
if (event.type === "email.received" && event.data.category === "human") {
  await handleReply(event.data.thread_id)
}
```
This filter reduces automatic reply loops, but classification can be wrong. Also apply sender checks, conversation reply limits, and human handoff rules. See [Email agents](../integrations/agents.md#decide-whether-to-answer).

### What automated mail asks for

Two kinds of automated mail ask for something a program can do on its own, so the email carries what that takes. Both fields are on the email and on its `email.received` event, and are `null` on every other category.

`verification` is on a `verification` email: the one-time code to type in and the link that verifies, confirms, signs in or resets.

```json
{
  "category": "verification",
  "verification": { "code": "482913", "link": "https://acme.dev/verify?t=9f8e7d" }
}
```
`unavailable` is on an `out_of_office` or `auto_reply` email that says its sender can't be reached: why, the day they're back, and the addresses it says to write to instead.

```json
{
  "category": "out_of_office",
  "unavailable": {
    "reason": "absent",
    "back_on": "2026-11-13",
    "contacts": ["anna.schmidt@bergmann-metall.de"]
  }
}
```
| `reason` | Meaning |
| --- | --- |
| `absent` | Away for now — holiday, leave, travel. `back_on` is the first day back when the reply names one; a reply away "until" a day is read as back the next working day. |
| `left` | No longer at this address. |
| `unmonitored` | Nobody reads the address. |

A ticket acknowledgement or a thank-you has no `unavailable`: its sender can be reached. A reply still on after its sender came back has no `back_on`: `unavailable` is read once, when the reply arrives.

A model picks these, but only from the email itself: each code, link and contact is a string written in it, and `back_on` comes from a day it names, so nothing in them is made up. A pick can still be the wrong one of those strings. When a value isn't in the mail, or the pick fails, the field is `null` or empty, and the email arrives as usual.

```ts
if (event.data.unavailable?.contacts.length) {
  // Turn the sequence to the colleague the reply named.
  await restartSequence(event.data.thread_id, event.data.unavailable.contacts)
} else if (event.data.unavailable?.back_on) {
  await pauseSequenceUntil(event.data.thread_id, event.data.unavailable.back_on)
}
```
Mail received before these fields were added has `null` in both.

### Authentication verdicts

`verdicts` reports what SPF, DKIM, DMARC, and spam scanning concluded about the *sender*: each of `spam`, `spf`, `dkim` and `dmarc` is `PASS`, `FAIL`, `GRAY` or `PROCESSING_FAILED`. Treat a `FAIL` as reason not to act on the content, particularly for anything that changes state.

The verdicts are stored with the message. The email you list or retrieve carries the same `verdicts` object the event did, so a sync that replays from `emails.list` loses nothing next to one fed by webhooks.

`verdicts` is `null` when the receiving side didn't say: mail that arrived through a [connected mailbox](mailboxes.md) — Gmail and Microsoft have made their own call before it reaches the inbox, and don't pass it on — and mail you sent. A single check is `null` when it wasn't reported — `spam` is, on emails received before late September 2026, when only the other three were kept.

## Threading

Replies land on the conversation they belong to, even when a client mangles or drops the reply headers, a colleague answers from another address, or the subject changes along the way. Subject alone is never enough to merge two conversations, so two unrelated messages that happen to share "Re: Invoice" stay apart, and mail that could belong to either starts a thread of its own.

The upshot for your code: trust `thread_id`. [Threads](threads.md) covers reading and replying.

## Routing which events go where

An endpoint subscribes to the event types it wants, and can further narrow by sender or recipient with routing rules — useful when one workspace serves several products:

```ts
await aiinbx.webhookEndpoints.create({
  url: "https://app.example.com/webhooks/support",
  subscriptions: ["email.received"],
  routing: [{ effect: "allow", field: "to", pattern: "support@yourapp.com" }],
})
```
See [Webhooks](../webhooks.md#routing) for how allow and block rules combine.

## Attachments on inbound mail

Inbound attachments are stored and exposed as metadata plus a short-lived download URL. PDFs, documents, and spreadsheets are additionally prepared into Markdown or text so a model can read them without you running a parser — see [Attachments](https://docs.aiinbx.com/guides/attachments.md).

## Next

**[Webhooks](../webhooks.md)**

Endpoints, signatures, retries, replays.

**[Threads](threads.md)**

Reading a conversation and replying to it.

**[Domains](domains.md)**

DNS records, verification, diagnostics.

**[Mailboxes](mailboxes.md)**

Gmail and Outlook, and your own OAuth app.
