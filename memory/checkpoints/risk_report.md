# sn_live_agent_broker — Risk Report

**Product:** sn_live_agent_broker
**Scope:** x_sn_live_agent_broker
**Author:** Vladimir Kapustin
**License:** AGPL-3.0
**Last updated:** 2026-05-26

---

## Risk Matrix

| Severity | Count | Definition |
|----------|-------|------------|
| P0 | 3 | Production outage or data loss possible within 24h |
| P1 | 5 | Feature degradation, manual workaround required |
| P2 | 6 | Non-blocking but reduces robustness |
| P3 | 4 | Future risk or cosmetic |

---

## P0 — Critical (Must Fix Before Production)

### RISK-P0-01: Live Agent Plugin Not Activated
**Category:** Platform dependency
**Description:** If the ServiceNow `com.glide.live_agent` plugin is not activated on the target instance, the broker cannot read `live_agent_session` or `live_agent_queue` tables. All brokering operations fail.
**Impact:** Complete outage — zero messages processed.
**Likelihood:** Medium (common in dev instances; admin oversight)
**Mitigation:**
1. Health-check endpoint verifies plugin activation on startup.
2. CI pipeline includes a plugin-activation gate before production deployment.
3. Admin documentation lists required plugins prominently.
**Detection:** Health endpoint returns `{"status": "degraded", "missing_plugins": ["com.glide.live_agent"]}`.

### RISK-P0-02: Session Continuity Race Condition
**Category:** Concurrency
**Description:** The in-memory `session_store` dict is not thread-safe. Under concurrent load from multiple REST endpoints, two simultaneous messages for the same `sessionId` could race on `_update_session()`, causing stale `last_seen` values and false continuity violations.
**Impact:** Legitimate messages rejected as `SessionContinuityError` — user chats interrupted.
**Likelihood:** High (any deployment with ≥2 concurrent connections)
**Mitigation:**
1. Production deployments must use Redis with distributed locking (`SETNX`).
2. `broker_session()` calls are sequential within a single session — partial protection.
3. Document thread-safety limitation in README Troubleshooting.
**Detection:** Load test with 100+ concurrent sessions; count false `SessionContinuityError`.

### RISK-P0-03: VisitorId Spoofing via Null Bypass
**Category:** Security
**Description:** `_check_session_continuity()` skips the visitorId match when `visitor_id is None` (line 169 of `broker_validator.py`). An attacker could send messages with a known `sessionId` but omit `visitorId`, bypassing session continuity checks.
**Impact:** Unauthorized message injection into existing chat sessions.
**Likelihood:** Medium (requires knowledge of active sessionId)
**Mitigation:**
1. Require `visitorId` on all session-continuity checks (remove `is not None` guard).
2. Track anonymous sessions separately with a null-token visitorId.
3. Add rate-limiting per sessionId to limit brute-force enumeration.
**Detection:** Penetration test: send message with stolen sessionId, null visitorId → should reject.

---

## P1 — High (Fix Within Sprint)

### RISK-P1-01: No CI/CD Pipeline
**Category:** DevOps
**Description:** Repository lacks GitHub Actions workflows for automated testing, linting, and license header verification. Manual testing is the only quality gate.
**Impact:** Regression bugs reach production; lint violations accumulate.
**Likelihood:** High (manual testing is inconsistent)
**Mitigation:**
1. Add `.github/workflows/ci.yml` with: `python -m unittest tests/test_broker.py -v`, `ruff check src/`, `black --check src/ tests/`.
2. Add LICENSE header verification script.
3. Run CI on every PR to main.

### RISK-P1-02: ISO-8601 Regex Too Strict for Edge Cases
**Category:** Input validation
**Description:** The ISO-8601 regex (`^\\d{4}-\\d{2}-\\d{2}T...`) rejects valid formats: milliseconds with 1-2 digits (`2026-05-26T12:00:00.5Z`), date-only (`2026-05-26`), and space-separated (`2026-05-26 12:00:00Z`). These are all valid per RFC 3339.
**Impact:** Valid Live Agent messages with edge-case timestamps rejected.
**Likelihood:** Low (most Live Agent clients use strict ISO-8601 with 3-digit ms)
**Mitigation:**
1. Replace regex with `datetime.fromisoformat()` try/catch after normalization.
2. Accept Python's built-in ISO-8601 parser which handles all RFC 3339 variants.
3. Add fuzz test with 500+ timestamp variants.

