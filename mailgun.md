# Mailgun API Reference

Source: https://documentation.mailgun.com/docs/mailgun (fetched 2026-06-27)

Mailgun is an email infrastructure platform for sending, receiving, and tracking transactional and bulk email via HTTP API or SMTP.

---

## Regions & Base URLs

Message data never leaves the region in which it is processed — pick the region that matches your domain.

| Region | Base URL |
|---|---|
| US | `https://api.mailgun.net` |
| EU | `https://api.eu.mailgun.net` |

All endpoints below are shown without the host — prepend the base URL for the region your domain lives in.

---

## Authentication

Mailgun uses **HTTP Basic Auth**.

- **Username:** `api`
- **Password:** your private API key

```bash
curl --user 'api:YOUR_API_KEY' https://api.mailgun.net/v3/YOUR_DOMAIN/messages ...
```

### Key types

| Key | Scope | Where |
|---|---|---|
| Primary Account API Key | Full CRUD across all domains | Dashboard → Account Settings → API Keys → Private API key |
| Domain Sending Key | Restricted to `/messages` and `/messages.mime` on a single domain | Dashboard → Sending → Domains → \[domain\] → Domain Settings → Sending API keys |

Domain Sending Keys cannot be re-displayed after creation — store them on creation or regenerate.

---

## Response format

- All responses are JSON.
- Dates are RFC-2822 (e.g. `Thu, 13 Oct 2011 18:02:00 +0000`). Avoid abbreviated zones (`EST`, `CET`) — they're ambiguous.

### Status codes

| Code | Meaning |
|---|---|
| 200 | OK |
| 400 | Bad request (JSON body contains error detail) |
| 401 | Unauthorized (bad API key) |
| 403 | Forbidden (insufficient permissions) |
| 404 | Not found |
| 429 | Rate limited |
| 500 | Internal server error |

### Rate-limit headers

| Header | Meaning |
|---|---|
| `X-RateLimit-Limit` | Max calls in the current window |
| `X-RateLimit-Remaining` | Calls left in the window |
| `X-RateLimit-Reset` | Unix ms until the window resets |

---

## Quickstart — send your first email

1. Sign up at https://signup.mailgun.com/new/signup.
2. Copy your **Private API key** from Dashboard → Account Settings → API Keys.
3. On a sandbox domain, add and verify a recipient address (Mailgun emails a confirmation link).
4. Send:

```bash
curl --user 'api:YOUR_API_KEY' \
  https://api.mailgun.net/v3/YOUR_SANDBOX_DOMAIN/messages \
  -F from='Test <postmaster@YOUR_SANDBOX_DOMAIN>' \
  -F to='your-email@example.com' \
  -F subject='Hello!' \
  -F text='Test message'
```

Expected response:

```json
{"id":"<...>@YOUR_SANDBOX_DOMAIN","message":"Queued. Thank you."}
```

For production, add a custom domain and configure SPF, DKIM, MX, and tracking CNAME records as instructed in the domain's setup page.

---

## Messages API

### Send a message (form-encoded)

`POST /v3/{domain}/messages`

#### Core parameters

| Param | Description |
|---|---|
| `from` | Sender address. `Display Name <user@domain>` allowed. Required. |
| `to` | Recipient(s). Comma-separated or repeated. Required. |
| `cc` | CC recipient(s). |
| `bcc` | BCC recipient(s). |
| `subject` | Subject line. |
| `text` | Plain-text body. |
| `html` | HTML body. |
| `amp-html` | AMP for Email body. |
| `attachment` | File attachment. Use `-F attachment=@/path/to/file` (repeat for multiple). |
| `inline` | Inline attachment (referenced by `cid:` in HTML). |
| `template` | Stored template name. |
| `t:version` | Template version tag. |
| `t:text` | `yes` to render plain-text from template. |
| `t:variables` | JSON map of template variables. |
| `recipient-variables` | JSON map keyed by recipient email for batch personalization. |

#### Custom headers (`h:` prefix)

Anything with `h:` prefix is added as an outbound MIME header:

```
-F h:X-My-Header='value'
-F h:Reply-To='support@example.com'
```

#### Sending options (`o:` prefix)

| Option | Values | Effect |
|---|---|---|
| `o:tag` | string (repeatable) | Tag for analytics/aggregation. |
| `o:dkim` | `yes` / `no` | Toggle DKIM signing. |
| `o:deliverytime` | RFC-2822 date | Schedule for later delivery (max 3 days out). |
| `o:deliverytime-optimize-period` | `24h`…`72h` | Send-time optimization window. |
| `o:time-zone-localize` | `HH:mm` | Deliver at recipient's local time. |
| `o:testmode` | `yes` / `true` | Accepted but not sent. **You are still billed.** |
| `o:tracking` | `yes` / `no` | Master tracking switch. |
| `o:tracking-clicks` | `yes` / `no` / `htmlonly` | Click tracking. |
| `o:tracking-opens` | `yes` / `no` | Open tracking (HTML emails only). |
| `o:require-tls` | `yes` / `no` | Require TLS at recipient MTA. |
| `o:skip-verification` | `yes` / `no` | Skip recipient TLS cert verification. |

#### Custom message variables (`v:` prefix)

Attach arbitrary JSON-serializable data echoed back in webhook events:

```
-F v:my-var='{"order_id":"12345"}'
```

#### Example: full-featured send

