---
name: durable-crm-lead-pipelines
description: Route web leads through a durable CRM bridge.
version: 0.1.0
metadata:
  hermes:
    tags:
    - crm
    - leads
    - cloudflare
    - workers
    - idempotency
    related_skills:
    - ldg-ket-noi-data-crm
author: Minh Duc (minhduc6560-gif), Hermes Agent
license: MIT
platforms:
- linux
- macos
- windows
---

# Durable CRM Lead Pipelines

Build server-side lead capture for gated content, multi-step funnels, payments, retries, or multiple destinations. Prefer this pattern over browser tracking whenever the visitor must not see success until the submission is durably stored.

## Procedure

### 1. Choose the integration path

- Use browser External Tracking only for simple best-effort forms where attribution convenience outweighs delivery guarantees.
- Use a server-side bridge when the flow gates content, needs retries, sends to several systems, or handles secrets.
- Do not attach both paths to one submit without a shared idempotency design; independent listeners can create duplicate contacts or trigger automation twice.

### 2. Lock the public contract

Send normalized contact fields, consent, source, attribution, funnel context, and a client-generated `idempotencyKey`. Keep technical source IDs ASCII/header-safe and separate them from localized display names.

Validate content type, body size, field lengths, email/phone formats, consent, allowlisted enum values, and anti-bot fields before any write. Never return raw upstream errors or echo PII in public responses.

### 3. Persist before acknowledging

Write the lead to durable storage before returning success. Put a unique constraint on `idempotencyKey`; a repeated request must return the original lead ID without inserting or syncing a second time.

Track at least:

```text
pending | synced | error
crm_contact_id
attempt_count
last_error_sanitized
created_at
synced_at
```

Only unlock gated content after the durable write succeeds. Do not make the visitor wait for CRM availability once the local write is safe.

### 4. Deploy a same-origin Worker + D1 bridge

When one Cloudflare Worker serves both the static funnel and its API, bind static assets and route `/api/*` through the Worker before asset fallback. Keep lead and booking tables separate, but carry the durable lead ID into booking when available.

Deploy in this order:

1. Create D1 in the nearest appropriate region and capture its database ID.
2. Write and apply the schema remotely before deploying code.
3. Bind D1 explicitly in Wrangler configuration and run the Worker first for `/api/*`.
4. Change the frontend so gated content unlocks only after `/api/leads` returns a durable ID; show booking success only after `/api/bookings` persists.
5. Deploy the Worker and static assets together, then verify `/api/health` against the bound database.
6. Submit a uniquely labeled QA lead and booking, read both rows back by exact email or idempotency key, resubmit the same key to prove deduplication, then delete QA rows and verify both counts return to zero.

For a same-origin frontend and API, reject mismatched `Origin` values instead of adding broad CORS. Keep API errors generic and never log submitted PII.

### 5. Sync through an outbox

Attempt CRM upsert in a background task after persistence, then retry unsynced rows with a scheduled worker or queue. Use bounded exponential backoff; retry network, 429, and 5xx failures, but do not retry 401/403 indefinitely.

Keep private tokens in the platform secret manager. Pass the Location ID explicitly, match by stable contact fields, and use stable source/tags. Custom fields must use IDs read from the exact target Location; use a structured note instead of guessing a field ID.

### 6. Decouple booking configuration

Serve service and booking configuration from the backend. Keep a service disabled until its calendar URL is verified against the intended Location and an allowed hostname. Never reuse an active calendar discovered in another Location merely because its shape looks suitable.

### 7. Verify both states

After one production QA submission:

1. Read back the durable row by its exact idempotency key or email.
2. Read back the CRM Contact by exact email/phone.
3. Confirm source, tags, notes/custom fields, and duplicate count.
4. Resubmit the same idempotency key and prove row and Contact counts remain unchanged.

A Worker/API 2xx proves only that endpoint's contract; it does not prove CRM synchronization unless the Contact is read back.

## Operational gates

- Restrict CORS to production, stable preview aliases, and a constrained Pages-project suffix; never use global `*` for PII endpoints.
- Add body limits, prepared statements, honeypot or anti-bot controls, and sanitized logging.
- Define PII retention and deletion procedures before production.
- Keep calendar integration independently switchable so lead capture can launch safely before booking is ready.

## Pitfalls

- Persist first because a direct CRM call can fail after the UI has promised success.
- Keep idempotency at the database boundary because button disabling alone does not prevent retries or duplicate network delivery.
- Separate localized brand copy from technical identifiers because spelling or naming changes must not fork production analytics and automation.
- Verify the target Location before using contacts, fields, forms, or calendars because authenticated tooling may be paired to a different sub-account.
