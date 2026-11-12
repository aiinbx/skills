# Mailboxes

Connect existing Gmail or Outlook accounts over OAuth from a hosted page, a link or the API, keep them synced, and brand the consent screen.

From https://docs.aiinbx.com/guides/mailboxes

A **mailbox** is an existing Gmail or Outlook account, connected once over OAuth. Its mail keeps flowing through Google or Microsoft exactly as before; AI Inbx syncs it, and can send and receive as that address.

That's the difference from a [domain](domains.md), which needs DNS access and takes over mail for every address on it. A mailbox needs nothing but the account holder clicking approve, changes nothing about their existing setup, and covers one account — its address and any [aliases](#aliases) it sends as.

Reach for it when the address already has an inbox somewhere — a rep's work account, a founder's inbox — and the mail should read as coming from that person.

## Connecting a mailbox

Three ways in, ordered by how much of the flow you own. They all end the same way: the person approves at Google or Microsoft, the mailbox lands in your workspace, and [`mailbox.connected`](https://docs.aiinbx.com/webhooks/events/mailbox-connected.md) fires.

| | You write | Your users see | Needs |
| --- | --- | --- | --- |
| [Hosted page](#hosted-page) | Nothing — share a link | A page in your brand, then "you're connected" | Your own [OAuth app](#your-own-oauth-app) |
| [Connect link](#connect-link) | A URL on a button | Only the provider's consent screen, then your page | Your own OAuth app, plus a registered return URL |
| [API](#api) | A server-side call per connection | Only the provider's consent screen, then your page | Your own OAuth app, plus a `full` API key |

### Hosted page

Every OAuth app of yours has a page at its `connect_url` — `https://aiinbx.com/connect/<slug>` — wearing your name, logo, accent and support address, with nothing of AI Inbx on it. Send people there, or put it in an email; it's a permanent link, not a one-time one.

```
https://aiinbx.com/connect/acme?ref=user_8812
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `ref?` | `string` | - | Your own identifier for the person, up to 200 characters. Kept on the mailbox and carried on every mailbox.* event. |
| `return_to?` | `string` | - | One of the app's registered return URLs. Adds a "Continue to <your app>" button to the page they end on, carrying the same query as a connect link does. |

Brand the page from the app's settings in the console, or through `oauthApps.create` — `accent`, `tagline`, `support_email`, `logo_url`.

### Connect link

The same handoff with no page of ours in it. The person clicks a button on your site, lands on the consent screen, and comes back to a page of yours. Build the URL yourself; there's nothing to call first:

```
https://aiinbx.com/connect/<slug>/start?return_to=<url>&ref=<id>
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `return_to` | `string` | - | Where they land afterwards. Must match one of the app's return_urls on origin and path; the query is yours to fill. |
| `ref?` | `string` | - | Your own identifier for the person, up to 200 characters. Kept on the mailbox and carried on every mailbox.* event. |

1. **Register where the link may return to**

    A connect link is opened with no credential, so it will only send someone back to a page you've listed on the app — the same idea as the redirect URIs you register with Google. Add them in the console, or on the app:

    ```ts
    await aiinbx.oauthApps.update(app.id, {
      return_urls: [
        "https://app.example.com/settings/mailboxes",
        "http://localhost:3000/settings/mailboxes",
      ],
    })
    ```
    Entries are matched on origin and path, so one covers every query string that page is opened with. `https` only, except on `localhost`. An unregistered `return_to` fails with [`return_url_not_registered`](../reference/errors.md#return_url_not_registered) before anyone reaches the provider.

2. **Put the link on a button**

    ```tsx React
    const connectUrl = new URL(`https://aiinbx.com/connect/acme/start`)
    connectUrl.searchParams.set("return_to", "https://app.example.com/settings/mailboxes")
    connectUrl.searchParams.set("ref", user.id)

    <a href={connectUrl.href}>Connect your Gmail</a>
    ```
    ```html HTML
    <a href="https://aiinbx.com/connect/acme/start?return_to=https%3A%2F%2Fapp.example.com%2Fsettings%2Fmailboxes&ref=user_8812">
      Connect your Gmail
    </a>
    ```
    Links are counted per IP address as they're opened: 30 a minute, far more than a person retrying needs. Past that the link answers `429` `too_many_requests` with `Retry-After`.

3. **Read the outcome off the query**

    They come back to `return_to` with the result appended:

    | Query | When | Value |
    | --- | --- | --- |
    | `mailbox` | Connected | The new mailbox's `mbx_…` id |
    | `connected` | Connected | The address they authorized |
    | `error` | Declined or failed | What went wrong, e.g. `access_denied` |

    Fine for showing a "connected" state on the page. Not proof that it is live — see the last step. It is where you decide whose mailbox it is, though: `return_to` is a page of yours, opened with your own session.

    ```ts
    // On return_to. The user is whoever your session says, not who the URL says.
    const mailboxId = new URL(request.url).searchParams.get("mailbox")
    if (mailboxId) await claimMailbox(session.user.id, mailboxId)
    ```
4. **Wait for mailbox.connected**

    The webhook is the authoritative signal that the mailbox is live. The browser landing on `return_to` only says it came back; a person who closes the tab at the consent screen never triggers it.

    ```ts
    if (event.type === "mailbox.connected") {
      await markMailboxReady(event.data.mailbox_id)
      // `reconnected` is true when this replaced an existing authorization.
    }
    ```
    The event carries `ref` too, but on a connect link that `ref` came off a URL anyone can write — somebody can send one of your users a link with their own id in it. Treat it as unauthenticated input: a handle to check against the session that came back, never what decides ownership on its own. For the same reason a `ref` from a connect link or the hosted page fills in a mailbox that has none and never replaces one that is set. When the webhook alone should decide, mint the URL with [`mailboxes.connect`](#api) — a `ref` set there is vouched for by your API key.

### API

For when your server should decide — the region, how much history to import, the [space](spaces.md). The URL it returns is short-lived and single-use — good for ten minutes, and spent the moment the person approves: redirect to it, don't email it. Declining doesn't spend it, so they can go back and try again.

```ts
const { url } = await aiinbx.mailboxes.connect({
  provider: "google",
  app_id: "app_...",
  return_to: "https://app.example.com/settings/mailboxes",
  ref: "user_8812",
  metadata: { tenant: "acme", plan: "pro" },
  backfill_days: 30,
})
// Send the customer to `url`.
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `provider` | `"google" \| "microsoft"` | - | Which provider to authorize against. |
| `return_to` | `string` | - | Where the customer lands after approving or declining. Any URL — the call is authenticated, so it isn't held to the app's return_urls. |
| `ref?` | `string` | - | Your own identifier for the person, up to 200 characters. Kept on the mailbox and carried on every mailbox.* event, so you know which user it is about. |
| `metadata?` | `Record<string, string>` | - | Anything else you want kept with the mailbox — see below. Up to 50 keys; a key is 1–40 characters, a value up to 500. |
| `app_id` | `string` | - | The OAuth app to connect through, registered for the same provider — see below. Without one the call fails with shared_apps_disabled. |
| `space_id?` | `string` | - | The space the mailbox lands in, and with it everything synced from it. Omit for the workspace itself. |
| `region?` | `"eu-central-1" \| "us-east-1"` | - | Where this mailbox's mail is processed and stored. Defaults to eu-central-1. |
| `backfill_days?` | `number` | - | How much history to import on first sync, 0–90 days. Defaults to 7, the same as the hosted page and a connect link. |

The return carries the same query as a connect link, and the same rule applies: wait for `mailbox.connected` rather than trusting the redirect. A URL that has expired or been used fails with [`expired_state`](../reference/errors.md#expired_state) — as a JSON `400` on our callback page, not as a redirect: what knew the `return_to` is gone with it. Creating connection URLs requires a `full` [scope](https://docs.aiinbx.com/authentication.md#scopes) key.

`space_id` is the reason a platform uses this path: it is the only one of the three that can put the mailbox in a [space](spaces.md). The hosted page and a connect link are opened with no credential, so they cannot take a space — anyone could put a mailbox into another customer's. A mailbox lands in a space only through this call, which binds the space to the URL it returns. So a platform mints the connect URL server-side, per customer, and never hands out the bare hosted link.

### Ref and metadata

`ref` is one identifier — your user's id — and the thing to match on. `metadata` is for everything else you'd otherwise have to look up when an event arrives: the tenant, the plan, which of your products the mailbox belongs to. Both are stored on the mailbox, returned wherever it is, and carried on every `mailbox.*` event — `mailbox.needs_reauth` six months later included.

```ts
if (event.type === "mailbox.needs_reauth") {
  const { ref, metadata } = event.data
  await promptReconnect({ userId: ref, tenant: metadata.tenant })
}
```
If you've been JSON-encoding context into `ref` to get it through, stop: it's capped at 200 characters and you have to parse it back out. Put the id in `ref` and the rest in `metadata`.

`metadata` is strings to strings, never read by AI Inbx, and always written whole — there is no merging. Reconnecting a mailbox through a URL that carries `metadata` replaces what the mailbox had; one that carries none leaves it alone; `metadata: {}` clears it. A `ref` from `mailboxes.connect` replaces and is left alone the same way, but has no value that clears it: once set, a `ref` changes and never goes back to `null`. There is no `PATCH /mailboxes/{id}` and no `?ref=` filter on `GET /mailboxes`: both fields change only through a reconnect, and a mailbox is found by the id you kept.

Only `mailboxes.connect` takes `metadata`. The hosted page and a connect link carry `ref` and nothing more: they're URLs anyone can edit, which is fine for a handle you check against your own records, and not for data you'll take at its word. Their `ref` is held to that as well — it names a mailbox that has no `ref` yet, and never overwrites one.

## Mailbox state

```ts TypeScript
const mailbox = await aiinbx.mailboxes.retrieve("mbx_...")

console.log(mailbox.address, mailbox.state, mailbox.last_sync_at)
```
```python Python
mailbox = client.mailboxes.retrieve("mbx_...")
print(mailbox["address"], mailbox["state"])
```
| State | Meaning | What to do |
| --- | --- | --- |
| `active` | Syncing normally. | Nothing. |
| `needs_reauth` | The grant expired or was revoked — password change, admin policy, manual revocation. | Prompt the customer through the connect flow again. |
| `disconnected` | Removed, by you or by them. | Reconnect if they want it back. |

`state_reason` carries the provider's explanation when there is one. Both transitions are also webhooks — [`mailbox.needs_reauth`](https://docs.aiinbx.com/webhooks/events/mailbox-needs-reauth.md) and [`mailbox.disconnected`](https://docs.aiinbx.com/webhooks/events/mailbox-disconnected.md) — which is the right place to trigger a re-authorization prompt rather than discovering it on the next failed send. The workspace's owners and admins also get an email when a mailbox needs re-authorization, together with whatever else changed rather than one per mailbox, at most one an hour (a customer's mailbox in a [space](spaces.md) in the daily summary) — each of them at the level they picked under Settings → Notifications.

Sending from a mailbox that isn't `active` fails with `409` [`mailbox_inactive`](../reference/errors.md#mailbox_inactive).

## Aliases

A Gmail account can send as more addresses than its own — a work address added under **Send mail as**, say. Those come along with the mailbox: each verified one is listed in `aliases`, and works as a `from` like the account's own address. Mail written to an alias arrives in the account as it always did, on the mailbox's threads, and a reply without a `from` goes out as the alias the message was written to.

```ts
const mailbox = await aiinbx.mailboxes.retrieve("mbx_...")
console.log(mailbox.aliases) // [{ address: "me@acme.dev", name: "Ada" }]
```
The list is read from Google when the mailbox connects and hourly after that, so an alias added or removed there follows on its own; [`sync`](#syncing) reads it at once. Nothing is editable here — the provider is the source. An address that is both an alias and on one of your verified [domains](domains.md) sends through the mailbox, so the mail lands in the account's own Sent folder. Outlook accounts list no aliases.

## Syncing

Mail arrives on its own — Gmail through push notifications, Outlook through Graph subscriptions. `sync` forces a catch-up when you have a reason to think something was missed:

```ts TypeScript
await aiinbx.mailboxes.sync("mbx_...") // 202, queued
```
```python Python
client.mailboxes.sync("mbx_...")
```
```bash curl
curl -X POST https://api.aiinbx.com/api/v2/mailboxes/mbx_.../sync \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
It returns `202` with `{ "status": "queued" }` — the work happens in the background, and the messages surface as ordinary `email.received` events. It isn't a way to poll for mail; it's a repair tool.

## Disconnecting

```ts
const mailbox = await aiinbx.mailboxes.disconnect("mbx_...")
console.log(mailbox.state) // "disconnected"
```
Sync stops and the stored credentials are dropped. Messages already received stay on their threads.

## Your own OAuth app

Every mailbox connects through an OAuth app of your own, registered with Google or Microsoft. The consent screen, the hosted page and the connect link all carry its name — your product's — which is what a business customer expects to see before granting mailbox access.

```ts
const app = await aiinbx.oauthApps.create({
  provider: "google",
  name: "Acme Assistant",
  slug: "acme",
  client_id: process.env.GOOGLE_CLIENT_ID!,
  client_secret: process.env.GOOGLE_CLIENT_SECRET!,
  tagline: "Let Acme handle your inbox",
  support_email: "support@acme.com",
  return_urls: ["https://app.acme.com/settings/mailboxes"],
})

console.log(app.redirect_uri) // register this with the provider
console.log(app.connect_url) // the hosted page; + "/start?return_to=…" is the connect link
console.log(app.push_endpoint) // Google only — the Pub/Sub push endpoint
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `provider` | `"google" \| "microsoft"` | - | Which provider this app is registered with. |
| `name` | `string` | - | Shown to the customer during consent. |
| `slug` | `string` | - | 3–64 characters, unique across AI Inbx; forms the hosted connect URL. |
| `client_id` | `string` | - | From the provider's console. |
| `client_secret` | `string` | - | Stored encrypted; never returned. |
| `tenant?` | `string` | - | Microsoft only — restrict to a single Entra tenant. |
| `pubsub_topic?` | `string` | - | Google only — your own Pub/Sub topic for push notifications. |
| `accent?` | `string` | - | Accent color for the hosted connect page. |
| `tagline?` | `string` | - | One line shown under the app name. |
| `support_email?` | `string` | - | Where a confused customer should write. |
| `return_urls?` | `string[]` | - | Pages of yours a connect link may return to, up to 20. Matched on origin and path. |

1. **Register the app with the provider**

    A Google Cloud OAuth client, or an Entra app registration.

2. **Create it here and read back redirect_uri**

    Add that exact URI to the provider's allowed redirect list.

3. **Google only: create a Pub/Sub topic**

    Grant publish rights, then set `pubsub_topic`. Push notifications go to the
    returned `push_endpoint`. Each app needs a topic of its own.

4. **Connect through it**

    Share the hosted `connect_url`, put a [connect link](#connect-link) on your
    site, or pass `app_id` to `mailboxes.connect`.

`slug` is claimed globally; a taken one fails with `409` [`slug_taken`](../reference/errors.md#slug_taken). `mailboxes.connect` without an `app_id` fails with [`shared_apps_disabled`](../reference/errors.md#shared_apps_disabled).

Secrets can be rotated with `oauthApps.update`; the old value is replaced immediately, so update the provider first.

## Domains or mailboxes?

Not a question of whose address it is — either one can be yours or your customer's. A domain you add on a customer's behalf, walking them through the DNS records, is an ordinary domain. What differs is what you need from them and what you get.

| | [Domain](domains.md) | Mailbox |
| --- | --- | --- |
| Needs | Access to the domain's DNS | One person clicking approve |
| Covers | Every address on the domain | One existing inbox, and its aliases |
| Mail is handled by | AI Inbx | The existing Google or Microsoft account, synced |
| Setup | Publish records, wait for DNS | OAuth consent, live in seconds |
| Receiving | The `INBOUND` MX record | Provider sync |
| Breaks when | The records change or lapse | The grant is revoked |

The deciding question is usually whether the address already receives mail somewhere. An address on a live Google Workspace or Microsoft 365 domain has an inbox already, and pointing the MX record at AI Inbx would take that over for the whole domain — connect the mailbox instead. A domain with nowhere for its mail to go yet, or a subdomain created for the purpose, is the domain case.

Plenty of products use both, and often both on behalf of the same customer: a subdomain for anything the product sends, connected mailboxes for anything written from a person's own inbox.
