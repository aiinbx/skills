# Suppressions

Lists that stop the next send — bounces, complaints, unsubscribes — scoped per campaign or across the whole workspace.

From https://docs.aiinbx.com/guides/suppressions

A suppression is a standing instruction not to mail an address. Bounces that show an address is dead, complaints, and unsubscribes create them automatically; you can add your own. Every send is filtered against them before it goes anywhere, and the addresses that were dropped come back in the response's `suppressed` array.

This is deliverability plumbing, not a nicety: mailbox providers judge you on whether you keep sending to addresses that have already bounced or complained.

## Two axes

Suppressions have a **key** (which list) and a **scope** (how strongly).

### Keys

Every entry belongs to a list. `*` is the workspace-wide list; any other key is one you name.

```json
{ "suppression_key": "product-updates" }
```
A send that names `product-updates` is checked against **that list and `*`**. A send that names nothing is checked against `*` alone.

That gives you the behaviour you want without any bookkeeping: someone who opts out of `product-updates` stops getting product updates and keeps getting their receipts, while a bounce that shows the address is dead lands on `*` and stops everything.

- *  (workspace-wide)/
  - dead addresses, complaints on sends without a key, anything you never want mailed again
- product-updates/
  - opt-outs from this list only
- onboarding-drip/
  - opt-outs from this list only

Use one key per thing a person could reasonably opt out of separately. A tenant identifier works well when you're sending on behalf of your own customers.

### Scopes

| Scope | Blocks |
| --- | --- |
| `all` | Every send. The default, and what bounces and complaints produce. |
| `optional` | Only sends marked `unsubscribe: true`. Marketing stops; the password reset still arrives. |

`optional` is what an unsubscribe link should create. `all` is what a complaint must create.

### Spaces

A [space](spaces.md) keeps lists of its own. A send from a space is checked against its space's lists **and** the workspace's — so a platform's `*` list still stops everything for every customer, while a customer's opt-outs stay theirs.

| A send from… | Is checked against |
| --- | --- |
| The workspace (`space_id: null`) | The workspace's `*` and the workspace's `key` list |
| A space | The above, plus the space's `*` and the space's `key` list |

What the automatic entries do follows from that: a **complaint** on a send from a space lands on the space's list, under the send's key; a **bounce** that shows the address is dead lands on the workspace's `*` list, because the address exists for nobody and no other customer should try it either.

Add to a space's list with `space_id`. Entries carry `space_id`, `null` for the workspace's own, and `suppressions.list` takes `space`.

```json
{ "addresses": ["grace@example.com"], "key": "product-updates", "scope": "optional", "space_id": "spc_..." }
```
## Marking a send as optional

