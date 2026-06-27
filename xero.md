# Xero API Reference

A practical reference for using the Xero APIs — what they cover, how OAuth works, how to call endpoints, and the gotchas that show up in production.

---

## 1. API Overview

Xero offers a suite of REST APIs covering the major areas of its accounting platform. Each API is independently scoped and most share the same OAuth 2.0 authentication layer.

- **Accounting API** — Core API. Invoices, contacts, bank transactions, accounts, payments, reports, etc. Base: `https://api.xero.com/api.xro/2.0/`
- **Payroll APIs** — Separate APIs for AU, UK, and NZ (each has its own scopes and endpoints, e.g. `https://api.xero.com/payroll.xro/1.0/` for AU, `payroll.xro/2.0/` for UK/NZ).
- **Files API** — Upload/attach files to Xero records and manage folders.
- **Assets API** — Fixed asset register management.
- **Projects API** — Project tracking, time entries, expenses, project tasks.
- **Bank Feeds API** — Partner-only API to push bank statement lines into Xero (requires special partnership approval).
- **Finance API** — Loan/lending-oriented data (cash validation, financial statements, balance sheet, P&L, trial balance) optimised for lender consumption.
- **App Store / Subscriptions API** — For apps listed on Xero App Store to handle subscriptions/billing.
- **Practice Manager API (XPM)** — Xero Practice Manager (separate product, separate base URL).
- **eInvoicing API** — PEPPOL-style e-invoicing.
- **Xero HQ API** — For accountants/bookkeepers managing client portfolios.

---

## 2. Authentication (OAuth 2.0)

Xero uses OAuth 2.0 exclusively (OAuth 1.0a is retired). Apps are registered at <https://developer.xero.com/myapps>.

### App / grant types
- **Web app (Authorization Code grant)** — Server-side apps. Has Client ID + Client Secret. Standard authorization-code flow.
- **Mobile / single-page / native (Authorization Code + PKCE)** — No client secret; uses PKCE code verifier (43–128 chars, `A-Z a-z 0-9 -._~`) and code challenge (S256).
- **Custom Connections (Client Credentials / M2M)** — Single-organisation, paid subscription required. Grant type `client_credentials`. Only available in AU/NZ/UK/US.

### Endpoints
- Authorize URL: `https://login.xero.com/identity/connect/authorize`
- Token URL: `https://identity.xero.com/connect/token`
- Connections (tenant list) URL: `https://api.xero.com/connections`
- Revocation URL: `https://identity.xero.com/connect/revocation`

### Authorization request (web app)
```
GET https://login.xero.com/identity/connect/authorize?
    response_type=code
   &client_id=YOUR_CLIENT_ID
   &redirect_uri=YOUR_REDIRECT_URI
   &scope=openid profile email accounting.transactions offline_access
   &state=RANDOM_STATE
```

### Token exchange (web app)
```
POST https://identity.xero.com/connect/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTH_CODE
&redirect_uri=YOUR_REDIRECT_URI
```

Response contains `access_token`, `refresh_token`, `expires_in`, `id_token` (if `openid` requested), and `token_type=Bearer`.

### Token lifetimes
- **Access token**: 30 minutes
- **Refresh token**: 60 days (rotates with each use; a new refresh token is returned on each refresh). There is a 30-minute grace period to retry refresh with the previous refresh token if the response is lost.
- **ID token (OIDC)**: short-lived JWT identifying the user.

### Tenant connections
After authorization the access token is **not** bound to a single tenant. You must call:
```
GET https://api.xero.com/connections
Authorization: Bearer ACCESS_TOKEN
```
Returns an array of `{ id, authEventId, tenantId, tenantType, tenantName, createdDateUtc, updatedDateUtc }`. Use `tenantId` as the `Xero-tenant-id` header on subsequent API calls. The `id` field is the connection ID — not the tenant ID. Filter with `?authEventId=...` for the most recent consent event.

