# React Email

Send React Email templates through AI Inbx: pass a component as `react` and the TypeScript SDK renders its HTML. Keep passing `text` beside it.

From https://docs.aiinbx.com/integrations/react-email

`emails.send` takes a `react` field. Pass a component and the SDK renders it to HTML before the request goes out, so there's no render step in your code.

## Install

`@react-email/render` is an optional peer dependency — the SDK imports it lazily and only when you actually use `react`. Nothing to install if you don't.

```bash
npm install aiinbx @react-email/render @react-email/components
```
## Send a component

```tsx
import { aiinbx } from "@/lib/aiinbx"
import { WelcomeEmail } from "@/emails/welcome"

await aiinbx.emails.send({
  from: { name: "Acme", address: "hello@acme.com" },
  to: "ada@example.com",
  subject: "Welcome to Acme",
  react: <WelcomeEmail name="Ada" />,
  text: "Your account is ready. Visit https://acme.com to get started.",
})
```
`react` replaces `html` — pass one or the other, not both.

> [!TIP]
> Keep passing `text`. React Email renders HTML only, and a message with no text part looks worse in plain-text clients and scores worse with spam filters. It's two lines, and it's the version a screen reader is most likely to read.

## Replies take HTML

`threads.reply` accepts `html` and `text`, but not `react`. Render it yourself when you need a component on a reply:

```tsx
import { render } from "@react-email/render"

await aiinbx.threads.reply(threadId, {
  html: await render(<ReplyEmail body={answer} />),
  text: answer,
})
```
## Failure mode

If `@react-email/render` isn't installed, the send throws before any request is made:

```
Failed to render React email. Install `@react-email/render`.
```
That's a missing dependency, not an API error — it will never show up in your delivery logs, because nothing was sent.

## Inline images

React Email won't upload your images. A `<img src="cid:logo">` in the component needs the matching [attachment](../guides/sending.md#attachments) on the same send:

```tsx
await aiinbx.emails.send({
  from: "Acme <hello@acme.com>",
  to: "ada@example.com",
  subject: "Your receipt",
  react: <Receipt items={items} />,
  text: receiptText,
  attachments: [
    {
      filename: "logo.png",
      content_type: "image/png",
      cid: "logo",
      content: logoBase64,
    },
  ],
})
```
Absolute `https://` URLs work too and keep the message smaller — at the cost of not rendering until the client loads remote images, which many don't by default.