The `unsubscribe` flag on a [send](https://docs.aiinbx.com/guides/sending.md#suppression-and-unsubscribe) or [reply](threads.md#replying) declares the message non-essential. It adds one-click unsubscribe headers, and it opts the message into `optional` suppressions:

```ts
await aiinbx.emails.send({
  from: "Acme <news@example.com>",
  to: "grace@example.com",
  subject: "What shipped in August",
  html: newsletter,
  suppression_key: "product-updates",
  unsubscribe: true,
})
```
### Placing the link

Put `{{unsubscribe_url}}` in the `html` or `text` wherever the link belongs, and it's replaced with the recipient's personal unsubscribe URL. Two or three braces and spaces inside them all work, so it survives whatever template engine rendered the body. The placeholder alone marks the send as optional; `unsubscribe: true` isn't needed alongside it.

```html
<a href="{{unsubscribe_url}}">Unsubscribe from product updates</a>
```
Without a placeholder in either body, an `unsubscribe: true` message gets one appended instead: a small centred "Unsubscribe" link at the end of the `html`, and an `Unsubscribe: <url>` line at the end of the `text`. With a placeholder in one body, nothing is appended to the other — put it in both if you send both.

The unsubscribe link is personal to its recipient, so an optional send takes exactly one address across `to`, `cc` and `bcc`. More fails with `400` [`unsubscribe_single_recipient`](https://docs.aiinbx.com/reference/errors.md#unsubscribe_single_recipient) — send one message per recipient.

> [!WARNING]
> Don't set `unsubscribe: true` on transactional mail. It offers the recipient an opt-out from messages they need — receipts, password resets, security notices — and once they take it, those stop.

## Adding suppressions

Up to 1000 addresses per call:

```ts TypeScript
const { data } = await aiinbx.suppressions.add({
  addresses: ["grace@example.com", "charles@example.com"],
  key: "product-updates",
  scope: "optional",
  note: "Unsubscribed from the in-app preferences page",
})
```
```python Python
created = client.suppressions.add(
    {
        "addresses": ["grace@example.com"],
        "key": "product-updates",
        "scope": "optional",
        "note": "Unsubscribed from the in-app preferences page",
    }
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/suppressions \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "addresses": ["grace@example.com"],
    "key": "product-updates",
    "scope": "optional",
    "note": "Unsubscribed from the in-app preferences page"
  }'
```
`key` defaults to `*` and `scope` to `all` — so the terse form, `{ "addresses": [...] }`, blocks those addresses everywhere. Adding an address that's already listed updates the existing entry rather than duplicating it, and never narrows a scope that's already wider.

Managing suppressions requires a `full` [scope](../authentication.md#scopes) key.

## What's on an entry

| Prop | Type | Default | Description |
| - | - | - | - |
| `address` | `string` | - | The suppressed address, lowercased. |
| `key` | `string` | - | Which list. `*` is workspace-wide. |
| `space_id` | `string \| null` | - | The space whose sends the entry stops; null stops every space's. |
| `reason` | `"complaint" \| "bounce" \| "unsubscribe" \| "manual"` | - | How the entry got here. |
| `scope` | `"all" \| "optional"` | - | Everything, or optional mail only. |
| `note` | `string \| null` | - | Free text, for entries you added. |
| `blocks` | `number` | - | How many sends this entry has actually stopped. |
| `last_blocked_at` | `string \| null` | - | When it last stopped one. |
| `source_email_id` | `string \| null` | - | The message whose bounce, complaint, or unsubscribe created it. |

`blocks` and `last_blocked_at` are the honest measure of a list. An entry with hundreds of blocks means a code path is repeatedly trying to mail someone who has opted out — worth finding.

## Listing and filtering

```ts TypeScript
for await (const entry of aiinbx.suppressions.list({
  key: "product-updates",
  reason: "unsubscribe",
})) {
  console.log(entry.address, entry.blocks)
}
```
```python Python
for entry in client.suppressions.iter(key="product-updates", reason="unsubscribe"):
    print(entry["address"], entry["blocks"])
```
```bash curl
curl "https://api.aiinbx.com/api/v2/suppressions?key=product-updates&reason=unsubscribe" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Filter by `key`, `reason`, or `address` (a substring search across addresses).

## Removing one

```ts
await aiinbx.suppressions.remove("sup_...")
```
> [!CAUTION]
> Removing a `bounce` or `complaint` entry means mailing an address that already rejected you or reported you as spam. That's how sender reputation gets destroyed. Remove those only when you have specific evidence the address was fixed — a mailbox restored, a typo corrected.

Removing an `unsubscribe` entry because the person asked to be re-subscribed is fine. Removing one because your open rate dropped is not.

## Reacting to the events

The three webhook events that create suppressions are worth handling in your own database too, so your UI reflects reality:

```ts
switch (event.type) {
  case "email.bounced":
    for (const bounce of event.data.bounces ?? []) {
      if (bounce.suppressed) await markUndeliverable(bounce.address)
    }
    break
  case "email.complained":
    await markComplained(event.data.recipients)
    break
  case "email.unsubscribed":
    await recordOptOut(event.data.address, event.data.key, event.data.scope)
    break
}
```
`email.unsubscribed` carries a `source` of `link`, `one_click`, or `reply` — the last meaning the recipient wrote back asking to stop, which was recognized as an opt-out.

## Checking before you send

You don't need to. Filtering happens server-side on every send, and the result comes back in `suppressed`:

```ts
const email = await aiinbx.emails.send(payload)

if (email.suppressed.length) {
  await recordSkipped(email.suppressed)
}
```
If every `to` recipient is suppressed, the send fails with [`recipients_suppressed`](https://docs.aiinbx.com/reference/errors.md#recipients_suppressed) rather than sending an email to nobody.
