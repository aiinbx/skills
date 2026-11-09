# Pacing

Pace outbound mail with sending hours and rate ceilings, see the queue they hold messages in, and inspect, release or override held sends.

From https://docs.aiinbx.com/guides/pacing

Pacing decides _when_ an accepted message actually goes out. Two kinds of rule — sending **hours** and rate **limits** — and everything they hold sits in a queue you can read and release from.

Warming a new domain, keeping a shared mailbox under a provider's ceiling, or simply not sending business mail at 3am on a Sunday: all the same mechanism.

## How a send meets the rules

1. **Match**

    A rule's `match` patterns are tested against the message's sender and
    recipients. A rule with no patterns matches nothing.

2. **Hold or go**

    Every matching rule gets a say. If any of them says no, the message waits.

3. **Report**

    The send response's `pacing` field names the rule that's holding it and
    projects when it will go.

4. **Release**

    The queue drains on its own as windows open and budget frees up — or you
    release messages by hand.

```ts
const email = await aiinbx.emails.send(payload)

if (email.pacing && email.pacing.held_by.kind !== "ready") {
  console.log(
    "held by rule",
    email.pacing.held_by.rule_id,
    "until ~",
    email.pacing.estimated_send_at
  )
}
```
`held_by.kind` is `ready` (going now), `hours` (outside a sending window), or `limit` (a rate ceiling is full). `estimated_send_at` is a projection from the current queue, not a promise.

## Matching

```json
{
  "match": [
    { "field": "from", "pattern": "*@newsletter.example.com" },
    { "field": "to", "pattern": "*@bigcorp.com" }
  ]
}
```
`field` is `from`, `to`, or `either`. Patterns match addresses, and `*` wildcards a segment — so `*@example.com` is a domain and `sales@*` is a local part across domains. A rule needs at least one pattern: an empty `match` array catches nothing. Use `{ "field": "from", "pattern": "*" }` for every send.

## Sending hours

An `hours` rule holds mail outside the windows you declare:

