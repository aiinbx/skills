# Errors

Every AI Inbx error code in the problem+json body, what each one means, how to recover from it, and which failures are worth retrying.

From https://docs.aiinbx.com/reference/errors

Every failure answers with `application/problem+json` ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)):

```json
{
  "type": "https://docs.aiinbx.com/problems/invalid_request",
  "title": "Unprocessable Content",
  "status": 422,
  "detail": "Request body is invalid",
  "code": "invalid_request",
  "request_id": "8f2c4a1b-...",
  "issues": [
    { "path": "to.0", "message": "Invalid email address" },
    {
      "path": "subject",
      "message": "String must contain at most 998 character(s)"
    }
  ]
}
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `code` | `string` | - | The stable, machine-readable identifier. Branch on this, never on `detail`. |
| `status` | `number` | - | The HTTP status, repeated in the body. |
| `title` | `string` | - | The standard reason phrase for the status. |
| `detail` | `string` | - | A human-readable explanation. Wording may change — don't parse it. |
| `request_id` | `string` | - | Also on the X-Request-ID header. Quote it when reporting a problem. |
| `type` | `string` | - | A URI for the code — it resolves to this page. |
| `issues?` | `object[]` | - | Field-level validation failures: a `path` and a `message` each. |

## Handling them

```ts TypeScript
import { APIError, NotFoundError, RateLimitError } from "aiinbx"

try {
  await aiinbx.emails.send(payload)
} catch (error) {
  if (error instanceof NotFoundError) return null
  if (error instanceof RateLimitError) return scheduleRetry()

  if (error instanceof APIError) {
    console.error(error.status, error.code, error.requestId)
    if (error.code === "invalid_request") {
      console.error(error.issues)
    }
  }

  throw error
}
```
```python Python
from aiinbx import APIStatusError

try:
    client.emails.send(payload)
except APIStatusError as error:
    print(error.status_code, error.request_id)
    print(error.body["code"], error.body.get("issues"))