### Common scopes
- **OpenID / user**: `openid`, `profile`, `email`
- **Refresh tokens**: `offline_access`
- **Accounting**: `accounting.transactions`, `accounting.transactions.read`, `accounting.contacts`, `accounting.contacts.read`, `accounting.settings`, `accounting.settings.read`, `accounting.reports.read`, `accounting.reports.tenninetynine`, `accounting.journals.read`, `accounting.attachments`, `accounting.attachments.read`, `accounting.budgets.read`, `accounting.budgets`
- **Files**: `files`, `files.read`
- **Assets**: `assets`, `assets.read`
- **Projects**: `projects`, `projects.read`
- **Bank Feeds**: `bankfeeds`
- **Finance**: `finance.statements.read`, `finance.cashvalidation.read`, `finance.accountingactivity.read`
- **Payroll (per region)**: `payroll.employees`, `payroll.payruns`, `payroll.payslip`, `payroll.timesheets`, `payroll.settings` (plus `.read` variants)
- **Non-tenanted**: `app.connections` (for the M2M client-credentials access token used to query `/connections`)

---

## 3. Rate Limits

Per tenant:
- **Concurrent**: 5 in-flight calls at a time
- **Per minute**: 60 calls/minute
- **Per day**: 5,000 calls/day

App-wide (across all tenants you connect to):
- **App minute limit**: 10,000 calls/minute

Starter-tier apps are additionally capped at **1,000 calls/day total** — to lift this, move to the Core tier or higher.

Uncertified apps are limited to **25 connected tenants** simultaneously. Certified/App Partner apps can connect unlimited tenants after Xero review.

### 429 handling
When you exceed any limit you receive HTTP 429 plus:
- `X-Rate-Limit-Problem`: which limit you hit (`minute`, `day`, `concurrent`, or `app-minute`)
- `Retry-After`: seconds to wait (set for minute and daily limits)

Every response also includes:
- `X-DayLimit-Remaining`
- `X-MinLimit-Remaining`
- `X-AppMinLimit-Remaining`

Use exponential backoff and a bounded thread-pool (≤5) to stay under the concurrent limit.

---

## 4. Accounting API Endpoints

All paths relative to `https://api.xero.com/api.xro/2.0/`. Main resources:

- **Invoices** — Sales (ACCREC) and bills (ACCPAY). Status flow DRAFT → SUBMITTED → AUTHORISED → PAID. Supports attachments, online invoice URL, email.
- **Contacts** — Customers and suppliers. Includes addresses, phones, payment terms, default tax types, contact groups.
- **ContactGroups** — Grouping of contacts for batch operations.
- **Accounts** — Chart of accounts.
- **BankTransactions** — Spend/receive money (cash coding), distinct from invoices.
- **BankTransfers** — Money transfers between bank accounts.
- **BatchPayments** — Combine multiple payments into one bank transaction.
- **BrandingThemes** — Invoice/quote branding templates.
- **BudgetSummary / Budgets** — Read-only budget data.
- **CreditNotes** — Sales and bill credit notes.
- **Currencies** — Active currencies on the org.
- **Employees** — Lightweight employee records (separate from Payroll API).
- **InvoiceReminders** — Reminder schedules.
- **Items** — Inventory/non-inventory items.
- **Journals** — Read-only journals view (use as a sync feed via `?offset=` or modified-since).
- **LinkedTransactions** — Billable expense links.
- **ManualJournals** — Manually entered journals.
- **Organisation** — Org details, base currency, tax basis, country.
- **Overpayments / Prepayments** — Customer/supplier overpayments and prepayments.
- **Payments** — Allocations of money to invoices, credit notes, overpayments, prepayments.
- **PaymentServices** — Online payment provider config.
- **PurchaseOrders** — Purchase orders to suppliers.
- **Quotes** — Sales quotes.
- **RepeatingInvoices** — Recurring invoice templates.
- **Reports** — Pre-built reports: `BalanceSheet`, `ProfitAndLoss`, `TrialBalance`, `BankSummary`, `BudgetSummary`, `ExecutiveSummary`, `AgedReceivablesByContact`, `AgedPayablesByContact`, `BankStatement`, `TenNinetyNine`.
- **TaxRates** — Tax rates and rate components.
- **TrackingCategories** — Up to two categories with options for tracking dimensions.
- **Users** — Org users (read-only).
- **Setup** — Conversion balances and accounts (one-shot per org).
- **Attachments** — Attach binary files to most transaction types: `GET/POST/PUT /{Entity}/{Guid}/Attachments[/{FileName}]`.
- **History & Notes** — `GET/PUT /{Entity}/{Guid}/History`.