```ts TypeScript
await aiinbx.pacingRules.create({
  name: "Business hours only",
  kind: "hours",
  match: [{ field: "from", pattern: "*@sales.example.com" }],
  schedule: {
    timezone: "Europe/Berlin",
    windows: [{ days: [0, 1, 2, 3, 4], start_minute: 540, end_minute: 1020 }],
  },
})
```
```python Python
client.pacing_rules.create(
    {
        "name": "Business hours only",
        "kind": "hours",
        "match": [{"field": "from", "pattern": "*@sales.example.com"}],
        "schedule": {
            "timezone": "Europe/Berlin",
            "windows": [
                {"days": [0, 1, 2, 3, 4], "start_minute": 540, "end_minute": 1020}
            ],
        },
    }
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/pacing-rules \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Business hours only",
    "kind": "hours",
    "match": [{ "field": "from", "pattern": "*@sales.example.com" }],
    "schedule": {
      "timezone": "Europe/Berlin",
      "windows": [{ "days": [0,1,2,3,4], "start_minute": 540, "end_minute": 1020 }]
    }
  }'
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `timezone` | `string` | - | IANA name, e.g. Europe/Berlin. Windows are evaluated in it, so DST is handled for you. |
| `days` | `number[]` | - | Days the window applies to. 0 is Monday, 6 is Sunday. |
| `start_minute` | `number` | - | Minutes past local midnight. 540 is 09:00. |
| `end_minute` | `number` | - | Minutes past local midnight. 1020 is 17:00; 1440 is end of day. |

Several windows can coexist — weekday mornings plus Saturday afternoon, say. A message is free to go when it falls inside any of them.

### Opening off the dot

Mail held overnight goes out the moment the window opens, so a 09:00 window sends at exactly 09:00:00 every day. Add `jitter` to let each opening and closing drift:

```json
"schedule": {
  "timezone": "Europe/Berlin",
  "windows": [{ "days": [0,1,2,3,4], "start_minute": 540, "end_minute": 1020 }],
  "jitter": {
    "open_early_minutes": 0,
    "open_late_minutes": 20,
    "close_early_minutes": 10,
    "close_late_minutes": 0
  }
}
```
Each day then opens at its own random moment between 09:00 and 09:20 and closes between 16:50 and 17:00. Each value is 0–60. The moment is fixed per rule and day, so `estimated_send_at` and the console preview match what really happens, and two rules with the same hours don't open at the same minute. Leave `jitter` out on create for edges on the dot. On update, leave it out to keep what the rule has, or send all zeros to put the edges back on the dot. Responses always include it, with zeros when nothing drifts.

Drift works on the hours as a whole, not window by window:

- Back-to-back windows (09:00–12:00 and 12:00–17:00) open and close once, at 09:00 and 17:00.
- A gap shorter than the drift can close on some days. With 09:00–12:00 and 13:00–17:00 and 60 minutes either way, the lunch break disappears when 12:00 drifts late and 13:00 drifts early.
- A window shorter than its drift still opens every day, for at least a minute, and never closes later than its latest closing time.

## Rate limits

A `limit` rule caps sends per rolling window:

```ts
await aiinbx.pacingRules.create({
  name: "Warm the new domain",
  kind: "limit",
  match: [{ field: "from", pattern: "*@new.example.com" }],
  limit: { scope: "from_domain", amount: 200, per_seconds: 3600 },
})
```
`amount` per `per_seconds`, counted against a **scope** — which is what the ceiling is per:

| Scope          | The ceiling is per…                                                                  |
| -------------- | ------------------------------------------------------------------------------------ |
| `rule`         | The rule as a whole. One shared budget for everything it matches.                    |
| `from_address` | Each distinct sender address.                                                        |
| `from_domain`  | Each distinct sending domain.                                                        |
| `to_address`   | Each distinct recipient. Good for "never more than one a day to the same person".    |
| `to_domain`    | Each distinct recipient domain. Good for staying under one company's gateway limits. |

```json
{ "limit": { "scope": "to_address", "amount": 1, "per_seconds": 86400 } }
```
The window rolls continuously — it isn't a bucket that resets on the hour.

## Spaces

A rule created with `space_id` reads that [space](spaces.md)'s mail alone. A rule without one — the workspace's — applies to every space, and stacks on top of whatever the spaces set: every matching rule gets a say, and any one of them can hold the message.

That is the shape a platform wants. Your rule caps every customer at once; a customer's own rule, created in their space, can only tighten what they send, never loosen your cap.

```ts
// Yours: 5,000 a day across the whole platform, whichever space sends it.
await aiinbx.pacingRules.create({
  name: "Platform ceiling",
  kind: "limit",
  match: [{ field: "from", pattern: "*" }],
  limit: { scope: "rule", amount: 5000, per_seconds: 86_400 },
})

// Theirs: business hours for one customer's mail only.
await aiinbx.pacingRules.create({
  name: "Acme business hours",
  kind: "hours",
  space_id: "spc_...",
  match: [{ field: "from", pattern: "*" }],
  schedule: {
    timezone: "Europe/Berlin",
    windows: [{ days: [0, 1, 2, 3, 4], start_minute: 540, end_minute: 1020 }],
  },
})
```
`match` still applies inside a space: a space's `*` is "everything in this space", not "everything in the workspace". `pacingRules.list` and `pacing.retrieve` take `space`, and each held message is held by whichever rule — of its space or of the workspace — said no.

## Spread

`spread` is a workspace-wide dial from 0 to 100 that adds randomness to send timing. At `0`, a rate-limited stream goes out on a metronome. Turn it up and the gaps between sends vary instead of being identical.

```ts TypeScript
await aiinbx.pacingRules.updateSpread({ spread: 40 })
```
```python Python
client.pacing_rules.update_spread({"spread": 40})
```
```bash curl
curl -X PATCH https://api.aiinbx.com/api/v2/pacing-rules/spread \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "spread": 40 }'
```
Perfectly regular timing is one of the cheaper signals of automation. A moderate spread costs a little throughput and buys traffic that doesn't look metronomic. It applies to limit rules; hours rules have nothing to jitter.

## The queue

```ts TypeScript
const queue = await aiinbx.pacing.retrieve({ limit: 100 })

console.log(queue.total, "held as of", queue.as_of)

