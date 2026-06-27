# Stripe API Reference

A practical reference for using the Stripe APIs — what they cover, how auth works, how to call endpoints, the modern payment flows, Connect, webhooks, and the gotchas that show up in production.

Pulled from `docs.stripe.com` (current as of 2026-06-27, API version `2026-06-24.dahlia`).

---

## 1. API Overview

- **Base URL:** `https://api.stripe.com`
- **File uploads:** `https://files.stripe.com` (separate host — easy to miss)
- **Request format:** `application/x-www-form-urlencoded`. Nested objects use bracket notation:
  ```
  items[0][price]=price_123
  items[0][quantity]=2
  metadata[order_id]=ord_42
  ```
- **Response format:** JSON, conventional HTTP status codes.
- **No bulk endpoints** — one object per request.
- **Test vs Live:** determined by the key prefix (`sk_test_…` vs `sk_live_…`). No URL change.
- **Sandboxes** (replaces single "test mode"): one account can spin up multiple isolated sandboxes, each with its own keys, webhook endpoints, and Connect setup. Switch via the Dashboard account picker.

### API versioning
- Current named major: `2026-06-24.dahlia`. Stripe ships backward-compatible monthly releases inside a named major; named majors introduce breaking changes.
- Per-request override: `Stripe-Version: 2026-06-24.dahlia`.
- Without the header, requests use the **account's pinned default** (set in Workbench → API versioning).
- SDK pinning: Ruby (v9+), Python (v6+), PHP (v11+), Node (v12+) pin to whatever was current when the SDK released. Java / Go / .NET always pin to their release-date version regardless of account default. Older SDKs use the account default.
- Best practice: pin in Workbench, then upgrade in lockstep with SDK upgrades. Don't rely on the account default in code.

### Pagination
- Cursor-based, on every `list` endpoint.
- Params: `limit` (1–100, default 10), `starting_after=<id>` (forward), `ending_before=<id>` (backward — mutually exclusive).
- Response shape:
  ```json
  { "object": "list", "data": [...], "has_more": true, "url": "/v1/customers" }
  ```
- SDKs ship auto-pagination helpers (e.g. Node `stripe.customers.list({limit:100}).autoPagingEach(...)`).

### Expansion
- `expand[]=<dot.path>` lets you inline related objects in the same response.
  ```
  expand[]=latest_invoice.payment_intent
  expand[]=customer.default_source
  ```
- Up to 4 levels deep.

### Search
- A subset of resources (`customers`, `charges`, `invoices`, `payment_intents`, `subscriptions`, `prices`, `products`) support `/search` with Stripe's query language:
  ```
  GET /v1/customers/search?query=email:'jenny@example.com' AND metadata['order_id']:'42'
  ```
- Note: search results are eventually consistent (lag ~1 min).

---

## 2. Authentication

Stripe uses HTTP Basic auth with the API key as the username and empty password, or equivalent `Authorization: Bearer <key>`.

```bash
curl https://api.stripe.com/v1/customers \
  -u sk_test_4eC39HqLyjWDarjtT1zdp7dc:
# equivalent:
curl https://api.stripe.com/v1/customers \
  -H "Authorization: Bearer sk_test_4eC39HqLyjWDarjtT1zdp7dc"
```

### Key types
| Prefix | Type | Use |
|--|--|--|
| `pk_test_…` / `pk_live_…` | **Publishable** | Safe in client code (Stripe.js, mobile). Cannot read or mutate sensitive data. |
| `sk_test_…` / `sk_live_…` | **Secret** | Full account access. Server-side only. |
| `rk_test_…` / `rk_live_…` | **Restricted** | Per-resource scoped permissions. Recommended over secret keys for new services. |
| `sk_org_…` | **Organization** | Operate across multiple accounts at the org level. |
| `whsec_…` | **Webhook signing secret** | HMAC verification only — not a request credential. |

### Connect: acting on a connected account
Add the `Stripe-Account` header to make a request as a connected account:
```bash
curl https://api.stripe.com/v1/charges \
  -u sk_test_…: \
  -H "Stripe-Account: acct_1MZx9Y2eZvKYlo2C" \
  -d amount=2000 -d currency=usd -d source=tok_visa
```
SDKs: `stripe.charges.create(params, { stripeAccount: 'acct_…' })`.

### Key rotation
- Rolling a key in the Dashboard supports a **7-day overlap** — old + new both work, so you can deploy without downtime.
- Store keys in a secrets manager; never commit; rotate when employees leave.

