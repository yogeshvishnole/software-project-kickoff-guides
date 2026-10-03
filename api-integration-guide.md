# External Partner API Integration: Engineering Framework & Validation Guidelines

## 1. Overview & Purpose
This framework provides reusable engineering guidelines for planning, validating, executing, and operating third-party / external partner API integrations.

Integrating external systems introduces risks outside direct system control—such as undocumented schema constraints, non-idempotent endpoints, test-environment bottlenecks, and state synchronization drift. This document outlines structural requirements, failure modes, and a pre-flight validation checklist.

---

## 2. Integration Architecture & System Boundaries

Before starting implementation, define the integration boundaries across all involved systems:

- **Business Objective:** Clearly define the operational, transactional, or financial outcome.
- **System Ownership & Authority:** Specify which system acts as the **System of Record (Source of Truth)** for user identity, entity status, account/wallet balance, transaction state, and reconciliation.
- **API Topology:** Document every inbound/outbound request, webhook/callback, polling mechanism, status query, and dependent master-data API.
- **Out-of-Scope Behaviors:** Explicitly capture unsupported scenarios, manual fallbacks, rate limits, and known third-party limitations.

---

## 3. Core Reliability & Fault-Tolerance Principles

### 3.1 Idempotency & Retries
- **Rule:** A network timeout does **not** equal a transaction failure.
- Every state-changing API (create, debit, block, fulfill, cancel) must support idempotency keys (e.g., UUID / unique client reference).
- If the partner API is **non-idempotent**, retries must be disabled until a status/GET query confirms the state of the initial attempt.

### 3.2 Transient Failure Handling
- Differentiate between retryable (HTTP 429, 502, 503, 504, network drops) and non-retryable (HTTP 400, 401, 403, 422, unrecoverable business errors).
- Implement exponential backoff with jitter and circuit-breaker patterns for downstream protection.