```
## Retry or not

| Status | Meaning                             | Retry?                   |
| ------ | ----------------------------------- | ------------------------ |
| `400`  | Malformed request                   | No — fix the request     |
| `401`  | Missing or invalid key              | No                       |
| `402`  | The plan doesn't cover it           | No — read `code` first   |
| `403`  | Key lacks the scope                 | No                       |
| `404`  | No such resource                    | No                       |
| `409`  | Conflicts with current state        | No — read `code` first   |
| `413`  | Payload too large                   | No                       |
| `415`  | Body isn't JSON                     | No                       |
| `422`  | Failed validation                   | No — read `issues`       |
| `429`  | Budget spent or provider throttling | Yes, after `Retry-After` |
| `500`  | Something broke on our side         | Yes, with backoff        |
| `502`  | Upstream mail provider failed       | Yes, with backoff        |
| `503`  | Temporarily unavailable             | Yes, with backoff        |

Both SDKs already retry `429` twice, and network failures, `408` and `5xx` twice when the retry cannot act twice — a `GET`, `PUT` or `DELETE`, or a send, reply or forward with an idempotency key — with bounded exponential backoff, honoring `Retry-After`. They never retry a `409` or other `4xx`. Tune with `maxRetries` / `max_retries`.

> [!WARNING]
> When you retry a send yourself, reuse the same [`Idempotency-Key`](conventions.md#idempotency). Without one, a retry after a timeout can send the message a second time — the request may well have succeeded before the connection dropped.

## Codes

Grouped by what actually went wrong.

### Authentication and authorization

#### unauthorized

`401` — no `Authorization` header, a malformed one, or a deleted key. The response carries `WWW-Authenticate: Bearer`. Check that the key is being read from the environment and hasn't been deleted. See [Authentication](https://docs.aiinbx.com/authentication.md).

#### forbidden

`403` — the key is valid but its [scope](https://docs.aiinbx.com/authentication.md#scopes) doesn't cover this operation, or the operation is admin-only and the key's creator is no longer an admin of the workspace. A `read` key can't send; a `sending` key can't change configuration. Retrying won't help; use a key with the right scope, made by a current admin.

#### workspace_required

`400` — an access token from a [CLI](https://docs.aiinbx.com/cli.md) or [MCP](https://docs.aiinbx.com/mcp.md) sign-in reaches several workspaces and the request named none. Send `AIInbx-Workspace` with an `id` from `GET /workspaces`. API keys belong to one workspace and never see this.

#### shared_apps_disabled

`403` — `mailboxes.connect` was called without an `app_id`. Connecting without one isn't available: register your own [OAuth app](../guides/mailboxes.md#your-own-oauth-app) and pass its ID.

### Plan and allowances

#### plan_required

`402` — the workspace has no plan, and a workspace without one can't send or create anything: domains, mailboxes, pacing rules, API keys, spaces, webhook endpoints, OAuth apps. Receiving is not affected. Choose one in the console's billing settings. Retrying won't help until then.

#### allowance_reached

`402` — the plan's allowance for this is used up; `detail` says which. Emails are counted per month: upgrade, turn on extra usage in billing settings, or wait for the month to turn. Custom domains, mailboxes and active custom pacing rules are counted as held: delete a domain, disconnect a mailbox or disable a rule to free one, or upgrade. Subdomains under a wildcard, the [provided domain](../guides/domains.md#your-provided-domain) and its [ceilings](../guides/pacing.md#the-provided-domains-ceilings) are never counted, and reconnecting a mailbox that is `active` or `needs_reauth` takes nothing new.

### Request format

#### unsupported_media_type

`415` — an operation that accepts a JSON body was called without `Content-Type: application/json`. Bodyless operations do not require this header.

#### invalid_json

`400` — the `Content-Type` said JSON but the body didn't parse. Usually truncation or a stray trailing comma.

#### invalid_request

`422` — the body parsed but failed validation. `issues` names each offending field with a `path` and a `message`. Also `400` from a few operations that validate outside the body schema.

#### invalid_query

`400` — a query parameter was rejected: an out-of-range `limit`, an unknown `status`, a malformed address filter. `issues` names the parameter.

#### invalid_cursor

`400` — the `cursor` isn't a cursor this endpoint issued. Cursors aren't portable between endpoints; use the `next_cursor` from the same list. See [Pagination](conventions.md#pagination).

#### invalid_thread_id

`400` — the `thread_id` isn't a well-formed thread ID (`thr_` plus 32 hex characters).

#### invalid_idempotency_key

`400` — the `Idempotency-Key` header is empty or longer than 255 characters.

#### method_not_allowed

`405` — the path exists but not for this HTTP method.

#### no_body

`400` — an edit with [`PATCH /emails/{id}`](https://docs.aiinbx.com/api/endpoints/emails/update-email.md) would leave the email with neither `html` nor `text`. A send without either is [`invalid_request`](#invalid_request).

### Content

#### no_recipients

`400` — a reply has nobody to go to. Either the email it answers went only to you, or its Reply-To names only your own addresses (a receipt asking answers to go to your support desk, say), and AI Inbx does not guess someone else in its place. Name `to`. A send with an empty `to` is [`invalid_request`](#invalid_request).

#### recipients_suppressed

`400` — every `to` address is [suppressed](../guides/suppressions.md), so there is nobody left to send to. On a partial send the response's `suppressed` array tells the same story less severely. An invitation to a suppressed address fails the same way, and nothing is created. Retrying will not help.

#### sender_unknown

`400` — the `from` address is on no [domain](../guides/domains.md) and no connected [mailbox](../guides/mailboxes.md) of the workspace. Add its domain, or connect the mailbox and check it's `active`. A domain the workspace has but hasn't verified gets [`domain_unverified`](#domain_unverified) instead. It also comes when the address now belongs to another space than the one the send is held to — an agent inbox, or mail scheduled before its domain moved.

#### tracking_unavailable

`400` — the send asked for `tracking.opens` or `tracking.clicks` from a connected [mailbox](../guides/mailboxes.md). Only mail sent from a [domain](../guides/domains.md#tracking-settings) is tracked; drop `tracking`, or set both to `false`.

#### too_many_recipients

`422` — a reply took its `to` or `cc` from the thread and found more than 100 addresses for one of them. Name the recipients yourself. More than 100 addresses in any recipient field you send is [`invalid_request`](#invalid_request).

#### unsubscribe_single_recipient

`400` — a send marked [`unsubscribe: true`](../guides/suppressions.md#marking-a-send-as-optional) has more than one recipient across `to`, `cc` and `bcc`. The unsubscribe link is personal; send one message per recipient.

#### attachments_too_large

`413` — the attachments given in one request decode to more than 3 MB in total, or, on [`PATCH /emails/{id}`](https://docs.aiinbx.com/api/endpoints/emails/update-email.md), the files the email would carry come to more than 7 MB together. Large files belong behind a link. (More than 20 attachments is [`invalid_request`](#invalid_request).)

### State conflicts

#### not_found

`404` — no such resource in this workspace. Also what you get for an ID of the wrong type, since IDs are type-tagged, and for a `space_id` or `space` that names no space of the workspace.

#### thread_not_found

`404` — the thread doesn't exist or isn't yours. Check the `thread_id` came from this workspace.

#### thread_empty

`409` — a forward was asked of a thread with nothing sent or received on it yet — only scheduled or canceled messages. There is nothing to forward until a message has left or arrived.

#### not_prepared

`404` — the attachment has no prepared text. Either its type is unsupported, extraction failed, or preparation hasn't finished. The original is still downloadable — see [Attachments](https://docs.aiinbx.com/guides/attachments.md#prepared-text).

#### not_scheduled

`409` — cancel or `PATCH /emails/{id}` was called on a message that is neither `scheduled` nor a `draft` any more. It has already begun sending, or was canceled or discarded. See [Scheduling](../guides/scheduling.md) and [Drafts](../guides/drafts.md).

#### draft_changed

`409` — the draft or scheduled email changed while this edit or send was being written, by another request. Read it again, then retry.

#### not_draft

`409` — `POST /emails/{id}/send` was called on a message that isn't a draft — it was sent already, or discarded. See [Drafts](../guides/drafts.md).

#### attachment_not_found

`422` — an edit kept an attachment, by `{ id }`, that isn't one of this email's. Keep only ids from the email's own `attachments`.

#### unsubscribe_recipient_fixed

`409` — an edit changed the recipients of an unsubscribable email. Its unsubscribe link was made for the one recipient it was written to; send anyone else a new email.

#### idempotency_key_reused

`409` — this `Idempotency-Key` was used with a different body. Either the key is being reused across genuinely different sends, or the body isn't as deterministic as you think (a timestamp in the subject will do it). See [Idempotency](conventions.md#idempotency).

#### domain_unverified

`409` — the `from` address is on a domain of the workspace whose DKIM identity isn't verified, or has lost verification. The message names the domain. Run `domains.verify`, then `domains.diagnostics` to see which records are missing and what else in the zone is in the way. A scheduled or queued email whose domain loses verification before it leaves fails, with the same message on its timeline. See [Domains](../guides/domains.md).

#### domain_taken

`409` — the space already has this domain (or the workspace, for a domain without a `space_id`). `saas.com` and `*.saas.com` are one identity, so either one takes the other. Another space or workspace having the name is not an error: whoever's DKIM record DNS carries sends as it — see [Moving a domain](../guides/domains.md#moving-a-domain-between-spaces-and-workspaces).

#### reserved_subdomain

`409` — the name is `bounces.` under a wildcard, which is where that wildcard's return path lives. Pick another label.

#### reserved_domain

`409` — the name is under `aiinbx.app`. Every workspace is given one name there, `<slug>.aiinbx.app`, and nobody adds another — see [Your provided domain](../guides/domains.md#your-provided-domain).

#### domain_platform

`409` — the domain is the one the platform sends its own mail from, and cannot be deleted. Only ever seen by the platform's own workspace.

#### domain_provided

`409` — the domain is the one the workspace was given. It came with the workspace and cannot be deleted; it goes when the workspace does.

#### rule_builtin

`409` — the pacing rule came with the workspace. The [provided domain's ceilings](../guides/pacing.md#the-provided-domains-ceilings) cannot be changed, switched off or deleted.

#### slug_taken

`409` — the `slug` is claimed. OAuth app slugs are globally unique because they form the hosted connect URL; workspace slugs because they form the console URL. Pick another one.

#### external_id_taken

`409` — another space in the workspace already has that `external_id`. One space per customer is the point of the field; find the existing one with `GET /spaces?external_id=`. See [Spaces](../guides/spaces.md#finding-a-space-by-your-id).

#### mailbox_inactive

`409` — the mailbox isn't `active`. Check its `state`: `needs_reauth` means the customer has to reconnect, `disconnected` means it's gone. See [Mailboxes](../guides/mailboxes.md#mailbox-state).

#### mailbox_busy

`409` — another request is changing the mailbox right now, such as a disconnect or a reconnect. Nothing changed; try again once it has finished.

#### mailbox_space_conflict

`409` — the mailbox can't be connected in this space. Either it is already connected in another space (a mailbox's space is fixed when it is first connected, so reconnect it through a URL minted for that space), or its address is on a domain verified in another space, which decides the address's space — connect it there. See [One address, one space](../guides/spaces.md#one-address-one-space).

#### invalid_domain

`400` — the domain name isn't a valid registrable domain.

#### invalid_webhook_url

`400` — a webhook endpoint `url` isn't one events can be sent to: not an absolute URL, not `https`, carrying credentials, on a port other than 443, or resolving to anything but public addresses (`localhost`, loopback, private and link-local ranges are refused). `detail` says which.

#### unknown_region

`400` — `region` isn't one of `eu-central-1` or `us-east-1`.

#### region_fixed

`409` — the domain is verified, or is a subdomain, and its region no longer moves. A verified domain changes region by being deleted and added again; a subdomain is handled wherever its wildcard is. See [Regions](../guides/domains.md#regions).

### OAuth apps

#### invalid_tenant

`400` — `tenant` isn't a Microsoft tenant ID or domain.

#### invalid_topic

`400` — `pubsub_topic` isn't of the form `projects/<project>/topics/<name>`.

#### invalid_slug

`400` — an OAuth app `slug` holds something other than letters, digits and dashes, or starts or ends with a dash.

#### invalid_accent

`400` — `accent` isn't a hex colour, like `#3d84ea`, or an `oklch()` one.