### RISK-P1-03: No Retry/Backoff on External REST Calls
**Category:** Resilience
**Description:** The broker has no retry logic for outbound REST calls to third-party endpoints (Slack, Teams). A transient network error drops the message permanently.
**Impact:** Message loss during network hiccups — 1-5% of volume.
**Likelihood:** High (network transient failures are common in cloud environments)
**Mitigation:**
1. Add exponential backoff with jitter: 1s, 2s, 4s, 8s (max 3 retries).
2. Dead-letter queue for messages that fail after all retries.
3. Integration test: simulate 500 → retry → success flow.

### RISK-P1-04: Session Store Not Persistent Across Restarts
**Category:** Data loss
**Description:** The in-memory `session_store` dict is wiped on process restart. Active sessions lose continuity tracking, and the 30-minute window resets.
**Impact:** Existing chat sessions briefly vulnerable to visitorId mismatch during restart window.
**Likelihood:** Medium (restarts are rare in production but do happen during deploys)
**Mitigation:**
1. Production: use Redis with `EXPIRE <sessionId> 1800` (30 min TTL).
2. Graceful shutdown: serialize session_store to disk on SIGTERM, reload on startup.
3. Blue-green deployment to avoid session loss during deploys.

### RISK-P1-05: No Rate Limiting on Validation Endpoints
**Category:** DoS resilience
**Description:** `validate_inbound()` and `validate_outbound()` have no rate limiting. An attacker could flood the broker with 100,000 invalid messages/second, exhausting CPU.
**Impact:** Denial of service — legitimate messages starved.
**Likelihood:** Low (requires access to broker endpoint)
**Mitigation:**
1. Token-bucket rate limiter: 1000 req/s per source IP.
2. Circuit-breaker: after 50 consecutive errors, reject all for 30 seconds.
3. Monitor `BrokerValidationError` rate and alert on spike.

---

## P2 — Medium (Fix This Quarter)

### RISK-P2-01: No Structured Logging
**Category:** Observability
**Description:** Errors are raised as exceptions but not logged to a structured format. Production debugging requires grep on stdout.
**Impact:** Increased MTTR during incidents — no correlation IDs, no log aggregation.
**Likelihood:** High (every production incident requires log analysis)
**Mitigation:**
1. Add `logging` module with JSON-formatted output.
2. Include `correlation_id`, `session_id`, `timestamp`, `error_code` in every log line.
3. Compatible with Splunk/Datadog/Elastic ingestion.

### RISK-P2-02: Hardcoded Session Window
**Category:** Configurability
**Description:** `_SESSION_WINDOW_MINUTES = 30` is a class constant — cannot be changed without modifying source code.
**Impact:** If business rules require 15-min or 60-min window, must fork.
**Likelihood:** Medium (different organizations have different session policies)
**Mitigation:**
1. Accept `session_window_minutes` as constructor parameter.
2. Default to 30 if not specified.
3. Document in README Configuration section.

### RISK-P2-03: Performance Untested at Scale
**Category:** Performance
**Description:** No load tests beyond ~50K messages. Unknown behavior at 500K+ messages/hour.
**Impact:** Unexpected latency under peak load (Black Friday, major incident).
**Likelihood:** Medium (peak loads are rare but high-impact)
**Mitigation:**
1. Run 1M-message load test with `time` measurements.
2. Profile with `cProfile` to identify bottlenecks.
3. Add performance benchmarks to CI (fail if >10% regression).

