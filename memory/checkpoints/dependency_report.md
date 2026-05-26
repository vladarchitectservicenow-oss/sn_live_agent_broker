# sn_live_agent_broker — Dependency Report

**Product:** sn_live_agent_broker
**Scope:** x_sn_live_agent_broker
**Author:** Vladimir Kapustin
**License:** AGPL-3.0

---

## 1. Internal Dependencies (ServiceNow Platform)

### Required ServiceNow Plugins

| Plugin ID | Plugin Name | Required | Version | Notes |
|-----------|-------------|----------|---------|-------|
| `com.glide.live_agent` | Live Agent | Yes | Zurich+ | Core Live Agent chat subsystem |
| `com.glide.rest.api` | REST API | Yes | Zurich+ | Inbound REST endpoint support |
| `com.snc.process_flow` | Process Flow | No | Any | If using brokered workflow triggers |

### ServiceNow Tables Referenced

| Table | Scope | Access | Purpose |
|-------|-------|--------|---------|
| `sys_log` | Global | Write | Structured JSON logging for all operations |
| `live_agent_queue` | Live Agent | Read | Queue membership validation |
| `live_agent_session` | Live Agent | Read | Session state queries |
| `sys_auth_profile` | Global | Read | Encrypted credential store (production only) |
| `sys_properties` | Global | Read/Write | Scoped configuration properties |
| `x_sn_live_agent_broker_config` | Scoped | Read/Write | Application-specific settings |
| `x_sn_live_agent_broker_log` | Scoped | Write | Audit trail for broker operations |

### ServiceNow Roles

| Role | Purpose |
|------|---------|
| `x_sn_live_agent_broker.admin` | Full configuration, deployment, report access |
| `x_sn_live_agent_broker.user` | Read-only report viewing and health-check access |
| `x_sn_live_agent_broker.api` | Service account role for CI/CD pipelines and automated brokering |

---

## 2. External Dependencies (Python Runtime)

### Runtime Dependencies

| Package | Version | Required | Purpose |
|---------|---------|----------|---------|
| Python stdlib | 3.11+ | Yes | `json`, `re`, `datetime`, `typing`, `unittest` |

**Zero external pip dependencies.** The broker uses only Python standard library modules.

### Test Dependencies

| Package | Version | Required | Purpose |
|---------|---------|----------|---------|
| Python stdlib | 3.11+ | Yes | `unittest` test framework |
| `pytest` | 7.x+ | Optional | Alternative test runner (compatible with unittest) |
| `requests` | 2.x+ | Optional | Only if integration tests hit real ServiceNow |

---

## 3. Third-Party Integration Dependencies

### Supported External Systems

| System | Protocol | Auth Method | Purpose |
|--------|----------|-------------|---------|
| Slack | HTTPS/REST | OAuth 2.0 Bot Token | Chat message relay |
| Microsoft Teams | HTTPS/REST | Azure AD App | War room bot integration |
| Custom Chatbots | HTTPS/REST | API Key / Bearer | Generic webhook consumer |
| Redis | TCP | AUTH token | Production session store (alternative to in-memory dict) |
| PostgreSQL | TCP | SSL + password | Production audit log backend |

---

## 4. CI/CD Pipeline Dependencies

| Tool | Purpose | Required |
|------|---------|----------|
| GitHub Actions | Automated test runner, linting, license header checks | Yes |
| `shellcheck` | Bash script linting | Yes |
| `black` | Python code formatting | Yes (CI only) |
| `ruff` | Python linting | Yes (CI only) |
| `git` | Version control, push/pull | Yes |
| `python3.11+` | Runtime | Yes |

---

## 5. Risk Assessment: Dependency Surface

### P0 — Critical (blocker)

| Dependency | Risk | Mitigation |
|------------|------|------------|
| Live Agent plugin disabled | All brokering stops | Health check monitors plugin status; alert if inactive |
| REST API plugin missing | Inbound endpoints unavailable | CI deploys test suite that pings REST endpoint before production release |

### P1 — High

| Dependency | Risk | Mitigation |
|------------|------|------------|
| Python version mismatch (< 3.11) | `datetime.fromisoformat("Z")` not supported | `pyproject.toml` enforces `requires-python >= 3.11` |
| Redis unavailable (production) | Session continuity checks fail open/closed | Configurable fail-open vs fail-closed mode |
| Slack Teams API rate limit | Messages delayed/dropped | Exponential backoff with retry queue |

### P2 — Medium

| Dependency | Risk | Mitigation |
|------------|------|------------|
| `sys_properties` table inaccessible | Configuration reads fail | Default fallback values in code for all properties |
| `sys_log` table full | Log writes fail gracefully | Ring-buffer log; alert when 80% capacity |
| GitHub Actions outage | CI pipeline blocked | Local pre-commit hooks mirror CI checks |
| ISO-8601 regex stale | New timestamp formats rejected | Fuzz test with 500+ timestamp variants per release |

### P3 — Low

| Dependency | Risk | Mitigation |
|------------|------|------------|
| Third-party chatbot API changes | Integration breaks | Versioned API contracts; webhook schema validation |
| PostgreSQL SSL cert rotation | Connection fails | Automated cert refresh via Vault/AWS Secrets Manager |

---

## 6. Lockfile & Pinning Strategy

- **No pip requirements.txt** for core broker — stdlib only eliminates supply-chain risk.
- For CI linting (`black`, `ruff`) — versions pinned in `.github/workflows/ci.yml`.
- For integration test extensions — optional `requirements-dev.txt` with pinned hashes.

---

## 7. Upgrade Compatibility Matrix

| Component | Zurich | Washington | Xanadu | Australia |
|-----------|--------|------------|--------|-----------|
| Live Agent plugin | ✓ | ✓ | ✓ | ✓ |
| REST API v2 | ✓ | ✓ | ✓ | ✓ |
| GraphQL API | — | ✓ | ✓ | ✓ |
| Git Integration | — | ✓ | ✓ | ✓ |
| Now Assist | — | — | ✓ | ✓ |

sn_live_agent_broker is tested against Zurich and Washington. Xanadu and Australia support is verified via API compatibility, not full regression.