---

## 5. Request Format

### Base URL & headers
```
Authorization: Bearer {access_token}
Xero-tenant-id: {tenantId from /connections}
Accept: application/json
Content-Type: application/json   (for POST/PUT)
```

`Xero-tenant-id` is required on every Accounting/Payroll/Files/etc. call. The `/connections` endpoint itself does not need it.

### JSON vs XML
Both JSON and XML are supported on most Accounting API endpoints. Set `Accept: application/json` for JSON responses (recommended). The legacy default for some PUT/POST is XML, so always pin `Accept` and `Content-Type`.

### Filtering, ordering, pagination
- **`where`** — Server-side filter expression. Field names are PascalCase. Example: `?where=Status=="AUTHORISED" AND Type=="ACCREC" AND Date>=DateTime(2026,01,01)`. String literals are double-quoted; booleans `true`/`false`; GUIDs `Guid("...")`; dates `DateTime(yyyy,mm,dd)`.
- **`order`** — e.g. `?order=UpdatedDateUTC DESC`
- **`page`** — 1-based. Most list endpoints page at 100 records; some (Invoices, Contacts, BankTransactions) support `pageSize` up to 1000 with `pageSize` query param on supporting endpoints.
- **`includeArchived=true`** — Include archived contacts/items.
- **`summaryOnly=true`** — Lighter payload on Invoices/Contacts.
- **`searchTerm=...`** — Full-text search on supporting endpoints (Contacts, Invoices).

### Conditional / sync
- **`If-Modified-Since: Wed, 01 Jan 2026 00:00:00 GMT`** — Returns only records modified after that timestamp. Primary delta-sync mechanism (cheaper than `where=UpdatedDateUTC>=...`).

### Other useful query params
- **`SummarizeErrors=false`** — On bulk POST/PUT, returns each rejected item with its `ValidationErrors` instead of failing the whole batch. Default `true`.
- **`unitdp=4`** — Use 4 decimal places for unit amounts (default is 2). Useful for inventory/items needing finer pricing.

---

## 6. Common Operations (HTTP verbs)

Xero verbs have a specific meaning:
- **GET** — Read. Single record by GUID (`/Invoices/{InvoiceID}`) or list (`/Invoices`).
- **PUT** — Create only. Fails if the record already exists.
- **POST** — Create or update. If the body contains an ID matching an existing record, it's an update; otherwise it's a create.
- **DELETE** — Used sparingly; many entities are voided via POST with `Status=DELETED` or `VOIDED` rather than DELETE.

### Bulk operations
Most create/update endpoints accept arrays under the root key, e.g.:
```json
POST /api.xro/2.0/Invoices
{ "Invoices": [ { ... }, { ... }, { ... } ] }
```
Combine with `SummarizeErrors=false` for per-record validation feedback.

### Example — create invoice
```
POST https://api.xero.com/api.xro/2.0/Invoices
Authorization: Bearer ...
Xero-tenant-id: ...
Accept: application/json
Content-Type: application/json
Idempotency-Key: 5d1e1c10-1f4b-4f02-9f8c-2c5a4f3f2a1b

{
  "Type": "ACCREC",
  "Contact": { "ContactID": "..." },
  "Date": "2026-06-27",
  "DueDate": "2026-07-27",
  "LineItems": [
    { "Description": "Consulting", "Quantity": 1, "UnitAmount": 500, "AccountCode": "200" }
  ],
  "Status": "AUTHORISED"
}
```

