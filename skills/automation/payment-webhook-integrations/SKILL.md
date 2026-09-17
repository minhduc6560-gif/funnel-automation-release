---
name: payment-webhook-integrations
description: Build verified QR payment webhook flows.
version: 0.1.0
author: Minh Duc (minhduc6560-gif), Hermes Agent
license: MIT
platforms:
- linux
- macos
- windows
metadata:
  hermes:
    tags:
    - payments
    - webhooks
    - qr
    - cloudflare
    - d1
    - crm
    related_skills:
    - durable-crm-lead-pipelines
---

# QR Payment Webhook Integrations

Build payment flows around a server-side order ledger. Treat the landing page as an interface, the payment webhook as the source of payment truth, and the CRM as an operational mirror rather than the financial database.

## When to use

Use for landing pages that need a unique transfer QR per order, automatic payment confirmation, and downstream CRM automation. Do not use a static QR when the business needs reliable customer-to-transaction matching.

## Procedure

### 1. Preserve the approved landing page

- Clone a materially different payment direction into a versioned sibling folder before editing; keep the approved baseline untouched for comparison and rollback.
- If a generator owns the page, copy and edit the generator together with the generated artifact. Run the generator twice to prove the integration is idempotent.
- Record the new frontend, backend, database, webhook, and production URLs in the project's source-of-truth planning file.

### 2. Choose the smallest reliable hosting shape

Prefer managed serverless infrastructure for a small landing-page checkout:

- Cloudflare Worker for HTTPS API endpoints.
- Cloudflare D1 for orders and webhook events.
- A provider-issued HTTPS domain during development; attach a custom subdomain for production when required.

Before provisioning, verify Wrangler rather than assuming account state:

```bash
npx --yes wrangler@4.133.0 --version
npx --yes wrangler@4.133.0 whoami
```

A public HTTPS endpoint is mandatory for production payment webhooks. Never point a provider at localhost or a private network address.

### 3. Freeze business rules before enabling order creation

Require explicit values for:

- server-side unit price;
- quantity bounds;
- shipping policy and fee;
- receiving bank code, account number, and display name;
- payment-code prefix and the provider's exact extraction rule;
- pending-order expiry;
- underpayment, overpayment, and duplicate-payment handling.

Do not silently convert an unknown shipping fee to zero; that turns an unresolved policy into an accidental free-shipping promise. Add an explicit server-side payment kill switch that defaults off, include it in `/api/config` readiness, and reject order creation until pricing, recipient details, webhook authentication, and provider test delivery are complete. A successful deployment must not make an incompletely connected QR payable.

### 4. Separate public and secret configuration

Store non-sensitive values as Worker environment variables and secrets with Wrangler/provider secret storage. Never embed webhook secrets, bank API credentials, or CRM tokens in HTML, JavaScript, repositories, chat, or planning files.

Typical configuration:

```text
UNIT_PRICE
SHIPPING_FEE
PAYMENT_CODE_PREFIX
BANK_CODE
BANK_ACCOUNT_NUMBER
BANK_SETTLEMENT_ACCOUNT_NUMBER
BANK_ACCOUNT_NAME
PAYMENTS_ENABLED
ALLOWED_ORIGIN
CRM_LOCATION_ID
```

Typical secrets:

```text
PAYMENT_WEBHOOK_SECRET
CRM_API_TOKEN
```

### 5. Freeze the frontend/backend contract, then build the API

Before parallelizing frontend and backend work, write one shared contract for exact request field names, response keys, status values, and configuration endpoint behavior. Exercise the real frontend payload against the backend tests; independently built halves can both pass while disagreeing on names such as `full_name` versus `customerName`, or `total` versus `totalAmount`.

Implement at least:

```text
GET  /api/config
POST /api/orders
GET  /api/orders/{public_token}/status
POST /api/payment/webhook
GET  /health
```

Use `GET /api/config` to expose only non-secret checkout readiness and authoritative pricing. Keep order submission disabled until it confirms the expected unit price and an explicit shipping fee.

