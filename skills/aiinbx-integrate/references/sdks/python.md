# Python

Install, configure, and use the aiinbx package — sync and async clients, pagination, raw responses, typed errors, and webhook verification.

From https://docs.aiinbx.com/sdks/python

The official Python client. Sync and async on [httpx](https://www.python-httpx.org), fully typed, Python 3.10+.

```bash
pip install aiinbx
```
## Getting started

```python
from aiinbx import AIInbx

with AIInbx() as client:  # reads AI_INBX_API_KEY
    email = client.emails.send(
        {
            "from_": {"name": "Ada", "address": "ada@example.com"},
            "to": ["grace@example.com"],
            "subject": "Hello",
            "text": "Sent with AI Inbx.",
        },
        idempotency_key="welcome-grace-v1",
    )

print(email["id"], email["thread_id"])
```
> [!NOTE]
> **`from_`, with an underscore**
>
> `from` is a reserved word in Python, so the sender field is `from_`. Every other field keeps its wire name — `thread_id`, `scheduled_at`, `suppression_key`.

Use the client as a context manager so its connection pool is closed. A long-lived client constructed once at startup is fine too — just close it on shutdown.

## Async

```python
from aiinbx import AsyncAIInbx

async with AsyncAIInbx() as client:
    page = await client.threads.list(limit=20)

    async for message in client.threads.iter_messages("thr_123"):
        print(message["snippet"])
```
`AsyncAIInbx` mirrors `AIInbx` method for method. Everything below applies to both, awaiting where appropriate.

## Configuration

```python
import os

client = AIInbx(
    api_key=os.environ["AI_INBX_API_KEY"],
    timeout=60.0,
    max_retries=2,
    # base_url="http://localhost:3000/api/v2",
)
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `api_key?` | `str \| None` | - | Defaults to AI_INBX_API_KEY. A missing key raises ValueError at construction. |
| `base_url?` | `str` | `"https://api.aiinbx.com/api/v2"` | Point at a local or self-hosted API. |
| `timeout?` | `float \| httpx.Timeout` | `60.0` | Seconds, or an httpx.Timeout for per-phase control. |
| `max_retries?` | `int` | `2` | 0–5. Retries: a 429 always; network errors, 408 and 5xx when a retry can't act twice (a GET, PUT or DELETE, or a keyed send, reply or forward). |
| `http_client?` | `httpx.Client \| httpx.AsyncClient \| None` | - | Bring your own — for proxies, custom transports, or a shared pool. |
| `default_headers?` | `Mapping[str, str] \| None` | - | Headers added to every request. |

## Resources

Each API resource is an attribute of the client — `client.emails`, `client.webhook_endpoints`, `client.pacing_rules` — and every endpoint page in the [API reference](https://docs.aiinbx.com/api.md) shows its Python call.

A method takes path IDs first, then the request body as one dict, then query filters and `idempotency_key` as keyword arguments. Bodies are `TypedDict`s from `aiinbx.models`, named after the API's schemas — `SendEmailRequest`, `ReplyThreadRequest`, `CreateDomainRequest`:

```python
domain = client.domains.create({"name": "mail.example.com", "region": "eu-central-1"})

page = client.suppressions.list(key="product-updates", reason="unsubscribe")
```
### Replying

Recipients, subject, and reply headers are inferred from the thread — pass only the content:

```python
reply = client.threads.reply(
    "thr_123",
    {"text": "Sounds good — see you Thursday."},
    idempotency_key="reply-thr-123-v1",
)
```
## Pagination

`list` returns one page; `iter` follows cursors lazily, keeping your filters:

```python
page = client.emails.list(limit=100, status="scheduled")
print(page["data"], page["next_cursor"])

for email in client.emails.iter(direction="inbound"):
    print(email["subject"])
```
On `AsyncAIInbx`, `iter` is an async iterator:

```python
async for email in client.emails.iter(direction="inbound"):
    print(email["subject"])
```
## Raw responses and request IDs

```python
response = client.with_raw_response.emails.retrieve("eml_123")

print(response.status_code, response.request_id)
email = response.json()

# The most recently completed response is also on the client.
print(client.last_request_id)
```
## Errors

Non-2xx responses raise typed subclasses of `APIStatusError`, each carrying `status_code`, the problem `code`, `request_id`, validation `issues`, response `headers`, and the decoded `body`:

```python
from aiinbx import APIStatusError, NotFoundError, RateLimitError

try:
    client.domains.retrieve("dom_missing")
except NotFoundError:
    domain = None
except RateLimitError as error:
    schedule_retry(error.headers.get("retry-after"))
except APIStatusError as error:
    print(error.status_code, error.code, error.request_id)
    print(error.issues)
```
| Exception                  | Raised for                       |
| -------------------------- | -------------------------------- |
| `BadRequestError`          | 400                              |
| `AuthenticationError`      | 401                              |
| `PermissionDeniedError`    | 403                              |
| `NotFoundError`            | 404                              |
| `ConflictError`            | 409                              |
| `UnprocessableEntityError` | 422                              |
| `RateLimitError`           | 429                              |
| `InternalServerError`      | 5xx                              |
| `APIConnectionError`       | The request never got a response |
| `APITimeoutError`          | The request timed out            |

All derive from `AIInbxError`, so one `except AIInbxError` catches everything the client raises. Codes and what to do about each are in [Errors](../reference/errors.md).

## Webhooks

Pass the **unmodified** request body. The verifier signs `<unix_timestamp>.<payload>` with HMAC-SHA256 and rejects payloads older than five minutes by default:

```python
event = client.webhooks.verify(
    request_body,
    request.headers["AIInbx-Signature"],
    webhook_secret,
)

if event["type"] == "email.bounced":
    for recipient in event["data"]["recipients"]:
        print(recipient)
```
`event` is a discriminated `WebhookEvent` union — a type checker narrows `data` from `type`.

It raises `WebhookVerificationError` on a bad signature, a malformed header, or a timestamp outside the window. Widen the window with `tolerance=` (seconds; `0` accepts only the same second) only as a diagnostic.

A module-level `verify_webhook` is exported too, for handlers that don't have a client to hand:

```python
from aiinbx import verify_webhook

event = verify_webhook(body, signature, secret)
```
### FastAPI

```python
from fastapi import FastAPI, Request, Response
from aiinbx import AIInbx, WebhookVerificationError

app = FastAPI()
client = AIInbx()

@app.post("/webhooks/aiinbx")
async def webhook(request: Request):
    body = await request.body()  # raw bytes, before any parsing

    try:
        event = client.webhooks.verify(
            body,
            request.headers["aiinbx-signature"],
            WEBHOOK_SECRET,
        )
    except WebhookVerificationError:
        return Response(status_code=400)

    if event["type"] == "email.received" and event["data"]["category"] == "human":
        await queue_reply(event["data"]["thread_id"])

    return Response(status_code=204)
```
Take `Request` rather than a Pydantic model — a declared body model parses the request before your handler runs, and the signature covers the exact bytes. More in [Verifying](../webhooks/verifying.md#getting-the-raw-body).

## Typing

The package ships `py.typed`, so mypy and Pyright see everything. Resources return `TypedDict`s that match the API's JSON, and models are importable:

```python
from aiinbx.models import ReplyThreadRequest, WebhookEvent

def summarize(event: WebhookEvent) -> str:
    if event["type"] == "email.received":
        return event["data"]["snippet"]  # narrowed
    return event["type"]
```