---

## 7. Webhooks

Subscribe via the developer portal. Currently supported event categories:
- **INVOICE** (CREATE, UPDATE)
- **CONTACT** (CREATE, UPDATE)
- **CREDIT NOTE** (CREATE, UPDATE)

### Intent-to-Receive
When you save the webhook URL, Xero immediately POSTs a test payload. Your endpoint must:
1. Verify the `x-xero-signature` header.
2. Return **HTTP 200** with valid signature → marked active.
3. Return **HTTP 401** with invalid signature → Xero retries; if too many fail Xero disables the subscription.

### Signature verification
```
HMAC-SHA256(webhookSigningKey, rawRequestBody)
```
Compare base64-encoded result against `x-xero-signature` using a constant-time compare. The signing key is shown once in the developer portal. **Verify against the raw bytes**, not the re-serialised JSON.

### Payload shape
```json
{
  "events": [
    {
      "resourceUrl":    "https://api.xero.com/api.xro/2.0/Invoices/{InvoiceID}",
      "resourceId":     "<guid>",
      "eventDateUtc":   "2026-06-27T10:15:00.000",
      "eventType":      "UPDATE",
      "eventCategory":  "INVOICE",
      "tenantId":       "<tenant-guid>",
      "tenantType":     "ORGANISATION"
    }
  ],
  "lastEventSequence":  1,
  "firstEventSequence": 1,
  "entropy": "..."
}
```
Webhooks are batched per app — a single payload can contain events from multiple tenants, categories, and types. Always look up tenantId per event.

Endpoint requirements: respond within 5 seconds with 2xx; otherwise Xero retries with exponential backoff. Queue work asynchronously after acking.

---

## 8. Official SDKs

