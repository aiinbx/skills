# Build an AI email agent

Build an AI email agent with signed webhooks, conversation threads, human review, and idempotent replies using the AI Inbx TypeScript SDK.

From https://docs.aiinbx.com/integrations/agents

An email agent receives a message, reads the conversation, decides whether to respond, and sends a reply. AI Inbx provides messages, threads, and signed events. Your application owns the decision, business context, and recovery of the workflow.

Start with the [quickstart](https://docs.aiinbx.com/quickstart.md) to configure a sender, install the SDK, and receive a verified webhook. For an overview of the email infrastructure and controls available to your agent, see [Email infrastructure for AI agents](https://aiinbx.com/use-cases/ai-email-agent).

For coding-agent access to these docs, see [Documentation for agents](https://docs.aiinbx.com/reference/agents.md). An assistant that only needs to send and read mail, with no worker of your own, can connect to the [MCP server](https://docs.aiinbx.com/mcp.md) or the [CLI](https://docs.aiinbx.com/cli.md), or get an [inbox](https://docs.aiinbx.com/guides/agent-inbox.md) of its own through the inbox server.

## Accept the event, then process it

Use the [production webhook handler](../guides/production.md#receive-events-durably) to verify and durably enqueue each event before returning `2xx`. Do not run a model inside the webhook request: delivery times out after 10 seconds.

The example below is a **worker outline**, not a complete runnable application. `draftOnce` and `markHandled` are application functions. The job runner must deduplicate by event ID, recover abandoned work, and serialize processing per thread. The answer is keyed on the email: `email.received` is sent again when the same email reaches another address of yours, and the email gets one answer.

```ts
import AIInbx from "aiinbx"
import type { WebhookEvent } from "aiinbx"

const aiinbx = new AIInbx()

async function handleEvent(event: WebhookEvent) {
  if (event.type !== "email.received") return
  if (event.data.category !== "human") return

  // Thread retrieval returns one page. Iterate to load all available messages.
  const messages = []
  for await (const message of aiinbx.threads.iterateMessages(event.data.thread_id)) {
    messages.push(message)
  }

  // Application-defined: evaluate policy, generate once, and persist the
  // approved request body before sending. Return that same body on retries.
  // Return null when a reply is not appropriate or needs human review.
  const payload = await draftOnce(event.data.email_id, event.space_id, messages)
  if (!payload) return

  // email_id: answer the email this event is about, even if more mail has
  // arrived on the thread since.
  const email = await aiinbx.threads.reply(
    event.data.thread_id,
    { ...payload, email_id: event.data.email_id },
    { idempotencyKey: `reply-${event.data.email_id}` }
  )
  await markHandled(event.id, email.id)
}
```
`draftOnce` must not regenerate a different answer on every worker retry. The same key with a different body causes an idempotency conflict. Persist the decision and payload, and reuse both after a crash. For very long conversations, build a bounded context window or a persisted summary rather than loading unlimited content into a model.

## Decide whether to answer

The `category` field helps exclude out-of-office replies, automated notifications, and other messages that should not receive an automatic answer. Treat `human` as one input to your policy, not proof that a message is safe or that its sender is authorized.

Some automated mail is still worth acting on. An out-of-office or auto-reply carries [`unavailable`](../guides/receiving.md#what-automated-mail-asks-for) — why its sender can't be reached, the day they're back, who to write to instead — so the agent can wait or turn to that colleague rather than answer the robot. A verification email — a sign-in, sign-up or reset — carries `verification`, its code and link, for an agent finishing a sign-up of its own.

Before drafting or sending:

- Resolve `event.space_id` to your customer and enforce their configured permissions.
- Check the sender, recipient, and [authentication verdicts](../guides/receiving.md#authentication-verdicts) against your application's policy.
- Exclude your own automated addresses and enforce a reply budget or cooldown per conversation.
- Check whether a person or another worker has already responded or taken ownership.
- Require review for actions outside the agent's authorized scope.

A valid webhook signature authenticates delivery from AI Inbx. It does not make the email body's instructions trustworthy. Keep email and attachment content separate from system instructions, and enforce tool permissions in application code.

## Read the conversation

`event.data.snippet` is a preview. Retrieve messages before answering, and account for [message pagination](../guides/threads.md#long-threads).

`stripped_text` removes recognized quoted history and repeated signatures — a sender's first signature in a thread stays, so the agent learns who is writing — while preserving inline answers where possible. Use it to reduce repeated context; retain access to the original text and headers when the distinction matters. See [What was written](../guides/receiving.md#what-was-written).

Prepared attachment text may be partial or unavailable. Check `preparation.status` and `warnings` before using it to answer a question about a document. Missing extraction is not evidence that the original file has no relevant content.

## Recover without duplicate replies

Webhook deduplication and send idempotency solve different problems:

| Mechanism | Responsibility |
| --- | --- |
| Unique event ID in durable storage | Avoid creating duplicate jobs for one event |
| Recoverable worker claim | Resume processing if a worker crashes |
| Persisted decision and payload | Keep retries consistent, including generated text |
| Stable reply idempotency key | Avoid sending another email when a send response is lost |
| Per-thread serialization or version check | Avoid conflicting replies to different events on one conversation |

Mark work complete after its effects succeed. If a worker crashes after sending but before recording completion, retry with the saved body and key. Inspect the reply response for suppressions and pacing just as you would for a new send.

## Hand off to a person

Keep drafts that require approval in your application's database or review queue. Send them only after approval is recorded. A scheduled send will eventually send without a human taking action, so it is not an approval gate.

To share a conversation with a support colleague, use `threads.forward`:

```ts
await aiinbx.threads.forward(
  threadId,
  { to: "support-leads@example.com", note: savedDecision.reason },
  { idempotencyKey: `handoff-${event.id}` }
)
```
Use an authorized handoff address and persist the reason so retries keep the same body. The forward creates a separate thread linked through `forward_of`; replying to the original thread still targets the original conversation. Record the handoff in your application so automated replies stop while a person is responsible.

## Keep business context in your application

Retrieve account data, product knowledge, and customer permissions from your own systems. An email's claims about an order or account are not authorization to modify it.

A reply can use a verified domain or a connected Gmail or Outlook mailbox. When automating a person's mailbox, make the scope of the automation and the route to human review clear in your product.

Continue with [Threads](../guides/threads.md), [email.received](../webhooks/events/email-received.md), and [Going to production](../guides/production.md).
