# Spaces

Group a workspace's domains, mailboxes and mail by customer — one space each — with rules and suppression lists that reach that space alone.

From https://docs.aiinbx.com/guides/spaces

A **space** is a group inside a workspace — one per customer, typically. It owns domains, mailboxes and single addresses assigned to it, and with them every email and thread that went through those. Pacing rules and suppression lists can be put in a space too; then they reach that space alone. New API keys and webhook endpoints stay yours: your servers hold the credential and receive every space's events.

Spaces are optional. A workspace that is not a platform never creates one, and every space-aware resource it has reads `space_id: null` — the workspace itself. Nothing on this page applies until you create a space.

## When you need one

You are a platform if your customers each have addresses of their own and must not see each other's mail. Then:

- Each customer gets a space, and a domain, a mailbox or single addresses in it — [Which to give each customer](#which-to-give-each-customer) picks one.
- Their mail lands in that space. Two of your customers on the same email — the same `Message-ID` — each get their own copy, and a reply is threaded inside its space and never across.
- A pacing rule and a suppression list can be per customer, while yours stay workspace-wide and apply to everyone.
- Sending reputation is per customer too: a space that bounces is paused by itself while the rest keep sending. See [One reputation per space](#one-reputation-per-space).
- Your one key and one endpoint serve every customer. Nothing is handed to a customer — you front the API for them — and every event says which space it is about.

You do not need one if every address in the workspace is yours, or if the separation you want is between campaigns rather than between customers — [suppression keys](suppressions.md#keys) and [webhook routing](../webhooks.md#routing) already do that without a second object.

## Which to give each customer

| When                                                                                                                        | Use                                                                                                                                         |
| --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| The customer brings their own domain: they have DNS access and want their brand and their own reputation.                   | [Their domain](domains.md), added in the customer's space.                                                                             |
| Each customer should get a name of their own, or any number of addresses, without you touching DNS per customer.            | [A subdomain under your wildcard](domains.md#wildcards-and-subdomains) — `acme.saas.com` under `*.saas.com` — in the customer's space. |
| Each customer, or each agent, has one or a few addresses on a domain you already have: `anna@agents.com`, `bob@agents.com`. | [Single addresses](#single-addresses-of-one-domain), each assigned to its space.                                                            |

One customer can have any mix of the three. When several addresses of one space write to the same contacts — a sales or outreach team — turn on [`thread_across_addresses`](threads.md#which-thread-incoming-mail-joins) for that space.

## How mail finds its space

Mail is never told its space. An email is in the space of the address it was sent from or received on when that address is [assigned to a space](#single-addresses-of-one-domain), else of its domain or mailbox; a reply is in its thread's space. So a space is decided when a domain, mailbox or address is put in it, and everything after that follows.

- workspace/
  - *.saas.com  (wildcard domain, space_id: null)
  - agents.com  (domain, space_id: null)
  - Acme  (spc_…)/
    - acme.saas.com  (subdomain)
    - acme.com  (their own domain)
    - tom@acme.com  (their Gmail)
    - anna@agents.com  (assigned address)
    - every email and thread through those
  - Globex  (spc_…)/
    - globex.saas.com

Everything created without a `space_id` belongs to the workspace itself. That includes the wildcard and `agents.com` above: they are yours, and the subdomains under the wildcard and the addresses assigned on `agents.com` are your customers'.

### One address, one space

An address belongs to one space at a time, so a conversation never splits across two:

- **A domain decides its addresses' space**, except an address [assigned to a space of its own](#single-addresses-of-one-domain). While `acme.com` is verified in Acme's space, a mailbox on `tom@acme.com` can only be connected there. Connecting it in another space fails with `409` [`mailbox_space_conflict`](https://docs.aiinbx.com/reference/errors.md#mailbox_space_conflict).
- **Names change hands.** When a customer lets `acme.com` lapse and someone else buys it, add it in the new customer's space. That works even though the old space still has the name. Whoever's DKIM record DNS carries sends and receives as it, exactly as between workspaces — see [Moving a domain](domains.md#moving-a-domain-between-spaces-and-workspaces). The old space keeps its mail, and anything still scheduled or queued there fails instead of going out as the new owner.
- **So do mailboxes.** Connecting a different Google or Microsoft account on an address already connected in another space disconnects the old one, because the address belongs to the new account now.

## A platform, end to end

The example is an AI email assistant sold to businesses. Each customer gets `<customer>.saas.com` to write from on day one, can bring their own domain later, and can connect a Gmail account so the assistant answers from their existing inbox. All of it with one workspace-wide key on your side.

1. **Connect the wildcard**

    `*.saas.com` is one domain that covers every subdomain. It is verified once.

    ```ts TypeScript
    const wildcard = await aiinbx.domains.create({
      name: "*.saas.com",
      region: "eu-central-1",
    })
    ```
    ```bash curl
    curl https://api.aiinbx.com/api/v2/domains \
      -H "Authorization: Bearer $AI_INBX_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{ "name": "*.saas.com", "region": "eu-central-1" }'
    ```
    ```json
    {
      "id": "dom_...",
      "name": "*.saas.com",
      "region": "eu-central-1",
      "space_id": null,
      "parent_id": null,
      "verified_at": null,
      "records": [
        {
          "purpose": "DKIM",
          "type": "TXT",
          "name": "aibx._domainkey.saas.com",
          "value": "v=DKIM1; k=rsa; p=…"
        },
        {
          "purpose": "INBOUND",
          "type": "MX",
          "name": "*.saas.com",
          "value": "10 inbound-smtp.eu-central-1.amazonaws.com"
        },
        {
          "purpose": "RETURN_PATH",
          "type": "MX",
          "name": "bounces.saas.com",
          "value": "10 feedback-smtp.eu-central-1.amazonses.com"
        },
        {
          "purpose": "SPF",
          "type": "TXT",
          "name": "bounces.saas.com",
          "value": "v=spf1 include:amazonses.com -all"
        },
        {
          "purpose": "DMARC",
          "type": "TXT",
          "name": "_dmarc.saas.com",
          "value": "v=DMARC1; p=none; rua=mailto:dmarc@saas.com"
        }
      ]
    }
    ```
2. **Publish the records**

    Four names in the `saas.com` zone. The MX record goes at `*.saas.com`, not the apex — `saas.com` keeps whatever mail setup it already has, and only `anything.saas.com` routes to AI Inbx.

    | Name                       | Type         | Serves                               |
    | -------------------------- | ------------ | ------------------------------------ |
    | `aibx._domainkey.saas.com` | `TXT`        | DKIM for every subdomain.            |
    | `*.saas.com`               | `MX`         | Inbound mail for every subdomain.    |
    | `bounces.saas.com`         | `MX` + `TXT` | Return path and SPF.                 |
    | `_dmarc.saas.com`          | `TXT`        | DMARC, inherited by every subdomain. |

    Then `domains.verify` or wait for [`domain.verified`](https://docs.aiinbx.com/webhooks/events/domain-verified.md). The wildcard and `saas.com` are one identity on the mail provider, so only one of the two can exist in AI Inbx — connect `saas.com` on its own if you want to send from the apex too.

    > [!WARNING]
    > If `*.saas.com` is already a wildcard `CNAME` — for customer sites on `<customer>.saas.com`, say — the MX record cannot coexist with it: a name with a `CNAME` may hold no other record. Use `*.mail.saas.com` or similar for mail then. Everything else on the list is safe beside an existing setup.

3. **Create a space for the customer**

    ```ts TypeScript
    const space = await aiinbx.spaces.create({
      name: "Acme",
      external_id: "cus_8812",
    })
    ```
    ```bash curl
    curl https://api.aiinbx.com/api/v2/spaces \
      -H "Authorization: Bearer $AI_INBX_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{ "name": "Acme", "external_id": "cus_8812" }'
    ```
    ```json
    {
      "id": "spc_...",
      "name": "Acme",
      "external_id": "cus_8812",
      "created_at": "2026-09-03T09:12:41Z"
    }
    ```
    `name` is for you; nothing is derived from it. `external_id` is the customer's id in _your_ system — the key in your tenant table — and it is what ties the two sides together:

    - It is unique in the workspace. Creating a second space with the same `external_id` fails with `409` [`external_id_taken`](https://docs.aiinbx.com/reference/errors.md#external_id_taken), so a create keyed on your customer id cannot make a duplicate on retry.
    - The space can be found by it: `GET /spaces?external_id=cus_8812` returns a page with that space or nothing.
    - It shows on the space in the console, and the spaces page searches it.

    Store the `spc_…` id against your customer record all the same — it is what every later call names, and what every event carries. `external_id` is for the other direction: the moment you hold a customer and need their space.

4. **Give them a subdomain, in the space**

    One label under a wildcard you own is created as a **subdomain** of it: no records, `parent_id` set, verified the moment the wildcard is.

    ```ts TypeScript
    const domain = await aiinbx.domains.create({
      name: "acme.saas.com",
      space_id: space.id,
    })

    console.log(domain.parent_id, domain.verified_at) // wildcard.id, already set
    ```
    ```bash curl
    curl https://api.aiinbx.com/api/v2/domains \
      -H "Authorization: Bearer $AI_INBX_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{ "name": "acme.saas.com", "space_id": "spc_..." }'
    ```
    ```json
    {
      "id": "dom_...",
      "name": "acme.saas.com",
      "region": "eu-central-1",
      "space_id": "spc_...",
      "parent_id": "dom_...",
      "verified_at": "2026-09-03T08:40:12Z",
      "records": []
    }
    ```
    Nothing to publish, nothing to wait for. `region` is the wildcard's, but the sending reputation is Acme's own: what they bounce counts against their space, not against `*.saas.com`. See [Domains](domains.md#wildcards-and-subdomains) for what is and isn't a subdomain.

5. **Send as the customer**

    Nothing on a send names a space. `from` decides: the send lands in the space of the address it is from, so your one key sends for every customer.

    ```ts TypeScript
    const email = await aiinbx.emails.send({
      from: { name: "Acme Assistant", address: "assistant@acme.saas.com" },
      to: "grace@example.com",
      subject: "Your meeting on Thursday",
      text: "Confirming 10:00 at the office. Reply here if that changes.",
    })

    console.log(email.space_id) // "spc_..."
    ```
    ```bash curl
    curl https://api.aiinbx.com/api/v2/emails \
      -H "Authorization: Bearer $AI_INBX_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "from": "Acme Assistant <assistant@acme.saas.com>",
        "to": "grace@example.com",
        "subject": "Your meeting on Thursday",
        "text": "Confirming 10:00 at the office. Reply here if that changes."
      }'
    ```
    The email and its thread carry `space_id: "spc_..."`. A customer's request to send goes through your product, which checks that the customer owns the address and then makes this call.

6. **Read the space off the webhook**

    Grace's reply arrives on the wildcard's MX, reaches `acme.saas.com`, and is threaded inside Acme's space. Every event names it on the envelope:

    ```json
    {
      "id": "evt_...",
      "type": "email.received",
      "created_at": "2026-09-03T10:31:04Z",
      "space_id": "spc_...",
      "data": {
        "email_id": "eml_...",
        "thread_id": "thr_...",
        "domain_id": "dom_...",
        "mailbox_id": null,
        "from": "grace@example.com",
        "to": ["assistant@acme.saas.com"],
        "delivered_to": ["assistant@acme.saas.com"],
        "subject": "Re: Your meeting on Thursday",
        "snippet": "Works for me…",
        "category": "human"
      }
    }
    ```
    Your one endpoint receives every space's events; `space_id` is how it finds the customer, and `domain_id` / `mailbox_id` say which of the customer's identities the mail went through. A customer who wants events in a system of their own gets them from your handler. See [Webhooks](../webhooks.md#spaces).

    ```ts
    if (event.type === "email.received" && event.data.category === "human") {
      const customer = await customerBySpace(event.space_id)
      await queueReply(customer, event.data.thread_id)
    }
    ```
7. **They bring their own domain**

    A customer who wants to write from `acme.com` goes through the ordinary [domain flow](domains.md) — records published at their registrar — with the domain created in their space.

    ```ts TypeScript
    const own = await aiinbx.domains.create({
      name: "acme.com",
      space_id: space.id,
    })

    // Show `own.records` to the customer; wait for domain.verified.
    ```
    ```bash curl
    curl https://api.aiinbx.com/api/v2/domains \
      -H "Authorization: Bearer $AI_INBX_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{ "name": "acme.com", "space_id": "spc_..." }'
    ```
    `domain.verified` arrives with `space_id: "spc_..."`, so the same handler flips the right customer's onboarding step.

8. **They connect Gmail**

    For the assistant to answer from a person's existing inbox, connect the mailbox into the space. Only the server-side call takes `space_id`: the hosted page and a plain connect link are opened with no credential, so they cannot name a space — anyone could put a mailbox into another customer's. The URL this call returns is bound to the space, which is why a platform mints it per customer and redirects to it, rather than handing out the bare hosted link.

    ```ts TypeScript
    const { url } = await aiinbx.mailboxes.connect({
      provider: "google",
      app_id: "app_...",
      space_id: space.id,
      return_to: "https://app.saas.com/settings/mailboxes",
      ref: "user_8812",
    })
    // Send the customer to `url`.
    ```
    ```bash curl
    curl https://api.aiinbx.com/api/v2/mailboxes/connect \
      -H "Authorization: Bearer $AI_INBX_API_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "provider": "google",
        "app_id": "app_...",
        "space_id": "spc_...",
        "return_to": "https://app.saas.com/settings/mailboxes",
        "ref": "user_8812"
      }'
    ```
    [`mailbox.connected`](https://docs.aiinbx.com/webhooks/events/mailbox-connected.md) fires with the same `space_id`, and from then on mail synced from that account is Acme's.

## Single addresses of one domain

A subdomain per customer gives each customer a name of their own, and every address under it. When customers share one domain instead — `anna@agents.com` for Acme, `bob@agents.com` for Globex — assign each address to its customer's space. Nothing changes in DNS and nothing is verified: the domain still signs and carries the mail, and the address's space decides everything else.

Assign single addresses when each customer, or each agent, has one or a few addresses on a domain you already have; use a subdomain when a customer needs a name of their own or any number of addresses. See [Which to give each customer](#which-to-give-each-customer).

```ts TypeScript
const assigned = await aiinbx.addresses.assign({
  address: "anna@agents.com",
  space_id: space.id,
})

console.log(assigned.space_id, assigned.domain_id) // "spc_...", the domain's "dom_..."

// Where one address is assigned: a page with it, or none. One not listed is
// where its untagged address is assigned, else in its domain's space.
const [where] = await aiinbx.addresses
  .list({ address: "anna@agents.com" })
  .all()

// One customer's addresses.
const acme = await aiinbx.addresses.list({ space: space.id }).all()

// Back in the domain's space.
await aiinbx.addresses.remove("anna@agents.com")
```
```bash curl
curl https://api.aiinbx.com/api/v2/addresses \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "address": "anna@agents.com", "space_id": "spc_..." }'

curl "https://api.aiinbx.com/api/v2/addresses?address=anna%40agents.com" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"

curl -X DELETE https://api.aiinbx.com/api/v2/addresses/anna%40agents.com \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
```json
{
  "address": "anna@agents.com",
  "space_id": "spc_...",
  "domain_id": "dom_...",
  "created_at": "2026-11-11T09:12:41Z"
}
```
The rules:

- **The most specific wins.** An address assigned on its own is in its space; every other address on the domain is in the domain's space. A plus-tagged address — `anna+orders@agents.com` — goes where `anna@agents.com` goes, unless it is assigned itself.
- **Assigning is optional.** An address that is not assigned works as before, in its domain's space.
- **Assigning again moves it.** Mail it already has stays in the space it was in, with that space's suppression lists. Sends from it still queued or scheduled fail with [`sender_unknown`](https://docs.aiinbx.com/reference/errors.md#sender_unknown), as they do when a domain moves. Mail already on its way while it moves stays in the space it was headed to.
- **An agent login stays where it connected.** One already connected to the address stays on the old space: it keeps reading the mail there and its sends are refused. Connect it again for the new space.
- **Removing the assignment** puts the address back where its untagged address is assigned, else in its domain's space, from its next email on.
- **An assignment counts under the domain entry it was made on.** [Moving the domain](domains.md#moving-a-domain-between-spaces-and-workspaces) to another space adds a new entry for the name. Once the next verification check moves sending and receiving to it, every address assigned on the old entry is back in the domain's space until it is assigned again. `GET /addresses` keeps listing those assignments until the old entry is deleted, which removes them.
- **`space_id: null`** assigns it to the workspace itself.
- **Events and emails carry the address's space.** `space_id` is the address's; `domain_id` stays the domain it is on.
- **Sending follows the space too.** Sender reputation, pacing rules and suppression lists are those of the address's space.

The domain has to be the workspace's and verified: an address on no domain of the workspace fails with `sender_unknown`, one on a domain still waiting on DNS with `409` [`domain_unverified`](https://docs.aiinbx.com/reference/errors.md#domain_unverified). Deleting the domain entry an assignment was made on, or the space it is assigned to, takes the assignment with it. In the console, assigned addresses are under **Domains → Addresses**, with **Assign address**, **Move** and **Unassign**.

## What else a space holds

Everything below is optional — a workspace-level resource keeps working for every space. Putting one in a space narrows it.

| Resource                                        | In a space                      | At the workspace (`space_id: null`)                                                       |
| ----------------------------------------------- | ------------------------------- | ----------------------------------------------------------------------------------------- |
| [Pacing rule](pacing.md#spaces)            | Reads that space's mail alone.  | Applies to every space, and stacks on top — your cap holds over whatever a customer sets. |
| [Suppression list](suppressions.md#spaces) | Checked for that space's sends. | Checked for everyone's.                                                                   |

[API keys](../authentication.md) and [webhook endpoints](../webhooks.md#spaces) belong to the workspace: a key reaches every space and an endpoint receives every space's events.

Suppressions have one asymmetry worth knowing: a complaint on a send from a space lands on the space's list; a bounce that shows the address is dead lands on the workspace's, because the address exists for nobody.

## One reputation per space

Each space sends on a reputation of its own. Bounces and complaints are counted against the space the mail left from, and a space whose rates cross the line is paused by itself: nothing else in the workspace, and no other space, is slowed or stopped by it. A workspace without spaces is one reputation, so this is what spaces add for a platform — a customer who mails a bad list is the only one who pays for it. A pause has no event of its own yet: you see it as that space's sends failing, until it lifts.

This holds for subdomains under your wildcard as well as for domains a customer brings: `acme.saas.com` bounces against Acme's space, never against `*.saas.com`. What spaces cannot divide is what mailbox providers see. Gmail and Outlook judge the domain the mail is signed with, and every subdomain of `*.saas.com` is signed as `saas.com` — one domain to them. A customer who wants a reputation that is wholly theirs, in every sense, connects [their own domain](domains.md).

## Listing by space

Every list of resources that live in a space takes `space`:

```ts TypeScript
for await (const thread of aiinbx.threads.list({ space: space.id })) {
  console.log(thread.subject)
}

const domains = await aiinbx.domains.list({ space: space.id }).all()

// Domains at the workspace itself, excluding domains in customer spaces:
const workspaceDomains = await aiinbx.domains.list({ space: "none" }).all()
```
```bash curl
curl "https://api.aiinbx.com/api/v2/threads?space=spc_..." \
  -H "Authorization: Bearer $AI_INBX_API_KEY"

curl "https://api.aiinbx.com/api/v2/domains?space=none" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Omit it to include the workspace itself and every space. Pass `none` for the workspace's own resources only — the API equivalent of **Workspace only** in the console. Naming a space that is not the workspace's is `404`.

## Finding a space by your id

When you hold a customer and not the space — a job that runs per tenant, a support tool, a migration — look it up by the `external_id` you set:

```ts TypeScript
const [space] = await aiinbx.spaces.list({ external_id: "cus_8812" }).all()
```
```bash curl
curl "https://api.aiinbx.com/api/v2/spaces?external_id=cus_8812" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
The match is exact, and the page holds one space or none. A space that never got an `external_id` reads `null` and cannot be found this way; set one with `spaces.update`.

## Updating and deleting

```ts
await aiinbx.spaces.update(space.id, { name: "Acme Corp" })
await aiinbx.spaces.update(space.id, { external_id: "cus_9001" })
await aiinbx.spaces.update(space.id, { external_id: null }) // clears it
await aiinbx.spaces.update(space.id, { thread_across_addresses: true })
await aiinbx.spaces.delete(space.id)
```
Only the fields given change, and an update moves nothing. `thread_across_addresses` lets an email without reply headers join any matching conversation in the space, not only one the address it arrived at takes part in; it starts off, and [Threads](threads.md#which-thread-incoming-mail-joins) explains when to turn it on. Deleting returns `204` once the space is marked for deletion and its teardown is queued. It immediately stops accepting work and disappears from the API; in the background, its domains come off sending and verification, its mailboxes are disconnected, and its emails, threads, rules and suppressions are removed. There is no undo.

A wildcard in the space goes with its subdomains, including any in _other_ spaces — they are names under its identity, and nothing under it survives it.

## Routing rules of thumb

- **One key, on your servers.** It names a space with `space_id` on creates and `space` on lists, and sees everything. Keys are never per customer: a customer who needs the API goes through your product, which knows what they own.
- **Never `bounces.<wildcard>`.** That label is the wildcard's return path and is refused with [`reserved_subdomain`](https://docs.aiinbx.com/reference/errors.md#reserved_subdomain).
- **One endpoint, branch on `space_id`.** Endpoints are yours, like keys; a customer who needs their own delivery target gets it from your handler.
- **Workspace rules are your guardrails.** A pacing rule with no `space_id` caps every customer at once; a customer's own rule can only tighten it further.
- **`space_id: null` means "the workspace", not "unassigned".** A resource created without a space stays there; there is no move.

## Next

**[Domains](domains.md#wildcards-and-subdomains)**

Wildcards, subdomains, and the two errors they can raise.

**[Webhooks](../webhooks.md#spaces)**

`space_id` on the envelope, and one endpoint for every space.

**[Conventions](https://docs.aiinbx.com/reference/conventions.md#spaces)**

The `space_id: null` rule and the `space` list filter.