Maintained by Xero on GitHub under <https://github.com/XeroAPI>:
- **xero-node** (Node.js / TypeScript) — `npm i xero-node`
- **Xero-NetStandard** (.NET / C#) — `Xero.NetStandard.OAuth2` NuGet
- **xero-python** (Python) — `pip install xero-python`
- **xero-php-oauth2** (PHP) — via Composer
- **xero-java** (Java) — Maven/Gradle
- **xero-ruby** (Ruby) — gem `xero-ruby`

All SDKs are generated from the official OpenAPI specs (<https://github.com/XeroAPI/Xero-OpenAPI>) and include OAuth helpers, token refresh, and the `Idempotency-Key` parameter.

---

## 9. Error Handling

| Status | Meaning | What to do |
|---|---|---|
| 200 | OK | — |
| 400 | Bad request / validation | Read `ValidationErrors` and `Elements[].ValidationErrors`; do not retry. |
| 401 | Unauthorized | Token expired/invalid or user disconnected. Refresh token; on persistent 401, prompt re-auth. |
| 403 | Forbidden | Missing scope or tenant disconnected. |
| 404 | Not found | Resource GUID wrong or wrong tenant. |
| 429 | Rate limit | Respect `Retry-After`, back off. |
| 500 | Server error | Retry with exponential backoff (idempotency key recommended). |
| 503 | Service unavailable | Standard outage signal; retry with backoff. |

### Validation error shape
```json
{
  "ErrorNumber": 10,
  "Type": "ValidationException",
  "Message": "A validation exception occurred",
  "Elements": [
    {
      "ValidationErrors": [
        { "Message": "Email address must be valid." }
      ],
      "InvoiceNumber": "INV-001"
    }
  ]
}
```
With `SummarizeErrors=false`, successful records are still saved and failures are itemised in `Elements`.

---

## 10. Demo Company (Sandbox)

There is no dedicated sandbox environment. Instead each Xero account ships with a **Demo Company** pre-populated with sample data. Sign in at <https://my.xero.com>, choose "Try the Demo Company", and connect your dev app to it like any other tenant. You can pick the region (AU/NZ/UK/US/Global) and reset it at any time. The Demo Company auto-resets after 28 days, after which you must reconnect.

The **API Explorer** in the developer portal lets you make live calls against the demo company without writing code — the fastest way to learn endpoint shapes.

---

## 11. Filtering and Querying — Cheat Sheet

```
where=Type=="ACCREC"
where=Status=="AUTHORISED" AND AmountDue>0
where=Contact.ContactID==Guid("d4e3e1a2-...")
where=Date>=DateTime(2026,01,01) AND Date<DateTime(2026,07,01)
where=Name.Contains("Acme")           // also StartsWith / EndsWith
where=IsCustomer==true
order=UpdatedDateUTC DESC
page=2
pageSize=100        // (where supported)
includeArchived=true
```

Best practice: prefer `If-Modified-Since` for delta sync, page on `UpdatedDateUTC` order, and use the `Journals` endpoint with `?offset=` for a strictly ordered, append-only ledger feed for financial reconciliation.

---

## 12. Best Practices

### Idempotency keys
Send an `Idempotency-Key` request header (UUID, ≤128 chars) on POST/PUT. If Xero saw the same key recently (24-hour window) it replays the original response instead of creating a duplicate. Essential for invoice / payment creation under flaky networks.

```
Idempotency-Key: 123e4567-e89b-12d3-a456-426614174000
```

### Retry strategy
- Retry only on 5xx and 429 (and on network errors).
- Exponential backoff with jitter; honour `Retry-After`.
- Cap concurrency at 5 per tenant (matches the concurrent limit).
- Combine with idempotency keys so retries are safe.

### Batching
- Use array bodies (up to ~1000 items per call) with `SummarizeErrors=false`.
- Reduce round trips on imports.

### Token hygiene
- Treat refresh tokens like passwords; store encrypted.
- After a successful refresh, persist the new refresh token immediately — the old one is invalidated.
- If a refresh fails, retry once within the 30-minute grace window using the previous refresh token.
- Refresh proactively a few minutes before the 30-minute access token expiry.

### Delta sync
- For Invoices/Contacts/Bank Transactions: `If-Modified-Since` + paged `order=UpdatedDateUTC ASC`.
- For an immutable audit log: the `Journals` endpoint with `?offset=` is the most reliable.

### Multi-tenancy
- One refresh token per user can correspond to many tenants — always re-call `/connections` after a refresh to detect added/removed tenants.
- Store `(userId, tenantId, connectionId)` triples; rebuild from `/connections`.

---

## Sources

- <https://developer.xero.com/documentation/guides/oauth2/auth-flow/>
- <https://developer.xero.com/documentation/guides/oauth2/pkce-flow>
- <https://developer.xero.com/documentation/guides/oauth2/custom-connections/>
- <https://developer.xero.com/documentation/guides/oauth2/scopes/>
- <https://developer.xero.com/documentation/guides/oauth2/tenants>
- <https://developer.xero.com/documentation/guides/oauth2/limits/>
- <https://developer.xero.com/documentation/best-practices/api-call-efficiencies/rate-limits>
- <https://developer.xero.com/documentation/best-practices/api-call-efficiencies/if-modified-since/>
- <https://developer.xero.com/documentation/api/accounting/overview>
- <https://developer.xero.com/documentation/api/accounting/requests-and-responses>
- <https://developer.xero.com/documentation/api/accounting/responsecodes>
- <https://developer.xero.com/documentation/guides/webhooks/overview/>
- <https://developer.xero.com/documentation/guides/webhooks/configuring-your-server/>
- <https://developer.xero.com/documentation/sdks-and-tools/libraries/overview/>
- <https://developer.xero.com/documentation/guides/idempotent-requests/idempotency/>
- <https://developer.xero.com/documentation/best-practices/data-integrity/managing-tokens>
- <https://developer.xero.com/documentation/development-accounts/>
- <https://developer.xero.com/documentation/api/finance/overview>
- <https://developer.xero.com/documentation/api/bankfeeds/overview>
- <https://github.com/XeroAPI/Xero-OpenAPI>