```bash
curl --user 'api:YOUR_API_KEY' \
  https://api.mailgun.net/v3/mg.example.com/messages \
  -F from='Sales <sales@mg.example.com>' \
  -F to='alice@example.com' \
  -F cc='bob@example.com' \
  -F subject='Your order' \
  -F text='Plain text fallback' \
  -F html='<h1>Hi Alice</h1>' \
  -F attachment=@/tmp/invoice.pdf \
  -F o:tag='order-confirmation' \
  -F o:tracking-clicks='yes' \
  -F o:deliverytime='Fri, 27 Jun 2026 12:00:00 +0000' \
  -F h:Reply-To='support@example.com' \
  -F v:order_id='12345'
```

### Send a MIME message

`POST /v3/{domain}/messages.mime`

Use when you've already assembled a raw MIME body (e.g. from your own composer):

```bash
curl --user 'api:YOUR_API_KEY' \
  https://api.mailgun.net/v3/mg.example.com/messages.mime \
  -F to='alice@example.com' \
  -F message=@/tmp/raw.eml
```

### Batch sending with `recipient-variables`

One API call, personalized per recipient. The `to` list is fanned out into separate sends with each recipient seeing only their own address:

```bash
curl --user 'api:YOUR_API_KEY' \
  https://api.mailgun.net/v3/mg.example.com/messages \
  -F from='Sales <sales@mg.example.com>' \
  -F to='alice@example.com' \
  -F to='bob@example.com' \
  -F subject='Hello %recipient.first%' \
  -F text='Hi %recipient.first%, your code is %recipient.code%.' \
  -F recipient-variables='{
        "alice@example.com": {"first":"Alice","code":"A1"},
        "bob@example.com":   {"first":"Bob",  "code":"B2"}
      }'
```

Up to 1,000 recipients per call.

---

## Tracking

Tracking categories: **delivery, opens, clicks, unsubscribes, spam complaints, failures.**

### Click & open tracking

Configure per-message with `o:tracking`, `o:tracking-clicks`, `o:tracking-opens`. Domain-wide defaults live in the Control Panel under domain settings.

### Unsubscribe variables

Embed these in HTML/text bodies; Mailgun rewrites them per recipient:

| Variable | Scope |
|---|---|
| `%unsubscribe_url%` | All mail from this domain |
| `%tag_unsubscribe_url%` | Specific tags only |
| `%mailing_list_unsubscribe_url%` | The mailing list of this send |

Once a user clicks one, Mailgun suppresses future sends to that address automatically.

---

## Webhooks

Configure URLs via the Webhooks API or Control Panel. Mailgun POSTs JSON for each tracked event.

### Event types

`delivered`, `opened`, `clicked`, `unsubscribed`, `complained`, `failed` (permanent or temporary), `accepted`, `rejected`.

### Signature verification

Every webhook payload includes a `signature` object:

```json
{
  "signature": {
    "timestamp": "1529006854",
    "token":     "a8ce0edb2dd8...",
    "signature": "d2271d12299f6592d9d4..."
  },
  "event-data": { ... }
}
```

Verify with HMAC-SHA256 of `timestamp + token` using your **HTTP webhook signing key** (Account Settings → API Security):

```python
import hashlib, hmac
def verify(signing_key, timestamp, token, signature):
    digest = hmac.new(
        signing_key.encode(),
        f"{timestamp}{token}".encode(),
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(digest, signature)
```

Reject requests where the timestamp is more than ~10 minutes stale to block replays.

---

## API surface (selected categories)

All paths are relative to the regional base URL.

| Category | Base path | Use |
|---|---|---|
| Messages | `/v3/{domain}/messages`, `/v3/{domain}/messages.mime` | Send mail (see above). |
| Domains | `/v4/domains`, `/v4/domains/{name}` | List, create, update, verify domains. |
| Events | `/v3/{domain}/events` | Paginated event log (delivered/opened/etc.). |
| Stats | `/v3/{domain}/stats/total` | Aggregated counters. |
| Suppressions | `/v3/{domain}/bounces`, `/unsubscribes`, `/complaints` | Manage suppression lists. |
| Routes | `/v3/routes` | Inbound routing rules. |
| Webhooks | `/v3/domains/{domain}/webhooks` | Configure event callbacks. |
| Mailing Lists | `/v3/lists`, `/v3/lists/{address}/members` | Manage lists & members. |
| Templates | `/v3/{domain}/templates` | Stored Handlebars templates and versions. |
| IPs | `/v3/ips`, `/v3/domains/{domain}/ips` | Dedicated IP and IP-pool management. |
| Tags | `/v3/{domain}/tags` | Tag analytics and management. |

---

## Notes & gotchas

- **Test mode is billed.** `o:testmode=yes` still counts against your quota.
- **Scheduled delivery cap:** `o:deliverytime` accepts up to 3 days in the future.
- **Sandbox domains** require recipients to be pre-authorized — fine for dev, useless for prod.
- **Region lock-in:** A domain created in the EU region can only be managed via `api.eu.mailgun.net`. Calling the US host with an EU domain returns 404.
- **Lost a Domain Sending Key?** It can't be re-displayed; generate a new one.
- **Display names** in `from` should be quoted in JSON/code to avoid header injection (`From: Display Name <addr>`).

---

## Useful links

- Docs home: https://documentation.mailgun.com/docs/mailgun
- API overview: https://documentation.mailgun.com/docs/mailgun/api-reference/api-overview
- Authentication: https://documentation.mailgun.com/docs/mailgun/api-reference/mg-auth
- Quickstart: https://documentation.mailgun.com/docs/mailgun/quickstart
- User manual: https://documentation.mailgun.com/docs/mailgun/user-manual/intro
- Control panel: https://app.mailgun.com/app/dashboard
- Help center: https://help.mailgun.com/hc/en-us