#### invalid_logo

`400` — `logo_url` isn't an `https` link.

### OAuth flow

#### invalid_state

`400` — the OAuth callback's state parameter didn't validate. Usually a link opened out of context; start the [connect flow](../guides/mailboxes.md#connecting-a-mailbox) again.

#### expired_state

`400` — the connection URL expired, was already used, or was never one of ours. They're short-lived and single-use: good for ten minutes, spent when the person approves — redirect the customer, don't email them the link. Answered as JSON on our callback page rather than on `return_to`, which went with the state.

#### invalid_return_url

`400` — a `return_urls` entry on an OAuth app isn't one a connect link could return to: not an absolute URL, not `https` (plain `http` is allowed on `localhost`), carrying credentials, or carrying a `#fragment`. At most 20 per app.

#### return_url_not_registered

`400` — a [connect link](../guides/mailboxes.md#connect-link) was opened with a `return_to` the app hasn't registered. Add it to the app's `return_urls` — matched on origin and path, so one entry covers every query string. The server-side `mailboxes.connect` call isn't held to the list, since it's authenticated.

#### connect_unavailable

`503` — the connect flow is briefly unavailable, so `mailboxes.connect`, a connect link or the callback can't go on. Nothing was spent; retry with backoff, or send the person through the flow again.

### Workspace and members

#### rename_cooldown

`409` — the workspace was renamed in the last 30 days. `detail` says when it can be renamed again.

#### owner_protected

`409` — the member is the workspace's owner, who can't be removed or given another role. The owner has to transfer the workspace first.

#### already_member

`409` — the invited address already belongs to a member of the workspace.

#### invitation_limit

`409` — the workspace already has 100 invitations waiting. Cancel one first.

#### invitation_not_sent

`429` — the invitation was saved but its email was held back: the address, the inviter or the platform has sent as many invitations as it may for now. `detail` says when to try again, which is in an hour or tomorrow, so the response carries no `Retry-After`. The invitation is listed, to be sent again or canceled.

### Upstream and server

#### too_many_requests

`429` — the workspace's [request budget](conventions.md#request-budgets) is spent, too much of it is in flight, or the platform is at capacity; `detail` says which. The response carries `Retry-After` in seconds and the `RateLimit-*` headers. Wait that long and retry — both SDKs do so automatically. Sustained `429`s mean the integration is faster than the budget: pace it from `RateLimit-Remaining`, or ask for a larger budget.

#### rate_limited

`429` — an upstream mail provider is throttling. Not the workspace's budget: that is [`too_many_requests`](#too_many_requests). The response carries `Retry-After` in seconds. Wait that long and retry — both SDKs do so automatically.

#### send_failed

`502` — the provider rejected the message at hand-off. `detail` carries what it said. Retryable, but if it repeats for the same message the content or sender is the problem.

#### provider_error

`502` — a mail provider (Gmail, Microsoft Graph, the sending backend) failed. Retry with backoff.

#### verification_unavailable

`503` — the mail provider or DNS didn't answer a domain check in time. Nothing changed; retry in a moment.

#### live_unavailable

`503` — a list asked for as `text/event-stream` can't be followed right now. Retry with backoff.

#### internal_error

`500` — something broke on our side. The `request_id` is already logged; quote it if it persists. Retry with backoff.

## Debugging

1. **Read `code`, not `detail`**

    `code` is stable. `detail` is prose and may be reworded.

2. **For 422, read `issues`**

    Each entry has a `path` into your request body and the reason it failed.

3. **Keep `request_id`**

    Log it beside your own trace ID — or send your own via
    [`X-Request-ID`](conventions.md#request-ids) so they're the same
    value.

4. **Check the scope for a 403**

    A `403` after a working `200` on a read endpoint is almost always a key
    scoped too narrowly.
