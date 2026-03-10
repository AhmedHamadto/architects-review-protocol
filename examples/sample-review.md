# Architecture Review — acme-orders-api
**Date:** 2026-03-08
**Scope:** 87 files across 6 modules
**Protocol Version:** 1.0

---

## Executive Summary

The codebase is structurally sound with clean module boundaries and consistent patterns. Two critical findings require architect attention before shipping: an unprotected admin endpoint and a missing transaction wrapper on multi-table order writes. Auth coverage is otherwise strong, and the agent made reasonable technology choices.

**Rating: Yellow — Moderate Confidence.** Address Tier 1 items before merging.

---

## Stop and Look

### 1. Unprotected Admin Endpoint
**Risk Score:** 24/27 (Irreversibility: 3, Blast Radius: 3, Likelihood: 3)
**Location:** src/routes/admin.py:42 (and 2 other admin routes)
**What:** The `/admin/users/delete` endpoint has no auth middleware applied. Any unauthenticated request can delete user accounts.
**Why it matters:** Data loss and security breach — user accounts can be deleted by anyone with the URL.
**What to verify:** Confirm `@require_admin` decorator is applied to all routes in `src/routes/admin.py`. Check for other admin routes missing auth.

### 2. Order Creation Missing Transaction Wrapper
**Risk Score:** 18/27 (Irreversibility: 3, Blast Radius: 2, Likelihood: 3)
**Location:** src/services/orders.py:78
**What:** `create_order()` writes to both `orders` and `order_items` tables but does not wrap the operation in a database transaction. A failure after inserting the order but before inserting items leaves orphaned orders.
**Why it matters:** Data corruption — orders exist without items, breaking downstream invoicing and fulfillment.
**What to verify:** Wrap in `async with db.transaction():` and add a test that simulates failure mid-write.

---

## Verify with a Test

- **Stripe webhook signature not validated** — src/routes/webhooks.py:15 — test that requests without valid `Stripe-Signature` header are rejected with 401
- **Cache TTL hardcoded to 3600s for user sessions** — src/services/cache.py:22 — verify TTL aligns with JWT expiry (currently 1800s), test that expired cache entries don't serve stale auth
- **Pagination missing on order list endpoint** — src/routes/orders.py:31 — test with 10,000+ orders, confirm response time and memory usage are acceptable
- **No retry limit on payment charge** — src/services/payment.py:45 — test that after 3 failed attempts, the operation fails rather than retrying indefinitely

---

## Noted

**Code Quality:** 3 findings
- God function `process_order()` is 280 lines — src/services/orders.py:120 — consider splitting into validation, charging, and fulfillment steps
- Unused import `datetime` in 4 files — src/utils/helpers.py:3, src/models/user.py:1, src/services/cache.py:2, src/routes/health.py:1
- Inconsistent error message formatting (some include error codes, some don't) — pattern across src/services/

**Testing:** 2 findings
- No tests for edge cases in order cancellation (partial refund, already-shipped) — src/services/orders.py:195
- Test fixtures contain hardcoded Stripe test key (sk_test_...) — tests/conftest.py:12 — move to environment variable

**Style & Consistency:** 1 finding
- Mixed use of `snake_case` and `camelCase` in API response fields — src/routes/orders.py returns `orderItems` while src/routes/users.py returns `user_email`

---

## Boundary Health

**services/orders <-> services/payment:**
- `create_order()` calls `PaymentService.charge(amount, currency)` — the payment service signature is `charge(amount_cents: int, currency_code: str)`. The parameter name `amount` vs `amount_cents` suggests a potential unit mismatch (dollars vs cents). **Verify the caller converts to cents before calling.**

**routes/webhooks <-> services/orders:**
- Webhook handler calls `OrderService.fulfill(order_id)` but the service method signature is `fulfill(order_id: str, shipping_info: dict)`. The webhook handler passes only `order_id`. **Missing required parameter — this will raise TypeError at runtime.**

**All other boundaries verified** — contracts match, error types align.

---

## Decisions for the Architect

- **JWT with RS256 for auth** — src/middleware/auth.py:8
  - Chosen: RS256 asymmetric signing with public key verification
  - Alternatives: HS256 (simpler, shared secret), OAuth2 with external provider
  - Risk if wrong: Key rotation is more complex; must manage keypair lifecycle

- **Redis for session caching** — src/services/cache.py:5
  - Chosen: Redis with 1-hour TTL
  - Alternatives: In-memory cache (simpler, no infra), database-backed sessions
  - Risk if wrong: Redis becomes a single point of failure; no documented fallback

- **Soft-delete for user accounts** — src/models/user.py:34
  - Chosen: `deleted_at` timestamp field, filtered in queries
  - Alternatives: Hard delete with audit log, anonymization
  - Risk if wrong: Soft-deleted data still exists in DB — GDPR compliance may require hard delete or anonymization

---

## Meta

| Metric | Value |
|--------|-------|
| Files scanned | 87 |
| Modules reviewed | 6 |
| Tier 1 findings | 2 |
| Tier 2 findings | 4 |
| Tier 3 findings | 6 |
| Boundary issues | 2 |
| Decisions flagged | 3 |
| Cross-cutting files | src/middleware/auth.py, src/db/connection.py, src/config.py |
| Protocol version | 1.0 |
