# Event reference

Every AI Inbx webhook event type, the envelope they share, what each one means, and how to handle email, thread, domain and mailbox events.

From https://docs.aiinbx.com/webhooks/events

Every webhook event arrives in the same envelope:

```json
{
  "id": "evt_...",
  "type": "email.received",
  "created_at": "2026-09-01T10:31:04Z",
  "space_id": null,
  "data": { }
}
```
`id` is stable across retries and replays — deduplicate on it. `space_id` is the [space](../guides/spaces.md) the resource is in, `null` for the workspace itself. `data` varies by `type`, and both SDKs discriminate on `type` so a type checker narrows it for you.

## Email

| Event | Fires when |
| --- | --- |
| [`email.received`](events/email-received.md) | Inbound mail arrives. |
| [`email.sent`](https://docs.aiinbx.com/webhooks/events/email-sent.md) | A message is handed to the mail provider. |
| [`email.delivered`](https://docs.aiinbx.com/webhooks/events/email-delivered.md) | The receiving server accepted it. |
| [`email.bounced`](https://docs.aiinbx.com/webhooks/events/email-bounced.md) | Delivery failed. |
| [`email.complained`](https://docs.aiinbx.com/webhooks/events/email-complained.md) | The recipient marked it as spam. |
| [`email.failed`](https://docs.aiinbx.com/webhooks/events/email-failed.md) | The provider refused it before any attempt. |
| [`email.unsubscribed`](https://docs.aiinbx.com/webhooks/events/email-unsubscribed.md) | The recipient opted out. |
| [`email.opened`](https://docs.aiinbx.com/webhooks/events/email-opened.md) | An open was tracked. |
| [`email.clicked`](https://docs.aiinbx.com/webhooks/events/email-clicked.md) | A tracked link was clicked. |

## Thread

| Event | Fires when |
| --- | --- |
| [`thread.created`](https://docs.aiinbx.com/webhooks/events/thread-created.md) | A new conversation started. |

## Domain

| Event | Fires when |
| --- | --- |
| [`domain.verified`](https://docs.aiinbx.com/webhooks/events/domain-verified.md) | The domain’s DKIM identity was verified. |
| [`domain.at_risk`](https://docs.aiinbx.com/webhooks/events/domain-at-risk.md) | A verified domain's DKIM record went missing; it still sends for 72 hours. |
| [`domain.restored`](https://docs.aiinbx.com/webhooks/events/domain-restored.md) | The missing DKIM record has been back for an hour before the domain was lost. |
| [`domain.lost`](https://docs.aiinbx.com/webhooks/events/domain-lost.md) | A verified domain stopped authenticating and can no longer send. |

## Mailbox

| Event | Fires when |
| --- | --- |
| [`mailbox.connected`](https://docs.aiinbx.com/webhooks/events/mailbox-connected.md) | A customer authorized a mailbox. |
| [`mailbox.needs_reauth`](https://docs.aiinbx.com/webhooks/events/mailbox-needs-reauth.md) | A mailbox's authorization stopped working. |
| [`mailbox.disconnected`](https://docs.aiinbx.com/webhooks/events/mailbox-disconnected.md) | A mailbox was removed. |

## Handle events in a worker

[Verify and durably enqueue](../guides/production.md#receive-events-durably) each event in your HTTP handler. In a worker, branch on `event.type` to select the appropriate application behavior:

| Type | Typical action |
| --- | --- |
| `email.received` | Load the message or thread and evaluate whether to respond |
| `email.bounced` | Record affected recipients, why each bounced, and which were suppressed |
| `email.failed` | Surface the provider's reason; the message never went out |
| `email.complained` | Record the complaint and review the recipient’s suppression state |
| `email.unsubscribed` | Update the matching preference in your application |
| `domain.at_risk` | Show the customer the missing DKIM record and `stops_sending_at` |
| `mailbox.needs_reauth` | Ask the mailbox owner to reconnect |

Make each action recoverable and idempotent. Mark work complete after it succeeds. Unknown event types should not crash the handler; record the type for diagnosis and ignore it until the application supports it.

**[Verifying requests](verifying.md)**

How to authenticate each delivery before storing or processing it.