### 3.3 State Drift & Pre-Flight Validation
- Real-time transactions must account for entity lifecycle changes (e.g., active in internal systems, but expired, blocked, or inactive in the partner's system).
- Perform pre-flight status validation as close to execution as possible.

### 3.4 Full-Dataset Reconciliation
- Status and GET endpoints must provide a complete audit and reconciliation dataset—not just high-level status strings.

---

## 4. Key Failure Modes & Technical Mitigations

| Failure Mode / Challenge | Impact | Mitigation / Design Pattern |
| :--- | :--- | :--- |
| **Schema Strictness (e.g., Decimal Rejection)** | Valid monetary amounts rejected due to unexpected integer/string requirements. | Test scale, precision, rounding, zero, negative, and string serialization across all endpoints during discovery. |
| **Hidden Boundary Limits (e.g., Date Windows)** | Transactions outside undocumented partner thresholds (e.g., ±N days) fail silently or throw generic errors. | Execute boundary-value analysis (exact limits, $N-1$, $N+1$) across past, present, and future timeframes. |
| **Test Environment & Credential Exhaustion** | Development blocks due to depleted test tokens, shared accounts, or rate limits. | Establish sandbox capacity, quotas, credential ownership, automated expiry alerts, and reset runbooks prior to sprint kick-off. |
| **Non-Idempotent Mutations** | Blind retries on timeout cause duplicate financial mutations or duplicate bookings. | Require partner-side idempotency keys. If unsupported, gate retries behind a mandatory GET/status reconciliation check. |
| **Entity Status Mismatch (Active Locally vs. Inactive Remotely)** | Transaction starts locally with valid status, but fails downstream due to stale remote master data. | Pre-flight partner status check + explicit error mapping + Standard Operating Procedure (SOP) for manual review/rollback. |
| **Incomplete GET / Status Payloads** | Inability to reconcile broken transactions after timeouts or partial failures. | Mandate that GET/status APIs expose internal reference, partner reference, amounts, timestamps, failure codes, and cancellation details. |

---

## 5. Partner API Pre-Flight Validation Checklist

Run this checklist during API discovery, sandbox certification, and integration testing.

### 5.1 Amount & Financial Fields
- [ ] Document permitted precision (e.g., integer vs. float, 2 decimal places, rounding mode).
- [ ] Test integer amounts, fractional amounts, and boundary values (e.g., `0`, `0.00`, `0.01`, `99999999.99`).
- [ ] Test excess precision handling (does partner reject `10.555` or truncate/round?).
- [ ] Test edge cases: negative amounts, empty values, nulls, scientific notation, strings.
- [ ] Verify currency codes (e.g., ISO-4217 `USD`, `INR`) and mismatch handling.
- [ ] Confirm consistency of amount values across Create, Read, Update, Refund, and Settlement APIs.
- [ ] Validate calculation consistency (e.g., Line Items + Taxes + Fees == Total Amount).

### 5.2 Date & Time Fields
- [ ] Document format (ISO 8601, Unix epoch, UTC vs. Local offsets).
- [ ] Test boundary past dates: exact lower limit, one day before, and historical out-of-range dates.
- [ ] Test boundary future dates: exact upper limit, one day after, and far-future dates.
- [ ] Test boundary conditions: midnight (`00:00:00`), end of day (`23:59:59`), leap years, month-end/year-end rollovers.
- [ ] Confirm whether date intervals are inclusive or exclusive.

### 5.3 Reference IDs, Transaction Keys & Idempotency
- [ ] Verify format, max length, supported character set, and case sensitivity.
- [ ] **Idempotency Test 1:** Send the exact same request payload with the same reference ID twice; verify the second returns the initial response without creating duplicates.
- [ ] **Idempotency Test 2:** Send a modified payload with a previously used reference ID; verify the partner rejects the conflict with HTTP 409 or equivalent.
- [ ] **Timeout / Recovery Test:** Simulate a network drop/timeout, then call the status/query API using the reference ID to confirm actual downstream state.
- [ ] **Concurrency Test:** Send duplicate concurrent requests to test for race conditions / lock contention.
- [ ] Verify reference ID lifecycle (can an ID be reused after failure/cancellation, or is it permanently burned?).

### 5.4 GET, Status & Reconciliation APIs
- [ ] Verify GET/status API returns the full entity model, not just status flags.
- [ ] Ensure the response includes:
  - Internal / Client Reference ID
  - Partner / Downstream Reference ID
  - Entity & Beneficiary Identifiers + current status
  - Transaction amounts, currency, and split breakdown
  - Granular status, error codes, and descriptive failure messages
  - Timestamps (Created, Updated, Settled)
  - Reversal, cancellation, or refund metadata
- [ ] Validate that partial failures, unacknowledged requests, and manual operations can be fully reconciled using GET APIs.

### 5.5 Entity Status Drift & Manual SOPs
- [ ] Test state transitions where an entity is active locally but inactive/suspended at the partner side.
- [ ] Define automated fallback: immediate local rollback vs. marking transaction in `PENDING_MANUAL_REVIEW`.
- [ ] **Standard Operating Procedure (SOP) Requirements:**
  - Dedicated operations/support team escalation channel.
  - Decision tree: partner account reactivation & retry vs. cancellation & refund.
  - Audit trail and customer notification communication templates.

---

## 6. Delivery & Production Readiness Controls

1. **Discovery-Phase Contract Testing:** Do not rely solely on partner documentation. Execute sandbox boundary tests to prove actual behavior.
2. **Explicit Partner Sign-off:** Secure written partner confirmation for undocumented constraints, rate limits, and boundary behaviors.
3. **Environment & Credential Safeguards:** Track API key expiry, sandbox reset policies, and ensure sandbox parity with production.
4. **Resilient Retry Design:** Never implement automated retry loops for mutating requests without idempotency guarantees or status lookups.
5. **Observability:** Propagate unified correlation IDs across headers, logs, and database records. Log sanitized payloads (masking PII/PCI).
6. **Reconciliation & Runbooks:** Implement daily automated reconciliation jobs and publish incident response runbooks before opening production traffic.