### RISK-P2-04: No Input Sanitization Against Injection
**Category:** Security
**Description:** Message payload `body` is validated for presence but not sanitized. A malicious payload with `<script>`, SQL injection, or log-injection patterns passes through unchecked. If the downstream consumer renders HTML or logs unsanitized, this becomes an XSS or log-injection vector.
**Impact:** XSS in chat UI if consumer doesn't sanitize; log injection if downstream logs raw payload.
**Likelihood:** Medium (downstream consumers may not expect malicious payloads)
**Mitigation:**
1. Strip HTML tags from `body` if output format is HTML.
2. Sanitize log-injection characters (`\n`, `\r`, `\x00`) in payloads before logging.
3. Document that consumers must sanitize. Defense-in-depth: broker sanitizes anyway.

### RISK-P2-05: No Message Size Limits
**Category:** Resource exhaustion
**Description:** No maximum limit on `payload.body` length. An attacker could send a 100MB body, exhausting memory on the broker process.
**Impact:** OOM kill; process restart; brief outage.
**Likelihood:** Low (requires intentional abuse or buggy client)
**Mitigation:**
1. Enforce max payload size: 1MB for text, 50MB for file metadata.
2. Reject oversized messages with clear error before deserialization.
3. Document limits in API Reference.

### RISK-P2-06: README Has Duplicate Sections (G8 Gate)
**Category:** Documentation quality
**Description:** The README has a legacy stub (~30 lines) appended before the expanded content, causing duplicate "## Architecture", "## License", "## Troubleshooting" sections.
**Impact:** Confuses readers; violates G8 gate.
**Likelihood:** Certain (current state)
**Mitigation:**
1. Strip lines 1-31 (the legacy stub) from README.
2. Verify with `grep -c '^## Architecture$' README.md` → must be exactly 1.

---

## P3 — Low (Backlog)

### RISK-P3-01: No i18n Support
**Category:** Internationalization
**Description:** Error messages are hardcoded in English. Non-English-speaking administrators receive opaque errors.
**Impact:** Support burden for non-English deployments.
**Likelihood:** Low (most ServiceNow admins read English)
**Mitigation:** Extract error messages to `errors.json` keyed by `error_code` + locale.

### RISK-P3-02: No API Versioning
**Category:** API stability
**Description:** The broker has no versioned API contract. Breaking changes to message schema could silently break integrations.
**Impact:** Integration failures on update.
**Likelihood:** Low (message schema is stable per Live Agent spec)
**Mitigation:** Add `version` field to broker API; reject messages with unsupported version.

### RISK-P3-03: No Health Check Endpoint
**Category:** DevOps
**Description:** No built-in HTTP health check endpoint. Monitoring tools cannot verify broker readiness without sending test messages.
**Impact:** Monitoring blind spot during startup.
**Likelihood:** Low (broker is embedded, not standalone service)
**Mitigation:** Add `Flask`/`FastAPI` wrapper with `/health` endpoint if deployed as microservice.

### RISK-P3-04: Dependency on Python 3.11+ `datetime.fromisoformat`
**Category:** Platform compatibility
**Description:** `datetime.fromisoformat("Z")` support was added in Python 3.11. Older Python versions (e.g., 3.9 on RHEL 8) cannot run the broker.
**Impact:** Cannot deploy on legacy enterprise Python installations.
**Likelihood:** Low (Python 3.11+ is widely available as of 2026)
**Mitigation:** Document Python version requirement clearly; provide Docker image with Python 3.12.

---

## Risk Heatmap

| Likelihood →<br>Impact ↓ | Low | Medium | High |
|-------------------------|-----|--------|------|
| **Critical** | — | P0-01 (Plugin), P0-03 (Spoof) | P0-02 (Race) |
| **High** | P1-02 (ISO), P1-05 (Rate) | P1-04 (Session loss) | P1-01 (No CI), P1-03 (Retry) |
| **Medium** | P2-05 (Size), P3-02, P3-03 | P2-02, P2-03, P2-04, P2-06 | P2-01 (Logging), P3-01, P3-04 |

---

## Summary

| Severity | Count | Action |
|----------|-------|--------|
| P0 | 3 | Must fix before production deployment |
| P1 | 5 | Fix within current sprint |
| P2 | 6 | Fix this quarter |
| P3 | 4 | Backlog — address as capacity allows |

**Highest priority action:** Add CI/CD pipeline (P1-01) and document production Redis requirement (P0-02). These two changes eliminate the largest risks with the least effort.