---

## 3. Idempotency

- Header: `Idempotency-Key: <up to 255 chars>` on any `POST`.
- Stripe caches the **status code and body** of the first response and replays it for retries with the same key.
- Retention: **at least 24 hours**.
- Reusing the same key with **different parameters** → `idempotency_error` (HTTP 400). Always generate a fresh UUID per logical operation.
- Never put PII or sequential counters in the key. V4 UUIDs are ideal.
- SDKs accept `{ idempotencyKey: '...' }` as a per-request option.

---

## 4. Errors

All errors return an `error` object:
```json
{
  "error": {
    "type": "card_error",
    "code": "card_declined",
    "decline_code": "insufficient_funds",
    "message": "Your card has insufficient funds.",
    "param": "source",
    "payment_intent": { "id": "pi_..." },
    "doc_url": "https://stripe.com/docs/error-codes/card-declined",
    "request_log_url": "https://dashboard.stripe.com/test/logs/req_..."
  }
}
```

### Error types
| Type | When |
|--|--|
| `card_error` | Card-specific issue (decline, expired, CVC). Show `message` to user. |
| `invalid_request_error` | Bad params, missing fields, wrong resource ID. |
| `api_error` | Stripe-side problem. Retry with backoff. |
| `idempotency_error` | Same key, different params. |
| `rate_limit_error` | Too many requests. Backoff + jitter. |
| `authentication_error` | Bad/missing API key. |

### HTTP status mapping
- `200` OK
- `400` invalid request
- `401` bad auth
- `402` request failed (card declined — payment-specific)
- `403` not permitted
- `404` not found
- `409` conflict / concurrent or idempotency mismatch
- `424` external decline (e.g. issuer)
- `429` rate limited
- `5xx` Stripe error — retry

### Decline codes
Common ones: `generic_decline`, `insufficient_funds`, `lost_card`, `stolen_card`, `expired_card`, `incorrect_cvc`, `processing_error`, `card_velocity_exceeded`, `do_not_honor`, `fraudulent`, `authentication_required`.

**UX gotcha:** never expose `lost_card` / `stolen_card` / `fraudulent` to the cardholder — show as generic decline.

---

## 5. Core payment flow (PaymentIntents)

PaymentIntents are the modern primitive — they handle SCA / 3D Secure, async confirmation, and multiple payment methods natively. Use them instead of legacy Charges.

### Lifecycle
```
requires_payment_method
  → requires_confirmation
  → requires_action       (e.g. 3DS challenge)
  → processing
  → requires_capture      (manual capture only)
  → succeeded | canceled
```

### Standard flow with Payment Element
1. **Server**: create the intent and return its `client_secret`.
   ```js
   const intent = await stripe.paymentIntents.create({
     amount: 2000,
     currency: 'usd',
     automatic_payment_methods: { enabled: true },
     metadata: { order_id: 'ord_42' },
   });
   res.json({ clientSecret: intent.client_secret });
   ```
2. **Client**: mount the Payment Element bound to that secret.
   ```js
   const stripe = await loadStripe('pk_test_…');
   const elements = stripe.elements({ clientSecret });
   elements.create('payment').mount('#payment-element');

   // on submit:
   const { error } = await stripe.confirmPayment({
     elements,
     confirmParams: { return_url: 'https://example.com/return' },
   });
   ```
3. **Return URL**: optionally `stripe.retrievePaymentIntent(clientSecret)` to inspect final status.
4. **Webhook**: trust `payment_intent.succeeded` for fulfillment — never trust the client.

### Key fields
- `amount` (integer, smallest currency unit), `currency`, `customer`, `payment_method`, `payment_method_types[]`, `automatic_payment_methods.enabled`
- `confirmation_method`: `automatic` (default) or `manual`
- `capture_method`: `automatic` | `automatic_async` | `manual`
- `setup_future_usage`: `on_session` | `off_session` (save the PM for later)
- `off_session: true` when charging a saved PM without the customer present
- Connect: `transfer_data.destination`, `application_fee_amount`, `on_behalf_of`
- `statement_descriptor`, `receipt_email`, `description`, `metadata`

### SetupIntents
Same shape as PaymentIntents but **no charge** — used to save a payment method for future off-session billing.
```js
await stripe.setupIntents.create({
  customer: 'cus_…',
  usage: 'off_session',
  automatic_payment_methods: { enabled: true },
});
```

