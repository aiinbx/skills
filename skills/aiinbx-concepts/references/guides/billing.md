# Plans and billing

What each plan includes, exactly what counts against it, how extra usage and spending caps work, and what the API answers at a limit.

From https://docs.aiinbx.com/guides/billing

A workspace is on one monthly plan. The plan includes an allowance of emails and advanced attachment processing pages, and sets how many custom domains, mailboxes, and pacing rules the workspace can hold. Plans differ only in those numbers.

All prices are in USD and exclude applicable taxes.

## Plans

|                                | Developer       | Growth          | Scale             |
| ------------------------------ | --------------- | --------------- | ----------------- |
| Price                          | $5/month        | $29/month       | $99/month         |
| Emails included                | 5,000           | 30,000          | 100,000           |
| Custom domains                 | 50              | Unlimited       | Unlimited         |
| Managed mailboxes              | 100             | Unlimited       | Unlimited         |
| Active custom pacing rules     | 100             | 1,000           | Unlimited         |
| Advanced attachment processing | 100 pages/month | 500 pages/month | 2,000 pages/month |

Choose or change a plan under **Settings → Billing** in the [console](https://aiinbx.com/app).

## Extra usage

Past the included allowance, usage is pay-as-you-go at the same rates on every plan:

| Add-on                         | Rate                            |
| ------------------------------ | ------------------------------- |
| Additional emails              | $0.001 per email ($1 per 1,000) |
| Advanced attachment processing | $0.01 per page ($10 per 1,000)  |

You pay only for the additional emails and pages actually used. There are no prepaid blocks and no monthly commitment; the amount is added to the monthly invoice. A month with 750 additional emails and 23 additional processed pages costs $0.75 + $0.23 = **$0.98**.

Extra usage is **opt-in**. Until you turn it on in billing settings, a plan stops at its allowance. Once it is on, you can set a monthly spending cap — one for emails, one for pages — and usage stops at the cap the same way it would at the allowance.

## What counts

### Emails

Every email that goes through AI Inbx counts as one, inbound or outbound.

- **Outbound**: an email you send counts once, when we take it — so one held by [pacing](pacing.md) or [scheduled](https://docs.aiinbx.com/guides/scheduling.md) already has its place in your allowance, and cannot be refused for it later. An email that is never sent gives it back: [canceled](https://docs.aiinbx.com/guides/scheduling.md), or failed before the sending provider accepted it. An email that was suppressed, or refused with an error, never counted. Our own retries do not count.
- **Inbound**: an email counts once, when it is stored — on a domain or in a connected mailbox.
- **Connected mailboxes**: everything we read from the mailbox counts the same way, including the last seven days read when it is connected and mail sent from its own client (Gmail, Outlook).

### Advanced attachment processing

Advanced processing reads the scanned pages of a PDF, and is counted per page. Standard text extraction — a PDF with a text layer, an office document — does not use the allowance. See [Attachments](attachments.md) for what is prepared and how.

Only pages that came back with text count. A reading that failed, a page skipped as too degraded to read reliably, and our own retries cost nothing.

### Domains, mailboxes, and pacing rules

These are limits on how many you hold at once, not monthly counts. Removing one frees its place immediately.

| Limit                      | Counts                                                                  | Does not count                                                                                           |
| -------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Custom domains             | Each domain you add, verified or not. A wildcard `*.example.com` is one | The provided `<slug>.aiinbx.app` domain; subdomains created under a [wildcard](domains.md) you hold |
| Managed mailboxes          | Every connected mailbox, including one waiting for re-authorization     | Disconnected mailboxes                                                                                   |
| Active custom pacing rules | Rules you created that are enabled                                      | Built-in rules; disabled rules                                                                           |

## Reset timing

Included emails and pages reset at the start of each billing month — the monthly anniversary of the day the plan started, not the first of the calendar month. Spending caps apply to the same period.

## At a limit

Receiving is never blocked. What stops depends on which allowance ran out:

| Reached                               | What happens                                                                                                                                        |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Email allowance, or its cap           | Sending is refused. Inbound mail keeps arriving, is stored, and fires webhooks as usual.                                                            |
| Page allowance, or its cap            | Further advanced processing pauses. Email delivery continues, subject to the email allowance and cap. Attachments stay accessible and downloadable. |
| Domain, mailbox, or pacing rule limit | Creating one more is refused — for a pacing rule, so is enabling a disabled one. Everything you already hold keeps working.                         |

A refused request answers `402` with `application/problem+json`. Two codes cover every case:

| Code                                                       | Meaning                                                                                                              |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| [`plan_required`](https://docs.aiinbx.com/reference/errors.md#plan_required)         | The workspace has no plan. Every metered request is refused until one is chosen.                                     |
| [`allowance_reached`](https://docs.aiinbx.com/reference/errors.md#allowance_reached) | The plan's allowance for that feature is used up, and extra usage is off or its cap is reached. `detail` says which. |

```json
{
  "type": "https://docs.aiinbx.com/problems/allowance_reached",
  "title": "Payment Required",
  "status": 402,
  "detail": "This workspace has used its plan's emails for the month. Upgrade, or turn on extra usage in billing settings.",
  "code": "allowance_reached",
  "request_id": "8f2c4a1b-..."
}
```
Neither is worth retrying unchanged: the answer stays the same until the plan, the extra-usage setting, or the cap changes, or the month resets. Treat a `402` on send as a signal to alert a person, and keep the email queued on your side. The full body shape is in the [error reference](https://docs.aiinbx.com/reference/errors.md).
