# Workflows

## Set up a sending domain

```bash
aiinbx domains create --name yourdomain.com --region eu-central-1 --json   # returns the DNS records
# publish every record at the DNS host, then:
aiinbx domains verify <domain-id> --json
aiinbx doctor yourdomain.com --json    # what is still missing, and the exact fix for each
```

Verification follows DNS propagation. `doctor` reads the live zone, so run it again after the user changes a record. Don't guess from the records list.

## Develop a webhook handler locally

```bash
export AI_INBX_WEBHOOK_SECRET=$(aiinbx listen --print-secret)
# start the app with that env, then in the background:
aiinbx listen --forward-to localhost:3000/api/webhooks > listen.ndjson &
aiinbx send --from you@yourdomain.com --to you@yourdomain.com --subject test --text hi
# read listen.ndjson: one line per delivery with the local status and latency
kill %1
```

The handler verifies with `verifyWebhook(rawBody, signature, process.env.AI_INBX_WEBHOOK_SECRET)` exactly as in production. Only the secret differs. The body is the summary envelope. `data.email`, which a `full` endpoint is sent, is not included, so read the email by `data.email_id`. To replay recent events, add `--since 30m`.

## End-to-end test of a sign-up with an emailed code

```bash
started=$(date -u +%FT%TZ)
curl -s -X POST https://app.example.com/signup -d email=qa+$RUN@yourdomain.com
email=$(aiinbx wait --to "qa+$RUN@yourdomain.com" --since "$started" --timeout 2m --json) || exit 1
code=$(echo "$email" | jq -er .verification.code) || exit 1
```

- Use a unique address per run (`+tag`) so parallel runs don't take each other's mail.
- Exit 124 means nothing arrived in time (the error says so if the API was failing at the end); exit 1 with an API error means it never answered. On 124, check `aiinbx emails list --direction inbound --limit 5`, then `aiinbx doctor yourdomain.com` (MX record).

## Reply in a thread

```bash
aiinbx threads --query "invoice" --limit 5 --json
aiinbx threads retrieve thr_123 --json          # messages carry stripped_text
aiinbx threads reply thr_123 --text "Thanks, we'll send it today" --json
```

## CI

```bash
# AI_INBX_API_KEY comes from the CI secret store
aiinbx doctor --json || exit 1
aiinbx send --from ci@yourdomain.com --to team@example.com --subject "Deploy done" --text-file notes.md --json
```
