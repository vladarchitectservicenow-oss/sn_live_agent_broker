# Regression Cases: sn_live_agent_broker

**Author:** Vladimir Kapustin
**License:** AGPL-3.0

---

## Regression Test Cases (8 cases)

### 1. Idempotent Validation — Re-run Produces Identical Results
**Setup:** Create a valid inbound text message.
**Action:** Call `validate_inbound()` twice on the same message.
**Expected:** Both calls pass without error. No side effects on session_store.
**Why:** Validators must be pure functions; double validation should not alter state.

### 2. Format Consistency Across Multiple Runs
**Setup:** Create 10 valid messages of varying types (text, file, system, typing, ended).
**Action:** Run `broker_session()` 3 times with identical input.
**Expected:** All 3 runs produce identical return values (same message list, no errors).
**Why:** Deterministic output is required for audit trail reproducibility.

### 3. Role Idempotency — Duplicate Session Registration
**Setup:** Create a session with `sessionId="sess-dup"`, `visitorId="v1"`.
**Action:** Call `broker_session()` twice with the same message.
**Expected:** Second call does not raise `SessionContinuityError` (same visitorId is fine).
**Why:** Duplicate registrations within the continuity window should be no-ops if visitorId matches.

### 4. Config Persistence — Session State Survives Independent Validations
**Setup:** Register a session via `broker_session()` with `sessionId="sess-persist"`, `visitorId="v1"`.
**Action:** Call `validate_inbound()` on a different `sessionId`. Then call `broker_session()` with `sessionId="sess-persist"`, `visitorId="v2"`.
**Expected:** Second call raises `SessionContinuityError` — the original session state persisted across unrelated operations.
**Why:** Session store must not be corrupted by unrelated validation calls.

### 5. Mixed Direction Brokering — Inbound Then Outbound
**Setup:** Validate a message as inbound, then validate the same message as outbound.
**Action:** `validate_inbound(msg)` → pass. `validate_outbound(msg)` → should fail (missing `routingQueue`).
**Expected:** Inbound passes; outbound raises `BrokerValidationError` about `routingQueue`.
**Why:** Direction-specific rules must be enforced independently.

### 6. Full Session Lifecycle — Open, Chat, End
**Setup:** Messages: `system(joined)`, `text("hello")`, `file(...)`, `text("goodbye")`, `system(left)`.
**Action:** Run entire sequence through `broker_session()` in one call.
**Expected:** All 5 messages pass validation. No continuity errors.
**Why:** End-to-end session lifecycle must work without intervention.

### 7. Boundary: 30-Minute Window Expiry
**Setup:** Register session at time T. Mock `datetime.now()` to return T+31 minutes.
**Action:** Send message with same `sessionId` but different `visitorId`.
**Expected:** No `SessionContinuityError` — the 30-minute window has expired, so the old session is considered stale.
**Why:** Session windows must expire correctly; stale sessions should not block new visitors.

### 8. Concurrent Session Isolation
**Setup:** Two independent sessions: `sess-A` (visitor `v-a`) and `sess-B` (visitor `v-b`).
**Action:** Interleave messages: A1, B1, A2, B2, A3, B3.
**Expected:** All pass. No cross-contamination between sessions.
**Why:** Session store must correctly isolate independent sessions.
