# Domains

Add a sending domain, publish the DNS records, verify it, diagnose the one record that won't resolve, and cover every subdomain at once with a wildcard.

From https://docs.aiinbx.com/guides/domains

Adding a domain makes every address on it usable for sending, and — if you publish the MX record — for receiving too. Adding one returns the DNS records to publish; verification checks that they resolve.

The domain does not have to be yours. A multi-tenant product adds a customer's domain, shows them the returned records to publish at their registrar, and polls `verify` or waits for [`domain.verified`](https://docs.aiinbx.com/webhooks/events/domain-verified.md). The only requirement is that _someone_ can publish DNS for it. Where nobody can, connect a [mailbox](mailboxes.md) instead.

## Your provided domain

Every workspace comes with one domain: `<slug>.aiinbx.app`, where `<slug>` is the workspace's URL handle — like a `.vercel.app` domain. It works right away, every address on it sends and receives, and there is no DNS to set up. Handy for a side project, or for getting going before you bring a domain of your own.

```ts
const domains = await aiinbx.domains.list().all()
const provided = domains.find((domain) => domain.provided)

await aiinbx.emails.send({
  from: `hello@${provided.name}`,
  to: "someone@example.com",
  subject: "Sent before a single DNS record",
  text: "It works out of the box.",
})
```
It is listed with `provided: true`, has no `records`, and cannot be deleted — it goes when the workspace does. Renaming the workspace does not rename it. A name under `aiinbx.app` cannot be added by hand, and fails with [`reserved_domain`](https://docs.aiinbx.com/reference/errors.md#reserved_domain).

Mail from it is capped — 100 a day, 20 an hour, 10 a day to any one recipient — by pacing rules that come with the workspace and cannot be changed; see [the provided domain's ceilings](pacing.md#the-provided-domains-ceilings). Bring a domain of your own for anything beyond that.

## Adding a domain

```ts TypeScript
const domain = await aiinbx.domains.create({
  name: "example.com",
  region: "eu-central-1",
})

for (const record of domain.records ?? []) {
  console.log(
    record.purpose,
    record.type,
    record.name,
    record.value,
    record.ttl
  )
}
```
```python Python
domain = client.domains.create({"name": "example.com", "region": "eu-central-1"})

for record in domain.get("records", []):
    print(record["purpose"], record["type"], record["name"], record["value"])
```
```bash curl
curl https://api.aiinbx.com/api/v2/domains \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "example.com", "region": "eu-central-1" }'
```
Creating, updating, and deleting domains needs a `full` [scope](../authentication.md#scopes) key.

### Regions

`region` selects the regional mail infrastructure for this domain: `eu-central-1` (the default) or `us-east-1`. Two of the records it owes name the region — the inbound MX and the return-path MX — so it is chosen before they are published. Until the domain verifies it can be moved with `PATCH /v2/domains/:id` and `{ "region": "us-east-1" }`: those two records are re-issued and start over as pending, DKIM stays as it is. Once verified the region is fixed (`409` [`region_fixed`](https://docs.aiinbx.com/reference/errors.md#region_fixed)); moving a verified domain means deleting and re-adding it, and republishing DNS. A subdomain is handled wherever its wildcard is.

A domain carried over from the previous aiinbx reports `us-east-1`, where that account handles its mail, until its records verify here; the region it moves to is chosen when the move is started in the console, and until it verifies the same `PATCH` changes it — `region` in the response keeps reading `us-east-1` meanwhile, the records carry the choice. Do not treat this setting alone as a guarantee that all application data, logs, or subprocessors stay in that region. Confirm your residency requirements separately before choosing a region.

Adding the same identity twice in one space fails with `409` [`domain_taken`](https://docs.aiinbx.com/reference/errors.md#domain_taken). That includes the apex and wildcard forms of one name, such as `example.com` and `*.example.com`. See [Moving a domain between spaces and workspaces](#moving-a-domain-between-spaces-and-workspaces).

### Spaces

`space_id` puts the domain in a [space](spaces.md), and with it every email and thread that goes through the domain from then on. Omit it and the domain belongs to the workspace itself, `space_id: null`. There is no move afterwards; a space is decided when the domain is created.

## The records

`records` lists everything to publish, each with the `purpose` it serves and a `state` of `pending`, `verified`, or `missing`.

| Purpose       | Type  | What it does                                                                                         |
| ------------- | ----- | ---------------------------------------------------------------------------------------------------- |
| `SPF`         | `TXT` | Authorizes AI Inbx to send for the domain.                                                           |
| `DKIM`        | `TXT` | Publishes the keys that sign your outgoing mail.                                                     |
| `DMARC`       | `TXT` | Tells receivers what to do when SPF or DKIM fails, and where to send reports.                        |
| `RETURN_PATH` | `MX`  | Routes bounces and complaints back so delivery status is accurate.                                   |
| `INBOUND`     | `MX`  | Delivers mail _to_ the domain into AI Inbx. Only needed if you want to [receive](receiving.md). |

> [!WARNING]
> **The MX record is a commitment**
>
> Publishing the `INBOUND` MX record routes **all** mail for the domain to AI Inbx. If the domain already receives mail somewhere — a Google Workspace or Microsoft 365 tenant, say — that takes the existing inboxes away. Use a dedicated subdomain (`mail.example.com`) for inbound instead, or connect the existing mailbox over OAuth. See [Mailboxes](mailboxes.md).
>
> This is the single most important thing to get right when you're onboarding someone else's domain: they will almost always already be receiving mail on it.

Publish the exact names, types, and values returned by the API. Check existing records before making changes, especially an existing DKIM selector or DMARC policy. Sending requires DKIM verification; the return-path, SPF, and DMARC records configure authentication and bounce handling. Receiving separately requires the inbound MX record.

## Verifying

Publishing DNS doesn't tell us anything by itself, so ask for a check:

```ts TypeScript
const domain = await aiinbx.domains.verify("dom_...")

if (domain.verified_at) {
  console.log("ready to send")
} else {
  const missing = domain.records?.filter((r) => r.state !== "verified")
  console.log(
    "still waiting on",
    missing?.map((r) => r.purpose)
  )
}
```
```python Python
domain = client.domains.verify("dom_...")
print(domain["verified_at"])
```
```bash curl
curl -X POST https://api.aiinbx.com/api/v2/domains/dom_.../verify \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
`verified_at` is `null` until the sending identity’s DKIM verification succeeds. It does not certify inbound MX, SPF, or DMARC configuration. Each record's `last_checked_at` tells you when it was last looked at. Propagation is usually minutes and occasionally hours — poll with a sane interval rather than in a tight loop.

Sending from an unverified domain fails with `409` [`domain_unverified`](https://docs.aiinbx.com/reference/errors.md#domain_unverified).

## Diagnostics

When a domain won't verify or a record won't pass, `diagnostics` reads the live zone and reports what it finds: each record the zone doesn't hold — what it's for, what its absence costs, and the exact record to add — and anything else in the zone in the way, like a CNAME on the name or a second SPF record. It's a read-only check — nothing is stored, so it always reflects DNS as it is right now, even where a record's `state` from the last check is behind.

```ts TypeScript
const { data } = await aiinbx.domains.diagnostics("dom_...")

for (const finding of data) {
  console.log(`[${finding.severity}] ${finding.title}`)
  console.log(finding.detail)
  console.log("fix:", finding.fix)
}
```
```python Python
findings = client.domains.diagnostics("dom_...")["data"]

for finding in findings:
    print(finding["severity"], finding["title"], finding["fix"])
```
```bash curl
curl https://api.aiinbx.com/api/v2/domains/dom_.../diagnostics \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Each finding carries a stable `id`, a `severity`, a one-line `title`, a `detail` explaining what's wrong, a `fix` describing what to change, and `evidence` — the actual records read from the zone. Findings about one of the domain's records also carry the `purpose` of that record; findings about the zone as a whole have none.

| Prop | Type | Default | Description |
| - | - | - | - |
| `breaking` | `severity` | - | Mail will fail or be rejected. Fix before sending. |
| `risk?` | `severity` | - | Deliverability is likely to suffer — an SPF record that lets anyone send as you (`+all`), a DMARC record without a `p=` policy. |
| `tip?` | `severity` | - | A hardening suggestion. Nothing is broken. |

Findings also carry `when`: `adding` applies while you're setting the domain up, `added` to one already live, and `both` either way.

The classic breaking finding is two SPF records on the same name — receivers treat that as a permanent error and ignore both. Diagnostics names the offending records so you know which to merge.

## Wildcards and subdomains

A wildcard — `*.saas.com` — is one domain that covers every subdomain. Publish its records once and `anything.saas.com` can send and receive, without touching DNS again. It is how a platform hands each customer an address of their own.

```ts
const wildcard = await aiinbx.domains.create({ name: "*.saas.com" })
```
The records go in the `saas.com` zone, but the MX record sits at the wildcard rather than the apex — `saas.com` keeps whatever mail setup it already has, and only its subdomains route to AI Inbx:

| Purpose       | Type  | Name                       |
| ------------- | ----- | -------------------------- |
| `DKIM`        | `TXT` | `aibx._domainkey.saas.com` |
| `INBOUND`     | `MX`  | `*.saas.com`               |
| `RETURN_PATH` | `MX`  | `bounces.saas.com`         |
| `SPF`         | `TXT` | `bounces.saas.com`         |
| `DMARC`       | `TXT` | `_dmarc.saas.com`          |

`saas.com` and `*.saas.com` are the same identity on the mail provider, so only one of the two can exist here: adding the second fails with [`domain_taken`](https://docs.aiinbx.com/reference/errors.md#domain_taken). Pick the apex if you send from `saas.com` itself; pick the wildcard if the addresses are `<customer>.saas.com`.

> [!WARNING]
> **An existing wildcard CNAME**
>
> A name that holds a `CNAME` may hold nothing else. If `*.saas.com` is already a wildcard `CNAME` — customer sites on `<customer>.saas.com`, typically — the MX record cannot be published beside it. Use a level down, `*.mail.saas.com`, and give customers `<customer>.mail.saas.com`. `diagnostics` reports that alias on a wildcard domain as a `wildcard-cname` finding, which is the sign to look for.

### Subdomains

Once a wildcard is yours, a name one label under it — `acme.saas.com` under `*.saas.com` — is created as a **subdomain** of it:

```ts TypeScript
const domain = await aiinbx.domains.create({
  name: "acme.saas.com",
  space_id: "spc_...",
})

console.log(domain.parent_id) // the wildcard's id
console.log(domain.records) // []
```
```python Python
domain = client.domains.create({"name": "acme.saas.com", "space_id": "spc_..."})
print(domain["parent_id"], domain["verified_at"])
```
```bash curl
curl https://api.aiinbx.com/api/v2/domains \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "acme.saas.com", "space_id": "spc_..." }'
```
A subdomain has no records of its own and nothing to verify — `verified_at` is set the moment the wildcard's is, and follows it. `parent_id` names the wildcard. `region` and the tracking flags are inherited. It is usually created in a [space](spaces.md), so that the customer's mail is theirs — and so is their sending reputation: a subdomain that bounces is paused with its space, and the other names under the wildcard keep sending. Mailbox providers still see one domain for all of them; see [One reputation per space](spaces.md#one-reputation-per-space).

What is not a subdomain: `a.b.saas.com` is two labels down and is an ordinary domain with records of its own, and so is `acme.saas.com` in a workspace that has no `*.saas.com` yet.

One name is refused: `bounces.saas.com` is the wildcard's return path, and fails with `409` [`reserved_subdomain`](https://docs.aiinbx.com/reference/errors.md#reserved_subdomain).

### Deleting

A subdomain is a row: deleting it unpublishes nothing. Deleting a wildcard deletes every subdomain filed under it, in whatever [space](spaces.md) they sit — they are names under its identity and its records, and nothing is left to publish for them. Mail already sent or received is kept either way. Remove DNS records that belonged to the deleted connection after checking that they are no longer needed.

## Tracking settings

`PATCH /domains/{id}` toggles open and click tracking:

```ts
await aiinbx.domains.update("dom_...", {
  track_opens: false,
  track_clicks: true,
})
```
The values are returned on the domain as `tracking.opens` and `tracking.clicks`. Both start off, unless the workspace's tracking defaults say otherwise — they apply to every new domain, however it was added, your [provided domain](#your-provided-domain) included. You set them in the console.

Both can also be set when the domain is created; what the request leaves out follows the defaults:

```ts
await aiinbx.domains.create({ name: "acme.dev", track_opens: true })
```
They are what every send from the domain does unless it says otherwise:

- **Opens** adds a 1×1 pixel to the HTML body, and reports each load as [`email.opened`](https://docs.aiinbx.com/webhooks/events/email-opened.md). A text-only email has nowhere to carry it.
- **Clicks** rewrites the links in the HTML body through a tracking host, and reports each as [`email.clicked`](https://docs.aiinbx.com/webhooks/events/email-clicked.md). Recipients see that host when they hover a link, not yours.

With both off, the mail goes out exactly as you wrote it.

The setting is read when a message leaves, not when it was accepted: a change applies from the next email out, including ones already [scheduled](https://docs.aiinbx.com/guides/scheduling.md) or held by [pacing](pacing.md). A subdomain starts with its wildcard's setting, except where its create request says otherwise, and is changed on its own from there.

One email can differ from its domain — a password reset from a domain that tracks its newsletters, say. See [tracking per send](https://docs.aiinbx.com/guides/sending.md#tracking).

> [!NOTE]
> Tracking is something a domain does. Mail sent through a [connected mailbox](mailboxes.md) leaves through Gmail or Outlook as written, and is never tracked.

## Listing and deleting

```ts TypeScript
const domains = await aiinbx.domains.list().all()

await aiinbx.domains.delete("dom_...")
```
```python Python
for domain in client.domains.iter():
    print(domain["name"], domain["verified_at"])

client.domains.delete("dom_...")
```
Deleting a domain stops future sends from it. Remove the DNS records afterwards — an SPF or DKIM record pointing at a domain that no longer exists is dead weight in your zone. The [provided domain](#your-provided-domain) cannot be deleted: the call fails with [`domain_provided`](https://docs.aiinbx.com/reference/errors.md#domain_provided).

## Knowing when it's ready

Rather than polling, subscribe to [`domain.verified`](https://docs.aiinbx.com/webhooks/events/domain-verified.md):

```ts
if (event.type === "domain.verified") {
  console.log(event.data.name, "is ready in", event.data.region)
}
```
## Staying verified

Verification is not a one-off. Domains are rechecked in the background. When a verified domain's DKIM record disappears from DNS — two of its last three checks miss it, even with one between that found it — the domain keeps sending for 72 hours and is rechecked every hour; the domain's `dkim_missing_since` says when the record went missing and `stops_sending_at` when it stops sending (a subdomain shows its wildcard's), the console marks it with that time, [`domain.at_risk`](https://docs.aiinbx.com/webhooks/events/domain-at-risk.md) is sent, and the workspace is mailed straight away. Receivers can't check the signature in the meantime, so mail from it may be rejected or land in spam. Put the record back within the 72 hours and it is rechecked every 15 minutes; once every check for an hour has found it, [`domain.restored`](https://docs.aiinbx.com/webhooks/events/domain-restored.md) is sent and nothing else changes. A check that misses it in that hour starts the hour again, and the 72 hours run on from the first miss. Both events are sent for the domain that holds the records, not for a wildcard's subdomains.

If it has not been back for that hour when the 72 hours are up, the domain loses `verified_at`: sends from it fail with [`domain_unverified`](https://docs.aiinbx.com/reference/errors.md#domain_unverified) until the record is back, and [`domain.lost`](https://docs.aiinbx.com/webhooks/events/domain-lost.md) tells you it happened. The same happens at once if the mail provider stops signing for the domain. Nothing else changes — the domain, its records and its mail stay where they were, and it is verified again once the key has been back for an hour.

Domains that are missing their record or were lost are also mailed to the workspace's owners and admins: what changed since their last mail, at most one an hour: a domain whose record went missing, the same domain again in its last day before it stops sending, when it stops, and one they were told about working again. Each gives one time to publish the record by, 75 minutes before sending stops. Domains in a [space](spaces.md) are your customers': they come in one summary a day at 08:00 UTC, since your webhook already had each change, with their last warning two days ahead so a summary always carries it. When 20 or more go missing together, the summary goes early, once, after we have read the rest of your domains. Each admin picks Everything, Workspace only (no daily summary), or Off under Settings → Notifications, or from the link at the bottom of any of these mails.

## Moving a domain between spaces and workspaces

A domain name can be connected in several spaces and workspaces, but only the connection whose DKIM key is currently published sends and receives for it. Add the domain where it should go (another workspace, or another customer's space when a lapsed name was bought by someone else), then publish the DNS records returned by that connection. The next verification check moves sending and receiving to it and marks the previous connection as lost. Existing mail stays where it was sent or received; scheduled and queued mail from the previous connection fails instead of going out through the new one. A name held in several spaces of one workspace counts once against the plan.

Within one space, the apex and wildcard are one connection: `example.com` and `*.example.com` cannot both be added there. Where several need to send at the same time, give each one a distinct subdomain.
