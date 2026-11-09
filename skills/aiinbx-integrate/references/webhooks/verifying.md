# Verifying

Verify AI Inbx webhook signatures: the HMAC-SHA256 scheme, verifying in the TypeScript and Python SDKs, and checking it by hand in any language.

From https://docs.aiinbx.com/webhooks/verifying

Your webhook URL is a public endpoint. Anyone who learns it can post to it. The signature is what separates an event AI Inbx sent from one somebody made up — verify it on every request, before you parse the body.

## The scheme

Each delivery carries:

```
AIInbx-Signature: t=1756721464,v1=5f2b8c...
```
`t` is the Unix timestamp of the signature. `v1` is `HMAC-SHA256(secret, "<t>.<raw body>")`, hex-encoded.

Two properties matter:

- **The signed string includes the timestamp**, so an old capture can't be replayed indefinitely. Anything outside a **five-minute** window is rejected by default.
- **It's computed over the raw bytes**, so re-serializing the JSON before verifying will produce a different string and fail. Key order, whitespace, and Unicode escaping all change the bytes.

### The secret

The secret is the `whsec_…` value in the response that created the endpoint. It is shown that once: reading the endpoint later does not return it, so store it then. One that was lost, or has leaked, is replaced with [`POST /webhook-endpoints/{id}/rotate-secret`](../webhooks.md#rotating-the-secret), which returns a new secret — also once — and keeps the old one verifying for `grace_seconds`, a day unless you say otherwise.

On your own machine, [`aiinbx listen`](https://docs.aiinbx.com/cli.md#listen-webhooks-on-localhost) forwards events to `localhost` signed with a secret of its own. `aiinbx listen --print-secret` prints it, and it stays the same from run to run, so the handler below verifies local deliveries unchanged.

During that grace period a delivery is signed with both secrets, and the header carries a `v1` for each:

```
AIInbx-Signature: t=1756721464,v1=<new secret's>,v1=<old secret's>
```
A request is genuine when **any** `v1` matches the one you compute, which is what lets you deploy the new secret at your own pace. The SDK helpers do this; a hand-rolled verifier has to as well, or it will reject every delivery until the grace period ends.

> [!CAUTION]
> **Verify before you parse**
>
> Reach for the raw body first. A framework that has already parsed JSON into an object has thrown away the exact bytes the signature covers — see [Getting the raw body](#getting-the-raw-body).

## TypeScript

`verifyWebhookRequest` takes the `Request` and does everything — raw body, header, timing-safe comparison, replay window:

```ts app/webhooks/aiinbx/route.ts
import { verifyWebhookRequest } from "aiinbx/webhooks"

export async function POST(request: Request) {
  let event

  try {
    event = await verifyWebhookRequest(
      request,
      process.env.AI_INBX_WEBHOOK_SECRET!
    )
  } catch {
    return new Response("invalid signature", { status: 400 })
  }

  // `type` is a discriminant — TypeScript narrows `data` to this event's shape.
  if (event.type === "email.received") {
    console.log(event.data.thread_id, event.data.category)
  }

  return new Response(null, { status: 204 })
}
```
When your framework hands you the body and header separately, use `verifyWebhook`:

```ts
import { verifyWebhook } from "aiinbx/webhooks"

const event = await verifyWebhook(
  rawBody, // string or Uint8Array — exactly as received
  request.headers["aiinbx-signature"],
  process.env.AI_INBX_WEBHOOK_SECRET!
)
```
Both throw `WebhookSignatureError` on a bad signature, a malformed header, or a timestamp outside the window. Both work anywhere Web Crypto does — Node 20+, Bun, Deno, Cloudflare Workers, Vercel Edge.

## Python

```python
from aiinbx import AIInbx

client = AIInbx()

event = client.webhooks.verify(
    request_body,                          # bytes or str, unmodified
    request.headers["AIInbx-Signature"],
    webhook_secret,
)

if event["type"] == "email.bounced":
    for recipient in event["data"]["recipients"]:
        print(recipient)
```
`event` is a discriminated `WebhookEvent` union, so a type checker narrows `data` from `type`. It raises `WebhookVerificationError` on a bad signature, a malformed header, or a timestamp outside the window.

## Any language

Six steps, no library required:

1. **Read the raw body**

    As bytes or a string. Do not parse it yet.

2. **Parse the header**

    Split `AIInbx-Signature` on `,`, then each part on the first `=`, giving `t`
    and one `v1` — two while a [rotated secret](#the-secret) is in its grace
    period.

3. **Check the timestamp**

    Reject if `|now − t|` exceeds 300 seconds.

4. **Build the signed string**

    `"<t>.<raw body>"` — a literal dot between them.

5. **Compute the HMAC**

    HMAC-SHA256 with your endpoint secret, hex-encoded.

6. **Compare in constant time**

    Use a timing-safe comparison, never `==`. Accept when any `v1` matches.

```go
func verify(body []byte, header, secret string) (bool, error) {
    var timestamp string
    var provided []string
    for _, part := range strings.Split(header, ",") {
        key, value, _ := strings.Cut(strings.TrimSpace(part), "=")
        switch key {
        case "t":
            timestamp = value
        case "v1":
            provided = append(provided, value)
        }
    }

    seconds, err := strconv.ParseInt(timestamp, 10, 64)
    if err != nil {
        return false, err
    }
    if math.Abs(float64(time.Now().Unix()-seconds)) > 300 {
        return false, errors.New("outside the replay window")
    }

    mac := hmac.New(sha256.New, []byte(secret))
    fmt.Fprintf(mac, "%s.%s", timestamp, body)
    expected := hex.EncodeToString(mac.Sum(nil))

    for _, signature := range provided {
        if hmac.Equal([]byte(expected), []byte(signature)) {
            return true, nil
        }
    }
    return false, nil
}
```
## Getting the raw body

The usual failure is a framework parsing JSON before you see it. Per framework:

**Next.js (App Router)**

`Request` gives you the raw body directly:

```ts app/webhooks/aiinbx/route.ts
export async function POST(request: Request) {
  const event = await verifyWebhookRequest(request, secret)
  return new Response(null, { status: 204 })
}
```
**Express**

Mount a raw body parser on the webhook route only:

```ts
app.post(
  "/webhooks/aiinbx",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const event = await verifyWebhook(
      req.body, // a Buffer, thanks to express.raw
      req.header("AIInbx-Signature")!,
      process.env.AI_INBX_WEBHOOK_SECRET!
    )
    res.status(204).end()
  }
)
```
A global `express.json()` mounted before this route will have consumed the stream — order matters.

**FastAPI**

```python
@app.post("/webhooks/aiinbx")
async def webhook(request: Request):
    body = await request.body()          # raw bytes
    event = client.webhooks.verify(
        body,
        request.headers["aiinbx-signature"],
        WEBHOOK_SECRET,
    )
    return Response(status_code=204)
```
Take `Request` rather than a Pydantic model — a declared body model parses the request before your code runs.

**Django**

```python
@csrf_exempt
def aiinbx_webhook(request):
    event = client.webhooks.verify(
        request.body,                    # raw bytes
        request.headers["AIInbx-Signature"],
        settings.AI_INBX_WEBHOOK_SECRET,
    )
    return HttpResponse(status=204)
```
## Clock skew

The five-minute window assumes your server's clock is roughly right. A host drifting by more than that rejects every event, which looks exactly like a wrong secret. If verification fails across the board and the secret is definitely correct, check NTP before anything else.

Both SDKs let you widen the window, in seconds, but treat that as a diagnostic rather than a fix. `0` accepts only the same second; a negative window is refused, so verification can't be switched off:

```ts TypeScript
await verifyWebhookRequest(request, secret, { tolerance: 600 })
```
```python Python
client.webhooks.verify(body, signature, secret, tolerance=600)
```
## After verifying

A valid signature proves the event came from AI Inbx. It doesn't promise you haven't seen it before — retries and manual replays deliver the same event again. Deduplicate on `event.id`:

```ts
const event = await verifyWebhookRequest(request, secret)

// Application-defined: atomically insert into durable storage by event.id.
// An existing event is a successful no-op; a storage failure must throw.
await enqueueOnce(event.id, event)
return new Response(null, { status: 204 })
```
Do not mark an event complete before its work succeeds. A separate “check then insert” can race under concurrent delivery; use a unique constraint or an equivalent atomic queue operation.

The pairing to remember: `event.id` for "have I seen this event", and an [idempotency key](../reference/conventions.md#idempotency) derived from it for "have I already sent the reply".
