# Phase 1: Discovery & Design — Live Agent Broker

## Scope
This document captures the requirements-gathering, architectural design, and planning phase for sn_live_agent_broker.

## Objectives
- Define functional and non-functional requirements.
- Map current-state pain points to future-state automation.
- Select integration patterns (REST, SOAP, GraphQL, JDBC, File).
- Design the scoped application skeleton and data model.
- Establish security, compliance, and audit requirements.
- Create a risk register and mitigation plan.

## Stakeholders
| Role | Contact | Responsibility |
|------|---------|---------------|
| Product Owner | Vladimir Kapustin | Scope, priority, acceptance criteria |
| ServiceNow Architect | Vladimir Kapustin | Technical design, Studio configuration |
| QA / SRE | TBD | Test planning, CI/CD pipeline |
| Security Officer | TBD | Threat modeling, ACL review |
| End Users | ServiceNow Admins & Developers | UAT, feedback, documentation review |

## Current State Analysis
Manual processes dominate the target workflow. Admins execute repetitive queries, export data to spreadsheets, apply filters by hand, and copy-paste results into Confluence or email. This process is error-prone, non-repeatable, and offers no audit trail. Security reviews are ad-hoc and rarely reproducible. Cross-instance comparisons require screen-sharing sessions rather than programmatic diffs.

## Requirements
### Functional Requirements
1. Automated extraction of target records from ServiceNow tables.
2. Rule-based classification and filtering with configurable logic.
3. Report generation in Markdown, JSON, and CSV.
4. Delta / incremental scanning with watermark tracking.
5. REST API endpoints for programmatic access.
6. Scheduled job support for recurring execution.
7. Health-check endpoint for monitoring integration.

### Non-Functional Requirements
1. Response time for API calls under 2 seconds (p95).
2. Processing throughput of at least 100,000 records per 5 minutes.
3. 99.9% availability during scheduled windows.
4. All credentials encrypted at rest and in transit.
5. Full audit trail with immutable logs.
6. Compatibility with ServiceNow Xanadu+ and ZURICH releases.

## Architecture Decisions
| Decision | Rationale |
|----------|-----------|
| Scoped Application | Isolation from global namespace; clean upgrade path |
| Python CLI + ServiceNow Scripts | Python for CI/CD and local analysis; ServiceNow for server-side logic |
| Delta Scanning | Minimizes instance load; supports continuous monitoring |
| Multi-format Reports | Serves technical and business audiences without extra tooling |
| Pluggable Processing Engine | Allows new rules without redeploying the entire app |

## Data Model (High-Level)
- `x_sn_live_agent_broker_config`: Application configuration and properties.
- `x_sn_live_agent_broker_scan_log`: Record of each scan execution (start, end, status, record count).
- `x_sn_live_agent_broker_report`: Generated reports with metadata (format, size, checksum).
- `x_sn_live_agent_broker_delta_watermark`: High-water marks for incremental scanning per table.

## Security & Compliance
- TLS 1.2+ mandatory for all external communication.
- Credentials stored in `sys_auth_profile` or environment variables.
- Scoped roles enforce least-privilege access.
- GDPR: PII anonymized via SHA-256 hashing before storage or transmission.
- SOX / ISO 27001: Immutable audit logs; role separation for config vs. read access.

## Risk Register
| ID | Risk | Impact | Probability | Mitigation |
|----|------|--------|-------------|------------|
| R1 | ServiceNow API rate limiting | High | Medium | Implement exponential backoff and delta scanning |
| R2 | Credential leakage | Critical | Low | Encrypted storage; no hardcoding; automated secret scanning |
| R3 | Scope collision with other apps | Medium | Low | Unique prefix `x_sn_live_agent_broker`; validate before install |
| R4 | Large table scans causing timeouts | High | Medium | Chunked pagination; configurable batch size |
| R5 | Incompatible future ServiceNow release | Medium | Low | Pin to supported releases; regression testing in sub-prod |

## Milestones
| Milestone | Target Date | Deliverable |
|-----------|-------------|-------------|
| M1 | Week 1 | Requirements document signed off |
| M2 | Week 2 | Architecture diagram and data model finalized |
| M3 | Week 3 | Security review and risk register approved |
| M4 | Week 4 | Development environment provisioned and skeleton deployed |

## Exit Criteria for Phase 1
1. All requirements documented and stakeholder-approved.
2. Architecture diagram reviewed by the ServiceNow Architect.
3. Data model created in Studio with core tables.
4. Security review completed with no critical findings.
5. Risk register accepted by the Project Sponsor.
