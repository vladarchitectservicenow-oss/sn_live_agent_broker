# Phase 2: Build & Validate — Live Agent Broker

## Scope
This document covers the implementation, testing, quality assurance, and packaging phase for sn_live_agent_broker.

## Build Environment
- **ServiceNow Instance:** Developer instance running ZURICH patch 2.
- **Local Development:** Python 3.11+, VS Code, Git.
- **CI/CD:** GitHub Actions (lint, test, license check).
- **Repositories:** GitHub — `vladarchitectservicenow-oss/sn_live_agent_broker`.

## Implementation Breakdown
### Module: Acquisition Layer
- REST client for Table API and GraphQL.
- Connection pooling and keep-alive for performance.
- Exponential backoff on 429/502/503.
- Configurable timeout and chunk size.

### Module: Processing Layer
- Script Include (`sn_live_agent_brokerProcessor`) encapsulating business rules.
- Pluggable classifier supporting whitelist and regex engines.
- Normalization of record fields (trim, lowercase enums, date ISO8601).

### Module: Storage Layer
- Scoped table writes with `GlideRecord` batching.
- Watermark updates in `x_sn_live_agent_broker_delta_watermark`.
- Report artifacts stored as attachments on `x_sn_live_agent_broker_report`.

### Module: Presentation Layer
- Markdown renderer with Jinja-like templating.
- JSON serializer with schema validation.
- CSV writer with UTF-8 BOM for Excel compatibility.

### Module: API & Integration
- Scripted REST API (`sn_live_agent_brokerAPI`) with named resources.
- Basic auth over HTTPS; scoped role enforcement.
- Health endpoint returning JSON telemetry.

### Module: Scheduled Jobs
- Recurring scheduled job for nightly delta scanning.
- On-demand execution via UI Action or REST.
- Job status tracked in `x_sn_live_agent_broker_scan_log`.

## Configuration Management
All configuration is externalized to scoped properties and environment variables. No hardcoded endpoints, credentials, or magic numbers exist in the source code. This facilitates multi-environment promotion without code changes.

## Testing Strategy
### Unit Tests
- Acquisition: mock HTTP responses; verify timeout and retry behavior.
- Processing: classify sample records; assert correct labels.
- Storage: mock `GlideRecord`; verify batch insert counts.
- Presentation: render fixtures; compare against golden files.

### Integration Tests
- Live Table API queries against developer instance (read-only).
- End-to-end CLI invocation with real credentials (local only).
- REST API schema validation using `responses` and `jsonschema`.

### Security Tests
- ACL matrix: confirm `user` role cannot POST to scan endpoint.
- Secret scanning: `truffleHog` and `git-secrets` in CI.
- Dependency audit: `safety` or `pip-audit` in CI pipeline.

### Performance Tests
- Scan 100,000 incident records; assert under 300 seconds.
- Generate 1,000-row CSV; assert under 10 seconds.
- Health endpoint p95 latency under 100ms.

## Quality Gates
| Gate | Criteria | Owner |
|------|----------|-------|
| Code Review | All commits reviewed by at least one peer | Lead Dev |
| Test Pass | 100% unit tests green; integration tests green | QA |
| Lint | `flake8` and `pylint` zero errors; warnings documented | Lead Dev |
| Security | No critical or high CVEs in dependencies; secrets scan clean | Security |
| Documentation | README > 2000 words; SOP complete; Phase 1+2 docs present | Tech Writer |
| License | Full AGPL-3.0 text in LICENSE; headers in source files | Compliance |

## Packaging & Delivery
1. Application scoped and published to Application Repository.
2. Source pushed to GitHub with signed commits.
3. Tag release following SemVer (e.g., `v1.0.0`).
4. GitHub Release notes include changelog, known issues, and migration notes.
5. Update set generated as fallback for instances without AppRepo.

## Deployment Environments
| Environment | Purpose | Refresh Frequency |
|-------------|---------|-------------------|
| Dev | Active development; feature branches | On commit |
| Test | Integration testing; regression suites | Weekly from dev |
| Staging | UAT; performance baseline; security scan | Monthly from test |
| Production | Live business value | Quarterly or on-demand from staging |

## Rollback Plan
If a release introduces regressions:
1. Halt scheduled jobs.
2. Revert the update set or restore the previous AppRepo version.
3. Validate health endpoint returns expected prior state.
4. Notify stakeholders via incident ticket.
5. Root-cause analysis within 48 hours.

## Known Issues & Limitations
- GraphQL API support is experimental; fallback to Table API recommended.
- PDF generation requires an external microservice (future roadmap).
- Cross-instance sync uses REST polling rather than webhooks (webhooks planned for v1.2).

## Exit Criteria for Phase 2
1. All modules implemented and peer-reviewed.
2. Test suite passes 100% (unit + integration).
3. Security scan shows zero critical/high findings.
4. README exceeds 2000 words and is stakeholder-approved.
5. Phase 1 and Phase 2 documentation merged to `main`/`master`.
6. Release tagged and pushed to GitHub.