For `POST /api/orders`:

1. Validate customer and shipping fields against the frozen contract.
2. Recalculate all money server-side.
3. Generate a hard-to-guess payment code that preserves the provider-configured prefix exactly. Keep any suffix-length restriction scoped to the variable portion; do not remove the prefix from the transfer memo merely because the numeric suffix has a length limit.
4. Generate a separate opaque public token for status polling; do not expose an internal row ID.
5. Insert the order as `pending_payment`.
6. Return only the order code, public token, amount, recipient display data, and generated QR URL.

Use integer minor-free VND amounts, not floating-point currency arithmetic.

### 6. Make the webhook the source of truth

Process the raw request body in this order:

1. Verify provider authentication before trusting JSON.
2. For HMAC, sign the exact raw bytes; parsing and re-serializing JSON changes the signature input.
3. Validate timestamp freshness to block replay attacks.
4. Parse and validate required fields.
5. Require an incoming transfer to the configured receiving account. Validate the provider's structured fields as a bank-specific tuple, not as one universal account field: a QR alias/VA can appear in `subAccount` while `accountNumber` contains the settlement account. Keep the QR destination and settlement account as separate exact strings, require the configured gateway plus the documented tuple, and log only the masked matching path. Use authenticated prose fallback only when the provider documents that structured identifiers can be absent—not merely because they disagree.
6. Deduplicate on the provider transaction ID with a database `UNIQUE` or primary-key constraint.
7. Match the extracted payment code to exactly one order.
8. Compare the received amount to the server-calculated order total.
9. Store the original payload for audit and reconciliation.
10. Update the order idempotently to `paid`, `underpaid`, or a review state.
11. Return the provider's exact success status/body contract quickly; dispatch slow CRM calls asynchronously.

Never confirm payment from a browser redirect, thank-you page, frontend callback, screenshot, or customer claim. Those signals are not bank confirmation.

### 7. Keep the CRM downstream

- Store the canonical order and transaction in the payment database.
- Upsert the Contact and operational order/opportunity into the CRM after the ledger update.
- Record CRM synchronization status and retry failures; a transient CRM outage must not lose or reverse a valid payment.
- Trigger paid workflows only from the synchronized paid state, with an idempotency guard so webhook replay does not send duplicate messages.

### 8. Build the frontend as a state machine

Use explicit states such as:

```text
editing → creating → pending → paid
                         ├→ underpaid
                         ├→ expired
                         └→ error
```

For physical-product checkout in physical-product checkout pages:

- Put quantity first and use accessible minus/input/plus controls with explicit bounds.
- Recalculate the visible quantity, subtotal, shipping, and total immediately on button clicks and direct input changes; still treat the server-calculated total as authoritative.
- Prefer a short shipping address form: province/city, ward/commune, and street address. Do not add district or order notes unless the brief requires them.

General rules:

- Disable repeated submits while creating an order.
- Display the exact server-returned amount and payment code beside the QR.
- Provide copy controls for account, amount, and transfer content.
- Poll only with the opaque public token and stop polling at a terminal state.
- Never show a paid state before the status API confirms it.
- Keep a visible non-production message when backend pricing or payment configuration is incomplete.

#### Dedicated confirmation and QR page

When checkout continues on a separate route such as `/confirmation/`:

1. Create a real order first, then redirect with only the opaque public token in the URL fragment: `/confirmation/#<public_token>`. Never place customer name, phone, email, address, internal row ID, or payment status in the URL; the fragment is not sent in HTTP requests or referrers.
2. Let the confirmation page read the fragment and call `GET /api/orders/{public_token}/status`. The response may expose only payment-safe fields: branded order code, transfer content, quantity, total/paid amounts, status, recipient display data, server-generated QR URL, and timestamps. Do not return customer PII.
3. Validate the token shape before calling the API and validate the QR URL against the provider's exact HTTPS hostname before assigning it to an image.
4. Render “order received” separately from “payment confirmed.” A redirect proves order creation only; show `paid` language only after the status endpoint reports the webhook-backed paid state.
5. Poll `pending` and `underpaid`, stop on `paid` or error, and make missing/invalid tokens recoverable with a clear return link. Keep the token in the fragment across refresh so the page can restore the order without browser storage.
6. Keep a source generator and verifier for the nested page when the project is generator-owned. Include the nested route and its shared assets in preview and production staging directories.

