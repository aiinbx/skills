# Attachments

Download what arrived, or read the prepared Markdown of each attachment so a model can use a PDF or a document without you running a parser.

From https://docs.aiinbx.com/guides/attachments

Inbound attachments come back two ways. There's the **original file**, byte for byte, and there's the **prepared text** — the same document extracted to Markdown or plain text so it can go straight into a prompt.

Sending attachments is covered in [Sending](https://docs.aiinbx.com/guides/sending.md#attachments); this page is about what arrives.

## What's on a message

Retrieve an email and each attachment carries its metadata plus two URLs:

```ts
const email = await aiinbx.emails.retrieve("eml_...")

for (const attachment of email.attachments) {
  console.log(attachment.filename, attachment.content_type, attachment.size)
  console.log(attachment.preparation?.status, attachment.preparation?.format)
}
```
| Prop | Type | Default | Description |
| - | - | - | - |
| `id` | `string` | - | att_… |
| `filename` | `string` | - | As the sender named it. |
| `content_type` | `string` | - | The declared media type. |
| `size` | `number` | - | Bytes. |
| `cid` | `string \| null` | - | Content-ID when the attachment is inline in the HTML body. |
| `download_url` | `string \| null` | - | Signed URL for the original file. Opens without your API key. Null when the bytes are gone. |
| `expires_at` | `string \| null` | - | When download_url and preparation.content_url stop working — one hour after the response. |
| `preparation` | `object \| null` | - | Extraction result, or null when nothing was attempted. |

> [!WARNING]
> `download_url` and `content_url` are signed: anyone holding one can open it, no API key needed, until `expires_at` — one hour after the response. Hand them to a worker or a model, but don't persist them in your database and expect them to work tomorrow. Store the `id` instead and resolve it on demand.

## Downloading the original

With a fresh email in hand, `GET` the `download_url` — no headers needed, and none wanted: don't send your API key to `download_url` or `content_url`. The signature in the URL is the credential, and storage answers a request that also carries an `Authorization` header with `400`, so an HTTP client that adds your key to every request needs a bare one here. With only a stored `id`, the download endpoint takes your API key and redirects to a newly signed URL. Both SDKs resolve the redirect for you:

```ts TypeScript
const bytes = new Uint8Array(await aiinbx.attachments.download("att_..."))
```
```python Python
data = client.attachments.download("att_...")
```
```bash curl
curl -L https://api.aiinbx.com/api/v2/attachments/att_... \
  -H "Authorization: Bearer $AI_INBX_API_KEY" \
  -o report.pdf
```
The endpoint answers `307` with a `Location` header pointing at a signed URL — hence `-L` on the curl. A `HEAD` on the same path returns the metadata in headers (`Content-Type`, `Content-Length`, `Content-Disposition`) without transferring the file, which is the cheap way to check a size before committing to a download.

## Prepared text

Documents that can be read as text are extracted on arrival. `preparation` tells you how that went:

| Prop | Type | Default | Description |
| - | - | - | - |
| `status` | `"ready" \| "partial" \| "unsupported" \| "failed"` | - | Whether the extraction produced usable text. |
| `format` | `"markdown" \| "text" \| null` | - | Markdown keeps document structure; text is a flat rendering. |
| `pages` | `number \| null` | - | Page count, where the format has pages. |
| `warnings` | `string[]` | - | What was skipped or approximated — worth logging on partial results. |
| `content_url` | `string \| null` | - | Signed URL for the extracted text, good until the attachment's expires_at. Null when there is none. |
| `text?` | `string \| null` | - | The extracted text inline — only present when you ask for it. |

| Status | What it means |
| --- | --- |
| `ready` | The whole document was extracted. |
| `partial` | Some of it came through; `warnings` says what didn't. Usable, with care. A scanned page too degraded to read reliably is skipped rather than guessed at. |
| `unsupported` | The file type has no text to extract — an image, an archive, a binary. |
| `failed` | Extraction was attempted and didn't work. The original is still downloadable. |

### Reading it

Fetch the text on its own:

```ts TypeScript
const markdown = await aiinbx.attachments.content("att_...")
```
```python Python
markdown = client.attachments.content("att_...")
```
```bash curl
curl -L https://api.aiinbx.com/api/v2/attachments/att_.../content \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
Or pull every attachment's text down with the message in one call, which is usually what you want before a model call:

```ts TypeScript
const email = await aiinbx.emails.retrieve("eml_...", {
  include: ["attachment_content"],
})

const documents = email.attachments
  .filter((a) => a.preparation?.text)
  .map((a) => `## ${a.filename}\n\n${a.preparation!.text}`)
```
```python Python
email = client.emails.retrieve("eml_...", include=["attachment_content"])

documents = [
    f"## {a['filename']}\n\n{prep['text']}"
    for a in email["attachments"]
    if (prep := a["preparation"]) and prep.get("text")
]
```
```bash curl
curl "https://api.aiinbx.com/api/v2/emails/eml_...?include=attachment_content" \
  -H "Authorization: Bearer $AI_INBX_API_KEY"
```
An attachment with no prepared text — because it's `unsupported`, `failed`, or still being worked on — comes back with `text: null`. Asking for content on one that was never prepared returns `404` [`not_prepared`](https://docs.aiinbx.com/reference/errors.md#not_prepared).

## Handling an incoming document

```ts
if (event.type !== "email.received") return

const email = await aiinbx.emails.retrieve(event.data.email_id, {
  include: ["attachment_content"],
})

for (const attachment of email.attachments) {
  const prep = attachment.preparation

  if (prep?.status === "ready" || prep?.status === "partial") {
    if (prep.warnings.length) {
      console.warn(attachment.filename, prep.warnings)
    }
    await ingest(attachment.filename, prep.text!)
    continue
  }

  // No text to read — keep the original for a human.
  const file = await aiinbx.attachments.download(attachment.id)
  await archive(attachment.filename, file)
}
```
The `email.received` webhook also carries a compact attachment list, so you can decide whether a message is worth fetching before you fetch it.

## Timing

Preparation runs as the message is ingested, so by the time `email.received` reaches you the result is normally already there. A large document can still be in flight — `preparation` is `null` in that case rather than an error. If you're handling big files, re-fetch the email rather than treating `null` as "no text".
