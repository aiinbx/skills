# Authentication

Authenticate with Bearer API keys: the three scopes and what each one reaches, how to keep keys server-side, and how to rotate and delete them.

From https://docs.aiinbx.com/authentication

Every request from your code carries an API key as a bearer token:

```bash
curl https://api.aiinbx.com/api/v2/threads \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Both SDKs read `AI_INBX_API_KEY` from the environment, so the common case needs no argument:

```ts TypeScript
import AIInbx from "aiinbx"

const aiinbx = new AIInbx() // or new AIInbx({ apiKey: "..." })
```
```python Python
from aiinbx import AIInbx

with AIInbx() as client:  # or AIInbx(api_key="...")
    ...
```
A key belongs to one workspace, reaches every [space](guides/spaces.md) in it, and can never cross into another workspace. Keys are for your servers; a customer who needs the API goes through your product, never through AI Inbx directly.

## Signed-in clients

The [CLI](https://docs.aiinbx.com/cli.md) and the [MCP servers](https://docs.aiinbx.com/mcp.md) sign in with your account instead of a key. On the consent page you pick the workspaces and access, or, for an [inbox](https://docs.aiinbx.com/guides/agent-inbox.md), the addresses it gets. They send the access token they get as the same bearer token. A token that reaches several workspaces names the one a request acts in with `AIInbx-Workspace: <workspace id>`; `GET /workspaces` lists them, and a request without it gets `400` [`workspace_required`](https://docs.aiinbx.com/reference/errors.md#workspace_required).

## Base URL

```
https://api.aiinbx.com/api/v2
```
Both [SDKs](https://docs.aiinbx.com/sdks.md) default to it. Override with `baseURL` (TypeScript) or `base_url` (Python) when you're pointing at a local instance.

> [!WARNING]
> **Missing or invalid keys**
>
> A request with no `Authorization` header, a malformed one, or a deleted key gets `401` with the code [`unauthorized`](https://docs.aiinbx.com/reference/errors.md#unauthorized) and a `WWW-Authenticate: Bearer` header. A valid key that lacks the scope for the operation gets `403` [`forbidden`](https://docs.aiinbx.com/reference/errors.md#forbidden) — a different failure, and one retrying won't fix.

## Scopes

A key is created with one of three scopes. `full` satisfies every requirement; the other two are deliberate narrowings for keys that live somewhere you'd rather not put a `full` key.

| Prop | Type | Default | Description |
| - | - | - | - |
| `full` | `scope` | - | Everything. Required for anything that changes configuration — keys, domains, mailboxes, OAuth apps, webhook endpoints, suppressions, and pacing rules. |
| `sending?` | `scope` | - | Reads, plus sending: send an email, reply or forward a thread, edit and send drafts, reschedule, and cancel. Cannot change configuration. |
| `read?` | `scope` | - | Reads only: list and retrieve emails, threads, domains, mailboxes, deliveries, suppressions, and pacing state. |

Concretely:

| Operation                                                                                                 | `read` | `sending` | `full` |
| --------------------------------------------------------------------------------------------------------- | :----: | :-------: | :----: |
| List and retrieve emails, threads, attachments                                                            |   ✓    |     ✓     |   ✓    |
| Read domains, mailboxes, webhook endpoints and deliveries, suppressions, pacing                           |   ✓    |     ✓     |   ✓    |
| [Send an email](https://docs.aiinbx.com/guides/sending.md), [reply on a thread](guides/threads.md#replying)                           |        |     ✓     |   ✓    |
| [Reschedule](https://docs.aiinbx.com/guides/scheduling.md#rescheduling) or [cancel](https://docs.aiinbx.com/guides/scheduling.md#canceling) a scheduled email |        |     ✓     |   ✓    |
| [Edit, send or discard a draft](https://docs.aiinbx.com/guides/drafts.md#editing-and-sending-from-code)                             |        |     ✓     |   ✓    |
| Create, update, delete domains, mailboxes, OAuth apps, webhook endpoints                                  |        |           |   ✓    |
| Manage [suppressions](guides/suppressions.md) and [pacing rules](guides/pacing.md)                            |        |           |   ✓    |
| Create and delete API keys                                                                                |        |           |   ✓    |

API keys, members, invitations and workspace settings are admin-only on top of the scope. A key acts with its creator's current role: once they are no longer an admin, or have left the workspace, those calls fail with 403 `forbidden`.

> [!TIP]
> A worker that only sends should hold a `sending` key, and an analytics job that only reads should hold a `read` one. Both are ordinary keys — the narrowing is enforced server-side, so a leaked `read` key cannot send.

## Managing keys

Keys are created in the [console](https://aiinbx.com/app) or through the API with a `full` key made by a current admin. The plaintext key is returned **once**, on the create call:

```ts TypeScript
const created = await aiinbx.apiKeys.create({
  name: "background worker",
  scope: "sending",
})

console.log(created.key) // the only time you see it
```
```python Python
created = client.api_keys.create({"name": "background worker", "scope": "sending"})

print(created["key"])  # the only time you see it
```
```bash curl
curl https://api.aiinbx.com/api/v2/api-keys \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "background worker", "scope": "sending" }'
```
Afterwards a key lists as metadata only — its `prefix`, `scope`, `created_at`, and `last_used_at`. `last_used_at` is the field to check before deleting a key you're no longer sure about.

### Rotating

Rotate a key by creating a replacement, deploying it, and deleting the old key after the cutover.

1. **Create the replacement**

    Same scope, a name that says what it replaces.

2. **Deploy it**

    Roll it out everywhere the old key was configured.

3. **Watch last_used_at**

    List keys until the old one stops advancing.

4. **Delete the old key**

    `apiKeys.delete(id)` — takes effect immediately.

Deletion is permanent. The key disappears from the list and authenticates nothing.

## Request IDs

Every response carries `X-Request-ID`, and every error body repeats it as `request_id`. If you send your own `X-Request-ID` — up to 128 characters of `A-Za-z0-9._:-` — it's echoed back instead of a generated one, which is what you want when you already have a trace ID to correlate against. See [Conventions](https://docs.aiinbx.com/reference/conventions.md#request-ids).