for (const rule of queue.rule_counts) {
  console.log(rule.rule_id, `${rule.held} held of ${rule.matches} matched`)
}

for (const item of queue.data) {
  console.log(item.queued_at, item.from, "→", item.to, item.subject)
}
```
```python Python
queue = client.pacing.retrieve(limit=100)
print(queue["total"], queue["as_of"])
```
```bash curl
curl "https://api.aiinbx.com/api/v2/pacing-queue?limit=100" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `total` | `number` | - | Everything currently held. |
| `sampled` | `number` | - | How many are in `data` — `limit` caps this at 2000, so it can be less than total. |
| `as_of` | `string` | - | When the snapshot was taken. |
| `spread` | `number` | - | The workspace spread in effect. |
| `rules` | `PacingRule[]` | - | The rules the snapshot was evaluated against. |
| `rule_counts` | `object[]` | - | Per rule: how many messages it matched, and how many it is holding. |
| `data` | `object[]` | - | A sample of held messages, oldest first. |

`rule_counts` is the diagnostic worth looking at first: it tells you _which_ rule is doing the holding, so you tune the one that matters instead of guessing.

## Releasing by hand

When something urgent is stuck behind a queue, push it out:

```ts TypeScript
const { data } = await aiinbx.pacing.release({
  email_ids: ["eml_...", "eml_..."],
  count_toward_limits: false,
})
```
```python Python
released = client.pacing.release(
    {"email_ids": ["eml_..."], "count_toward_limits": False}
)
```
```bash curl
curl https://api.aiinbx.com/api/v2/pacing-queue/release \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "email_ids": ["eml_..."], "count_toward_limits": false }'
```
Up to 500 IDs per call; the response lists the ones actually released. `count_toward_limits` defaults to `true` — leave it there unless the release is genuinely exceptional, because `false` means the send doesn't consume budget and the rule under-counts.

Better than releasing repeatedly: set `pacing.skip` on the messages that should never queue in the first place.

```json
{ "pacing": { "skip": true, "count": false } }
```
Password resets, verification codes, and security alerts belong in that category.

## Managing rules

```ts TypeScript
const rules = await aiinbx.pacingRules.list().all()

await aiinbx.pacingRules.update("pace_...", { enabled: false })
await aiinbx.pacingRules.delete("pace_...")
```
```python Python
for rule in client.pacing_rules.iter():
    print(rule["name"], rule["kind"], rule["enabled"])

client.pacing_rules.update("pace_...", {"enabled": False})
```
`enabled: false` parks a rule without losing its configuration — the right move when you're diagnosing which rule is holding mail. Creating and changing rules requires a `full` [scope](../authentication.md#scopes) key; reading the queue does not.

## The provided domain's ceilings

Every workspace's [provided domain](domains.md#your-provided-domain), `<slug>.aiinbx.app`, is for a side project and for the weeks before you bring a domain of your own — so mail sent from it answers to three rules that came with the workspace:

| Rule                                    | Ceiling                                           |
| --------------------------------------- | ------------------------------------------------- |
| Provided domain: 100 a day              | 100 emails per rolling 24 hours, in total         |
| Provided domain: 20 an hour             | 20 emails per rolling hour, in total              |
| Provided domain: 10 a day per recipient | 10 emails per rolling 24 hours to any one address |

They are listed with `builtin: true`, hold mail like any other limit — the send response's `pacing` names them — and cannot be changed, switched off or deleted: the call fails with [`rule_builtin`](https://docs.aiinbx.com/reference/errors.md#rule_builtin). Mail from a domain of your own is not held to any of this.

## Warming a new domain

A staged limit is the usual pattern. Start conservative and raise `amount` as reputation builds:

| Week | `amount` | `per_seconds` |
| ---- | -------- | ------------- |
| 1    | 50       | 86400         |
| 2    | 200      | 86400         |
| 3    | 1000     | 86400         |
| 4    | 5000     | 86400         |

```ts
await aiinbx.pacingRules.update(warmupRuleId, {
  limit: { scope: "from_domain", amount: 200, per_seconds: 86_400 },
})
```
Watch `email.bounced` and `email.complained` between steps. Rising complaints mean hold, not advance.