### 9. Verify before real money

Run these gates in order:

1. Static frontend and backend checks.
2. Local order creation against an isolated local database.
3. Bad signature and stale timestamp rejection.
4. Correct signed webhook with insufficient amount → `underpaid`.
5. Correct signed webhook with sufficient amount → `paid`.
6. Replay the same provider transaction ID → no second state change or CRM action.
7. Desktop and true-width mobile render QA.
8. Deploy the payment variant to a preview branch/alias first; do not overwrite the approved production landing page.
9. Read back preview HTML and a prominent asset over HTTPS, then verify `/health` and `/api/config` from the exact preview Origin.
10. Exercise plus/minus controls in a real browser and assert the displayed totals for multiple quantities plus `scrollWidth <= clientWidth` on mobile.
11. If checkout redirects to a dedicated confirmation page, submit the actual preview form in a browser—not only a direct API fixture—and require the URL to become the intended route with exactly one opaque fragment token. Render the loaded pending-payment state at desktop and true-width mobile sizes, verify the safe status response and no horizontal overflow, then delete every QA order and query the ledger to prove cleanup.
12. Use the provider's test-send function and inspect its delivery log, including the exact HTTP status and response body. Distinguish a webhook-resource creation status from an actual delivery result: require a matching Worker invocation or ledger/diagnostic record rather than accepting a setup-screen `201` as delivery proof.
13. Run one controlled real transfer, then reconcile provider transaction, payment ledger, frontend state, and CRM record by the same order code.
14. If a real transfer receives a non-success webhook response, fix the receiver before asking for another transfer. Use provider auto-retry, replay, or an authenticated transaction-API reconciliation path; never make replay the only recovery mechanism and never ask the customer to pay twice.
15. Before promoting the preview, query both the order and webhook-event tables by the same payment code; require the order to be `paid`, the amount to match, and exactly one accepted provider event. A successful webhook log without the corresponding order mutation is not enough.

Do not report production readiness from a successful deployment alone.

### 10. Promote the verified variant to Cloudflare Pages production

When the custom domain is already attached to a Cloudflare Pages project, publish through that project rather than forcing the CRM application Page Builder workflow:

1. Regenerate twice and rerun the static verifier immediately before deployment.
2. Inspect `wrangler pages project list` and `wrangler pages deployment list --project-name <project>` to confirm the exact project, custom domain, production branch, and current rollback deployment.
3. Build a clean staging directory containing only public artifacts such as `index.html`, nested route pages, and `assets/`; do not deploy Worker source, tests, QA screenshots, plans, or secrets with the site.
4. Promote explicitly to the project's production branch:

```bash
npx --yes wrangler@4.133.0 pages deploy <stage-dir> \
  --project-name <project> \
  --branch <production-branch>
```

5. Read back the deployment list and require the new deployment to show `Production` on the expected branch.
6. Verify the custom-domain root HTML, every newly added nested route, and one prominent asset over HTTPS, then call `/api/config` with the exact custom domain in the `Origin` header. Compare local and downloaded production artifact hashes when exact promotion matters. Confirm the production HTML contains checkout/API markers, lacks the retired form embed, and receives the expected CORS origin, authoritative unit price, and shipping fee.
7. Update `PROJECT-PLAN.md` with the custom URL, immutable deployment URL/ID, branch, live-payment evidence, and production verification. Keep the preserved baseline source listed separately for rollback.

## Provider depth

For SePay payload fields, response contract, authentication, QR construction, and Cloudflare-specific implementation notes, read `references/sepay-cloudflare.md`.
