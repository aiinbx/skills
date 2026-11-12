---
name: aiinbx-concepts
description: "Understanding AI Inbx's objects before acting on them — what makes an address one the workspace can send from (a domain or a connected mailbox), how an email relates to its thread, what a send response and an event each prove, why a recipient can be dropped or a message held, and how spaces keep each customer's mail apart, by domain or by single address. Load before designing or changing an AI Inbx integration, or when an email, thread or event does not behave as expected."
license: MIT
metadata:
  author: aiinbx
  version: "1.1.0"
  homepage: https://docs.aiinbx.com/skills
  source: https://github.com/aiinbx/skills
---

# Concepts

AI Inbx is an email API for software that holds conversations: it sends, receives, threads and replies over one API, from a domain the workspace verified or from a Gmail or Outlook mailbox someone connected. It delivers the mail; the application keeps the records it needs for its own history.

## Object graph

```txt
Workspace
- api keys[]          -> scope: full | sending | read; reach every space
- webhook endpoints[] -> url, subscriptions[], secret; receive every space's events
- domains[]           -> Domain
- mailboxes[]         -> Mailbox
- spaces[]            -> Space
- suppressions[], pacing rules[]

Who the workspace can send as:
Domain (dom_)
- name, region, records[] to publish, verified_at
- provided: true -> <slug>.aiinbx.app, verified with no DNS, capped, one per workspace
- every address on a verified domain sends; the inbound MX record makes it receive
Mailbox (mbx_)
- one Gmail or Outlook account, connected over OAuth
- its address and aliases send and receive; the grant can expire (needs_reauth)

What was sent and what came back:
Email (eml_)
- direction: outbound | inbound
- thread_id -> Thread
- status: draft | scheduled | queued | sending | sent
          -> delivered | bounced | complained | failed; inbound is received
- outbound: suppressed[] (recipients dropped), pacing (non-null while held)
- inbound: delivered_to (your addresses it arrived at), category, stripped_text,
           verdicts, verification, unavailable
- attachments[] -> Attachment (att_), with prepared text for received PDFs
Thread (thr_)
- subject, participants, messages[] -> Email
- threads.reply answers it: sender, recipients, subject, headers inferred
Event (evt_)
- type, created_at, space_id, data
- delivered to webhook endpoints; also listed by GET /events

What stops a send:
Suppression (sup_)  -> address on a list (key), checked on every send
Pacing rule (pace_) -> hours or a rate limit; holds mail it matches

Grouping by customer:
Space (spc_)
- owns domains, mailboxes and assigned addresses, and through them their emails and threads
- an email is in the space of the address it went through when that address is
  assigned to one, else of its domain or mailbox
- thread_across_addresses: off by default (also on the workspace)
- space_id: null is the workspace itself
Address (assigned)
- one address of a verified domain, assigned to a space of its own: addresses.assign / list / remove
- no DNS, no verification: the domain still signs and carries its mail
- most specific wins: the address, else its untagged address (anna+orders@ -> anna@), else the domain
- assigning again moves it from its next email on; mail it already has stays where it was

From call to conversation:
emails.send --> Email (outbound) + Thread
recipient answers --> Email (inbound) on the same Thread --> email.received
threads.reply(thread, email_id) --> Email (outbound) on the same Thread
```

## What each thing proves

Most mistakes come from reading more into a signal than it says.

- **A send response** says an email was created and taken, with its `id` and `thread_id`. It does not say it was delivered. `suppressed` lists recipients that were dropped; a non-null `pacing` says it is held and until when.
- **`email.sent`** says the sending provider accepted it. **`email.delivered`** says a delivery was reported. Neither says a person read it.
- **`email.bounced`** reports a failed delivery per recipient. Only an address shown to be dead is suppressed for later sends.
- **`email.received`** says mail arrived, which thread it joined, and in `delivered_to` which of your addresses it arrived at. `to` is what the sender wrote in the To header. An email that reaches another address of yours later is announced again with the same `email_id`. Its `category` (`human`, `out_of_office`, `auto_reply`, `bounce`, ...) is a classification, not proof of who wrote it.
- **A valid webhook signature** proves the delivery came from AI Inbx. It says nothing about the email's content, which stays untrusted.

## Rules that hold everywhere

- A thread is found for inbound mail from its reply headers and its quoted content. A shared subject alone never merges two conversations. A real reply always joins its thread; mail without reply headers joins by a guess only a thread the address it arrived at already takes part in, unless `thread_across_addresses` is on for its space, which lets the guess join any thread of that space.
- Mail between two of the workspace's own addresses arrives: the receiver gets an email of its own, with its own `email_id` and its own `email.received`. In one space it shares a thread with the sent email; across two spaces each is in a thread of its own space. A reply goes out from the address the mail arrived at.
- A reply needs a thread AI Inbx holds. Mail it has never seen is answered with `emails.send` and `in_reply_to`.
- IDs are prefixed, opaque and stable. An ID of the wrong type is refused.
- Lists are paginated; a thread's messages paginate separately from the list of threads.
- A key belongs to one workspace and reaches every space in it. Keeping one customer out of another's mail is the application's job.
- Events can repeat and arrive out of order. `event.id` is the same across retries and replays.

## Definitions

Load the matching reference when reasoning about that object.

For the whole model in prose, with one message followed through all of it, read `references/concepts.md`.

For domains — the records, verification, regions, diagnostics and the provided domain, read `references/guides/domains.md`.

For mailboxes — connecting a Gmail or Outlook account, aliases and re-authorisation, read `references/guides/mailboxes.md`.

For threads — listing, reading, replying and forwarding, read `references/guides/threads.md`.

For inbound mail — the payload, `stripped_text`, categories and automated replies, read `references/guides/receiving.md`.

For events and webhook endpoints — subscriptions, retries, replay and catching up, read `references/webhooks.md`. For each event type, read `references/webhooks/events.md`.

For suppression lists and unsubscribe, read `references/guides/suppressions.md`.

For pacing rules, held mail and the provided domain's ceilings, read `references/guides/pacing.md`.

For spaces — when a product needs them, how mail finds its space, which to give each customer (their own domain, a subdomain under a wildcard, or single assigned addresses) and `thread_across_addresses`, read `references/guides/spaces.md`.

For attachments — sending files and reading prepared text, read `references/guides/attachments.md`.

For API keys and their scopes, read `references/authentication.md`.

For plans, what counts as an email and what the API answers at a limit, read `references/guides/billing.md`.