### Amount gotcha
`amount` is in the **smallest currency unit**:
- `2000` USD = $20.00
- `2000` JPY = ¥2000 (zero-decimal currency — not ¥20)
- `2000` BHD = 2.000 BHD (three-decimal)

---

## 6. Resources

### Customers — `/v1/customers`
Endpoints: `create`, `retrieve`, `update`, `delete`, `list`, `search`.
Key fields: `email`, `name`, `phone`, `address`, `shipping`, `metadata`, `balance`, `currency`, `invoice_settings.default_payment_method`, `tax_exempt`, `tax_id_data[]`, `preferred_locales`.

```bash
curl https://api.stripe.com/v1/customers -u sk_test_…: \
  -d email=jenny@example.com -d name="Jenny" \
  -d "metadata[crm_id]=42"
```

### PaymentMethods — `/v1/payment_methods`
Endpoints: `create`, `retrieve`, `update`, `list`, `attach`, `detach`.
- Per-customer list: `GET /v1/customers/:id/payment_methods`.
- **Raw card data cannot be sent from your server** unless you're PCI-DSS Level 1. Always tokenize via Stripe.js → get a `pm_…` ID → attach.
- Events: `payment_method.attached`, `.detached`, `.updated`, `.automatically_updated` (network-driven card updates — important for saved-card freshness).

### Charges (legacy) — `/v1/charges`
Don't use for new integrations. Useful only when reading historical data or working with very old Connect direct-charge flows. Equivalent operations now live on PaymentIntents.

### Refunds — `/v1/refunds`
Endpoints: `create`, `retrieve`, `update`, `list`, `cancel`.
```js
await stripe.refunds.create({
  payment_intent: 'pi_…',
  amount: 1000,            // partial refund (optional)
  reason: 'requested_by_customer',
});
```
- `reason`: `duplicate` | `fraudulent` | `requested_by_customer`
- Connect: `refund_application_fee`, `reverse_transfer`
- Events: `refund.created`, `.updated`, `.failed`, `charge.refund.updated`.

### Disputes — `/v1/disputes`
Endpoints: `retrieve`, `update` (submit evidence), `list`, `close`.
- Update evidence by setting fields on `evidence{}`. To submit, POST with `submit=true` (or wait for Stripe's auto-submit at the deadline).
- Statuses you'll see: `needs_response`, `under_review`, `won`, `lost`, `warning_closed`.
- Events: `charge.dispute.created`, `.updated`, `.closed`, `.funds_withdrawn`, `.funds_reinstated`.

### Products & Prices
- **Products** (`/v1/products`): `name`, `description`, `active`, `images[]`, `metadata`, `default_price`, `tax_code`, `unit_label`, `shippable`.
- **Prices** (`/v1/prices`): `product`, `unit_amount`, `currency`, `recurring.{interval, interval_count, usage_type, aggregate_usage}`, `billing_scheme` (`per_unit` | `tiered`), `tiers[]`, `tiers_mode` (`graduated` | `volume`), `lookup_key`, `nickname`, `active`.
- One Product → many Prices (e.g. `$10/mo`, `$100/yr`, `€9 once`).
- Prices have no `delete` — deactivate via `active=false`.

### Subscriptions — `/v1/subscriptions`
Endpoints: `create`, `update`, `retrieve`, `list`, `cancel` (DELETE), `resume`, `migrate`, `search`.

Statuses: `incomplete`, `incomplete_expired`, `trialing`, `active`, `past_due`, `canceled`, `unpaid`, `paused`.

```js
const sub = await stripe.subscriptions.create({
  customer: 'cus_…',
  items: [{ price: 'price_…' }],
  payment_behavior: 'default_incomplete',
  payment_settings: { save_default_payment_method: 'on_subscription' },
  expand: ['latest_invoice.payment_intent'],
});
// sub.latest_invoice.payment_intent.client_secret → confirm on client
```

Key fields:
- `items[]`, `default_payment_method`, `collection_method` (`charge_automatically` | `send_invoice`)
- `billing_cycle_anchor`, `proration_behavior` (`create_prorations` | `none` | `always_invoice`)
- `payment_behavior: 'default_incomplete'` — **use this for SCA** so the first PaymentIntent can be confirmed client-side.
- `trial_end`, `trial_period_days`, `cancel_at_period_end`, `pause_collection`

### Invoices — `/v1/invoices` and InvoiceItems `/v1/invoiceitems`
Endpoints: `create`, `finalize`, `pay`, `void`, `mark_uncollectible`, `send`, `list`, plus `GET /v1/invoices/upcoming` (preview the next subscription invoice).

Lifecycle: `draft` → `finalize` → automatic collection (`charge_automatically`) or email link (`send_invoice`) → `paid` / `void` / `uncollectible`.

- `auto_advance: true` lets Stripe auto-finalize & collect.
- After `invoice.finalized`, automatic collection kicks in **~1 hour later** to allow last-minute updates.
- Hosted assets: `hosted_invoice_url`, `invoice_pdf`.

### Coupons / Promotion Codes / Discounts
- **Coupons** (`/v1/coupons`): `percent_off` XOR `amount_off`+`currency`, `duration` (`once` | `repeating` | `forever`), `duration_in_months`, `max_redemptions`, `redeem_by`, `applies_to.products`.
- **Promotion codes** (`/v1/promotion_codes`): customer-facing codes wrapping a coupon, with their own redemption limits and active/inactive state.
- **Discounts** apply to Customers / Subscriptions / Invoices — **not** to one-off PaymentIntents.

### Tax
Enable per-product `tax_code`, per-customer `tax_exempt` and `tax_ids`.
- `automatic_tax: { enabled: true }` on Checkout Sessions, Subscriptions, Invoices.
- **Tax API** for off-Stripe flows: `/v1/tax/calculations` (preview) → `/v1/tax/transactions/create_from_calculation` (record).
- `/v1/tax/registrations` declares where you're registered — Stripe only computes tax in those jurisdictions (or use Tax Monitoring for threshold tracking).
- Customer tax IDs: `/v1/customers/:id/tax_ids` (`eu_vat`, `us_ein`, `gb_vat`, etc.).

### Checkout Sessions — `/v1/checkout/sessions`
Stripe-hosted (or embedded) checkout page.

```bash
curl https://api.stripe.com/v1/checkout/sessions -u sk_test_…: \
  -d mode=payment \
  -d success_url=https://example.com/success \
  -d cancel_url=https://example.com/cancel \
  -d "line_items[0][price]=price_…" \
  -d "line_items[0][quantity]=1"
```

Key fields:
- `mode`: `payment` | `subscription` | `setup`
- `ui_mode`: `hosted` (default redirect) | `embedded` | `custom`
- `line_items[]`, `customer` / `customer_email`, `payment_method_types[]`
- `automatic_tax`, `allow_promotion_codes`, `billing_address_collection`, `shipping_address_collection`
- `client_reference_id` (your ID), `metadata`
- `payment_intent_data`, `subscription_data` for nested config
- Event to listen for: `checkout.session.completed`

### Payment Links — `/v1/payment_links`
No-code reusable URL for a checkout. Endpoints: `create`, `retrieve`, `update` (deactivate via `active=false`), `list`, `GET /:id/line_items`.

### Customer (Billing) Portal — `/v1/billing_portal/sessions`
```js
const session = await stripe.billingPortal.sessions.create({
  customer: 'cus_…',
  return_url: 'https://example.com/account',
});
res.redirect(session.url);
```
Stripe-hosted UI for subscription / payment method / invoice management. Configure features in Dashboard → Billing → Customer portal (or via `/v1/billing_portal/configurations`).

---

## 7. Stripe Connect

For platforms / marketplaces that move money between sellers, providers, or sub-merchants.

### Account types (legacy framing — still supported)
| Type | Onboarding | Dashboard | Liability / fees | Use when |
|--|--|--|--|--|
| **Standard** | Stripe-hosted, full KYC | Connected sees full Stripe Dashboard | Connected acct owns fraud/disputes | Lightest integration; partners already use Stripe |
| **Express** | Stripe-hosted | Express Dashboard (lightweight) | Platform shares more responsibility | Most marketplaces |
| **Custom** | You build everything | No Dashboard for connected | Platform owns all compliance UI | Fully embedded, white-label experience |

**Accounts v2** is the new recommended path: compose capabilities (`losses`, `requirement_collection`, `stripe_dashboard.type`, `fees.payer`) on a single account type instead of choosing one of the three above.

### OAuth (Standard accounts)
```
GET https://connect.stripe.com/oauth/authorize
    ?response_type=code
    &client_id=ca_…
    &scope=read_write
    &state=<csrf>
    &redirect_uri=https://your.app/oauth/return
```
Exchange:
```bash
curl https://connect.stripe.com/oauth/token \
  -d client_secret=sk_test_… \
  -d code=ac_… \
  -d grant_type=authorization_code
# → { stripe_user_id, scope, livemode, token_type, access_token, refresh_token }
```

### Account Links (Express / Custom onboarding)
```js
const link = await stripe.accountLinks.create({
  account: 'acct_…',
  refresh_url: 'https://your.app/reauth',
  return_url: 'https://your.app/return',
  type: 'account_onboarding',           // or 'account_update'
});
res.redirect(link.url);
```
Links are single-use and short-lived (~5 min).

### Charge patterns
| Pattern | Created on | Funds flow | Refund debits |
|--|--|--|--|
| **Direct** | Connected account (via `Stripe-Account` header) | Connected → platform takes `application_fee_amount` | Connected balance |
| **Destination** | Platform, with `transfer_data[destination]=acct_…` | Platform → auto-transfer to connected | Platform balance |
| **Separate charges & transfers** | Platform; then explicit `POST /v1/transfers` | Platform → manual transfer(s) to one or more accounts | Platform balance |

Common params:
- `application_fee_amount` — platform's cut (in smallest unit)
- `on_behalf_of` — settles in connected acct's country, uses its descriptor (destination charges only)
- `transfer_data.destination`, `transfer_data.amount`

### Other Connect resources
- `/v1/accounts`, `/v1/accounts/:id/persons`, `/v1/accounts/:id/external_accounts` (bank accounts / debit cards)
- `/v1/transfers`, `/v1/transfers/:id/reversals`
- `/v1/payouts` (per connected account when called with `Stripe-Account`)
- `/v1/application_fees`, `/v1/application_fees/:id/refunds`
- `/v1/topups` (US only, fund platform balance)

---

## 8. Webhooks

### Setup
- Register endpoints in Dashboard or via `/v1/webhook_endpoints` (max 16 per account/sandbox).
- Each endpoint has a signing secret `whsec_…` shown once on creation.

### Signature verification
Header: `Stripe-Signature: t=<unix>,v1=<hex>,v0=…` — only `v1` matters in prod.
- HMAC-SHA256 over `${t}.${rawBody}` with the signing secret.
- Default tolerance: **5 minutes**.
- Constant-time compare against `v1`.
- **Use the raw request body** — any JSON re-serialization (Express's default `express.json()`, etc.) breaks signatures. In Express:
  ```js
  app.post('/webhook',
    express.raw({ type: 'application/json' }),
    (req, res) => {
      const event = stripe.webhooks.constructEvent(
        req.body, req.headers['stripe-signature'], process.env.STRIPE_WH_SECRET
      );
      // ... handle event
      res.json({ received: true });
    });
  ```

### Retries & ack
- Return `2xx` ASAP; do real work async.
- Non-`2xx` → Stripe retries with exponential backoff for **up to 3 days** (live) / fewer in sandbox.
- Manual resend: Dashboard (15 days) or CLI (`stripe events resend evt_…`, 30 days).
- Events may be delivered **more than once** and **out of order** — handlers must be idempotent. Track processed `event.id` in your DB.

### Common event types
- `payment_intent.succeeded`, `payment_intent.payment_failed`, `payment_intent.requires_action`
- `charge.refunded`, `charge.dispute.created`, `charge.dispute.closed`
- `checkout.session.completed`, `checkout.session.async_payment_succeeded`
- `invoice.paid`, `invoice.payment_failed`, `invoice.finalized`
- `customer.subscription.created`, `.updated`, `.deleted`, `.trial_will_end`
- `customer.created`, `customer.updated`
- `payout.paid`, `payout.failed`
- `account.updated` (Connect — capability/requirement changes)

### Events V2 / "thin events"
Newer event model with minimal payloads (just IDs); subscriber fetches full object via the API. Used by v2 namespace resources (Accounts v2 etc.). Configured via Event Destinations (`/v2/core/event_destinations`) which can also route to Amazon EventBridge.

### Restricted webhook endpoints
Scope an endpoint to receive only specific event types; pair with restricted API keys for least-privilege.

---

## 9. Stripe.js, Elements & Payment Element

```html
<script src="https://js.stripe.com/v3/"></script>
```
or
```js
import { loadStripe } from '@stripe/stripe-js';
const stripe = await loadStripe('pk_test_…');
```

### Pattern
1. Server creates a `PaymentIntent` / `SetupIntent` / `CheckoutSession` and returns its `client_secret`.
2. Client mounts the Payment Element:
   ```js
   const elements = stripe.elements({ clientSecret, appearance });
   const paymentElement = elements.create('payment');
   paymentElement.mount('#payment-element');
   ```
3. On submit:
   ```js
   const { error } = await stripe.confirmPayment({
     elements,
     confirmParams: { return_url: 'https://your.app/return' },
   });
   ```
4. After redirect, optionally `stripe.retrievePaymentIntent(clientSecret)` to surface status.

### Embedded Checkout (recommended for most integrations)
Use the Checkout Sessions API with `ui_mode: 'embedded'`, then mount the Checkout Element. Stripe maintains the entire payment UI for you.

### Notes
- `client_secret` is per-intent — generate a fresh one per attempt. Don't cache server-side.
- `automatic_payment_methods: { enabled: true }` defers method selection to Dashboard config; explicit `payment_method_types[]` pins exactly what's offered.

---

## 10. Stripe Terminal (in-person)

Hardware: BBPOS WisePOS E, Stripe Reader S700/M2, Tap-to-Pay on iPhone/Android.

Flow:
1. Server: `POST /v1/terminal/connection_tokens` → short-lived `secret`.
2. Client SDK (JS / iOS / Android / RN) uses it to discover/connect readers.
3. Register hardware: `POST /v1/terminal/readers`. Assign to a `Location` (`POST /v1/terminal/locations`) — locations drive tax + reader assignment.
4. Server creates a PaymentIntent (`payment_method_types: ['card_present']`); client SDK calls `collectPaymentMethod` → `processPayment`; server `capture`.

Supports offline-mode payments and contactless.

---

## 11. Stripe Tax, Radar, Identity

### Stripe Tax
- Auto-compute sales tax, VAT, GST.
- `automatic_tax.enabled: true` on Checkout / Subscriptions / Invoices.
- Tax API: `/v1/tax/calculations`, `/v1/tax/transactions`, `/v1/tax/registrations`, `/v1/tax_rates`, `/v1/tax_codes`.

### Radar
- Real-time fraud scoring + rules on every PaymentIntent/Charge.
- Each charge: `outcome.risk_level` (`normal`/`elevated`/`highest`) and (with Radar for Fraud Teams) `outcome.risk_score`.
- Reviews land in `/v1/reviews`.
- Rules language: `:risk_score: > 70`, `:card_funding: = 'prepaid'`, allowlist/blocklist by email/IP/BIN.
- For non-Stripe payment forms, collect a **Radar Session** ID client-side and pass on the PaymentIntent.

### Identity
- `POST /v1/identity/verification_sessions` → `client_secret` for redirect or embedded modal.
- Types: `document` (gov ID + optional selfie), `id_number` (SSN match).
- On completion → `VerificationReport`.
- Events: `identity.verification_session.verified`, `.requires_input`, `.canceled`.

---

## 12. Issuing, Treasury, Capital, Atlas, Climate

| Product | What it is |
|--|--|
| **Issuing** | Issue virtual + physical cards. `cardholders`, `cards`, `authorizations` (realtime webhook decisions), `transactions`, `disputes`. Spending controls on amount, MCC, geography. Apple/Google Pay provisioning. |
| **Treasury** | Embedded banking (US/GB). `financial_accounts`, `outbound_payments`, `outbound_transfers`, `received_credits`, `received_debits`. USD/GBP/EUR/USDC. |
| **Capital** | Revenue-based financing for connected accounts. Flat fee, no interest. Auto-repaid from sales. US/UK/FR/DE/AU. |
| **Atlas** | US company incorporation (Delaware LLC/C-Corp), EIN, 83(b), founder vesting, $50k+ partner perks. |
| **Climate** | `/v1/climate/orders`, `/v1/climate/products`, `/v1/climate/suppliers`. Commitments auto-direct a % of revenue to carbon removal via Frontier. |

---

## 13. Financial Connections, Crypto Onramp

### Financial Connections
- Securely link end-user bank accounts.
- `/v1/financial_connections/sessions` (create with `account_holder`, `permissions[]`: `balances`/`ownership`/`transactions`/`payment_method`, `filters`).
- `/v1/financial_connections/accounts` (list, refresh, subscribe), `/v1/financial_connections/transactions`.
- Use cases: instant ACH verification (skip microdeposits), balance check before debit, ownership verification.

### Crypto Onramp
- Stripe is merchant of record; handles KYC, fraud, compliance for fiat → crypto.
- `OnrampSession` server-side → `client_secret` to JS SDK. Configure `destination_currency`, `destination_network`, `destination_amount`, `wallet_addresses`.
- Requires application/approval in Dashboard.

---

## 14. Files & File Links

- Upload host: `https://files.stripe.com/v1/files` (not api.stripe.com!), `multipart/form-data`.
- `purpose` values: `dispute_evidence`, `identity_document`, `business_logo`, `business_icon`, `account_requirement`, `customer_signature`, `pci_document`, `tax_document_user_upload`, `terminal_reader_splashscreen`, `additional_verification`, `finance_report_run`, `sigma_scheduled_query`.
- Constraints: `identity_document` max 8000×8000 px; Office docs with VBA macros rejected; MIME must match.
- `/v1/file_links` creates short-lived public URLs to a stored file.

```bash
curl https://files.stripe.com/v1/files -u sk_test_…: \
  -F purpose=dispute_evidence -F file="@./receipt.pdf"
```

---

## 15. Reporting, Sigma, Balance Transactions

- **`/v1/balance_transactions`** — source of truth for reconciliation. Every money movement (charge, refund, payout, adjustment, transfer, fee) is a balance transaction with `amount`, `fee`, `fee_details[]`, `net`, `available_on`, `currency`, `reporting_category`.
- **`/v1/payouts`** — funds moved from Stripe balance → your bank.
- **Reports API:**
  - `GET /v1/reporting/report_types` (e.g. `balance.summary.1`, `payout_reconciliation.itemized.5`)
  - `POST /v1/reporting/report_runs` with `parameters: { interval_start, interval_end, columns, timezone }` → result is a downloadable `file`.
- **Sigma** — schedule SQL queries against Stripe's data warehouse; results delivered as files.

---

## 16. Usage-based Billing (Meters)

Native primitives — newer accounts may be steered toward Metronome as Stripe's recommended platform, but Meters still work:

```js
// Define a meter
await stripe.billing.meters.create({
  display_name: 'API Requests',
  event_name: 'api_request',
  default_aggregation: { formula: 'sum' },
  customer_mapping: { event_payload_key: 'stripe_customer_id', type: 'by_id' },
  value_settings: { event_payload_key: 'value' },
});

// Record events
await stripe.billing.meterEvents.create({
  event_name: 'api_request',
  payload: { stripe_customer_id: 'cus_…', value: '5' },
  identifier: 'req_unique_id',  // idempotency
});
```

- `POST /v1/billing/meter_event_adjustments` for corrections.
- `GET /v1/billing/meters/:id/event_summaries` returns aggregated windows.
- Bind a meter to a recurring Price via `recurring.meter` to convert events into invoice lines.
- Events process asynchronously — summaries lag.

---

## 17. SDKs

| Language | Install | Init |
|--|--|--|
| Node | `npm install stripe` | `const stripe = require('stripe')('sk_test_…');` |
| Python | `pip install stripe` | `import stripe; stripe.api_key = 'sk_test_…'` |
| Ruby | `gem install stripe` | `Stripe.api_key = 'sk_test_…'` |
| PHP | `composer require stripe/stripe-php` | `\Stripe\Stripe::setApiKey('sk_test_…');` |
| Go | `go get github.com/stripe/stripe-go/v79` | `stripe.Key = "sk_test_…"` |
| Java | Maven `com.stripe:stripe-java` | `Stripe.apiKey = "sk_test_…";` |
| .NET | `dotnet add package Stripe.net` | `StripeConfiguration.ApiKey = "sk_test_…";` |

All ship: typed models, automatic retries on network errors (configurable `maxNetworkRetries`), per-request idempotency-key support, auto-pagination, per-request `stripeAccount` for Connect.

---

## 18. Stripe CLI

```bash
brew install stripe/stripe-cli/stripe        # macOS
# scoop / choco on Windows, apt repo on Linux

stripe login                                  # pair with account
stripe listen --forward-to localhost:3000/webhook
  # prints the whsec_… to use locally
stripe listen --events payment_intent.succeeded,charge.refunded
stripe trigger payment_intent.succeeded       # synthetic event
stripe logs tail                              # live API request logs
stripe customers list --limit 3
stripe events resend evt_…
stripe fixtures fixture.json                  # replay scenarios
stripe samples create accept-a-payment        # scaffold sample app
```

---

## 19. Rate limits

- Live mode: ~**100 req/s** account-wide. Sandbox: ~**25 req/s**.
- Per-endpoint cap ~25 req/s. Files API ~20 req/s. Search ~20 read req/s.
- Read/transaction ratio cap ~500 reads per processed transaction over 30 days (with a 10k reads/mo floor).
- 429 responses include `Stripe-Rate-Limited-Reason` (`global-rate`, `endpoint-rate`, `global-concurrency`, `endpoint-concurrency`, `resource-rate`).
- Strategy: client-side token bucket + exponential backoff with jitter + `Idempotency-Key` so safe to retry. SDKs auto-retry transient network errors but **not** 429 by default — configure `maxNetworkRetries`.

---

## 20. Stripe Apps & Workbench

- **Workbench** — in-Dashboard developer environment. API version management, event inspector, request logs, webhook tester.
- **Stripe Apps** — UI extensions that run inside the Dashboard. Built on the App SDK + `stripe-app.json` manifest declaring views, permissions (`secret_key_read`, `customer_read`, etc.), distribution. CLI: `stripe apps create | start | upload`. Distribute privately or to the public Stripe Apps Marketplace.

---

## 21. Cross-cutting gotchas

1. **Use the raw request body for webhook signature verification.** Default JSON middleware breaks signatures.
2. **`client_secret` is per-intent.** Generate fresh per checkout attempt; never cache server-side.
3. **For Subscriptions with SCA**, set `payment_behavior: 'default_incomplete'` and expand `latest_invoice.payment_intent` so the client can confirm on first invoice.
4. **`Idempotency-Key` with different params → `idempotency_error`.** Always fresh UUID per logical op.
5. **`amount` is in the smallest currency unit** — and JPY/KRW are zero-decimal, BHD/KWD are three-decimal.
6. **Connect refunds debit different balances** depending on charge pattern (direct → connected, destination → platform). Design refund UX accordingly.
7. **Webhook ordering is not guaranteed.** Track `event.id` and write idempotent handlers.
8. **`automatic_payment_methods` vs explicit `payment_method_types`** — the former defers to Dashboard config; the latter pins exactly what's offered.
9. **API versions:** pin via Workbench *and* in code. Old SDKs ignore your account default and use their own pinned version.
10. **File uploads** go to `files.stripe.com`, not `api.stripe.com`.
11. **Trust webhooks, not the client.** The `return_url` after a redirect can be hit before the payment actually succeeds — fulfill on `payment_intent.succeeded` / `checkout.session.completed`.
12. **Search is eventually consistent** (~1 min lag). For just-created records, use `retrieve` or `list` with filters.
13. **Key rotation has a 7-day overlap window** — use it; don't redeploy and rotate simultaneously.
14. **Sandboxes ≠ legacy test mode** — webhook endpoints, Connect setup, and keys are all per-sandbox; recreate when spinning up a new one.

---

## 22. Quick reference

### Useful endpoints (cheat sheet)
```
POST   /v1/customers
POST   /v1/payment_intents
POST   /v1/payment_intents/:id/confirm
POST   /v1/payment_intents/:id/capture
POST   /v1/setup_intents
POST   /v1/payment_methods/:id/attach
POST   /v1/refunds
POST   /v1/checkout/sessions
POST   /v1/billing_portal/sessions
POST   /v1/subscriptions
POST   /v1/invoices/:id/finalize
POST   /v1/invoices/:id/pay
POST   /v1/account_links
POST   /v1/transfers
POST   /v1/payouts
POST   /v1/webhook_endpoints
POST   /v1/files                       (host: files.stripe.com)
GET    /v1/balance_transactions
GET    /v1/events
```

### Headers
```
Authorization: Bearer sk_…           # or HTTP Basic with empty password
Stripe-Version: 2026-06-24.dahlia    # override account default
Stripe-Account: acct_…               # act as connected account
Idempotency-Key: <uuid>              # on POST
```

### Dashboard / dev links
- Dashboard: <https://dashboard.stripe.com>
- API reference: <https://docs.stripe.com/api>
- Events list: <https://docs.stripe.com/api/events/types>
- Error codes: <https://docs.stripe.com/error-codes>
- Decline codes: <https://docs.stripe.com/declines/codes>
- Webhooks: <https://docs.stripe.com/webhooks>
- Stripe CLI: <https://docs.stripe.com/stripe-cli>
- Libraries: <https://docs.stripe.com/libraries>
