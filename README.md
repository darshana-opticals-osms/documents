# Optical Shop Management System (OSMS) — Documents Repository

[![CI Pipeline](https://github.com/darshana-opticals-osms/documents/actions/workflows/ci.yml/badge.svg)](https://github.com/darshana-opticals-osms/documents/actions)
![Release](https://img.shields.io/badge/release-v1.0.0-blue.svg)
![Course](https://img.shields.io/badge/SENG%2034213-System%20Development%20Project-brightgreen.svg)
![Automated Tests](https://img.shields.io/badge/automated%20tests-3416%20passed-success.svg)

Academic project documentation and deliverables repository for the **Optical Shop Management System (OSMS)** developed for **Darshana Opticals (Pvt) Ltd**.

This repository is submitted as part of **SENG 34213: System Development Project** (Year 3, Semester 2), Software Engineering Teaching Unit, Faculty of Science, University of Kelaniya, Sri Lanka. It continues directly from the prerequisite course **SENG 31242: System Design Project**.

---

## Live Application & Evaluator Access

In accordance with Chapter 9 of the University Development Guideline, the staged deployment environment and evaluation credentials are provided below for academic review:

| Service | Environment / Endpoint | Description |
|---|---|---|
| **Live Frontend** | [https://darshana-opticals.vercel.app](https://darshana-opticals.vercel.app) | Production-staged client web application |
| **Backend REST API** | [https://osms-backend-staging.onrender.com](https://osms-backend-staging.onrender.com) | Staged server-side API service |
| **Backend Health Check** | [https://osms-backend-staging.onrender.com/api/health](https://osms-backend-staging.onrender.com/api/health) | Automated service health and availability probe |

### Demonstration Credentials

The following role-based demo accounts are pre-configured in the staging environment for evaluation and panel walk-throughs:

| Role | Email | Password | Access Scope |
|---|---|---|---|
| **System Administrator** | `admin.dev@gmail.com` | `DevPassword123!` | System configuration, user management, audit trails, system logs |
| **Branch Manager** | `brachmanager@gmail.com` | `DevPassword123!` | Branch inventory, staff oversight, sales & inventory reporting |
| **Branch Assistant** | `branchassistant@gmail.com` | `DevPassword123!` | Customer order handling, dispensing, in-store checkout |
| **Optometrist** | `optometrist@gmail.com` | `DevPassword123!` | Clinical prescription records, patient consultation histories |
| **Customer** | `kavindueka@gmail.com` | `Amalka@123` | Frame catalogue, virtual try-on, appointment booking, order tracking |

> **Evaluator Notice:** These credentials belong exclusively to dedicated synthetic-data demonstration accounts configured for academic evaluation and testing purposes. They contain no personally identifiable information (PII) or real patient records.

---

## Project Repository Ecosystem

Following Section 3.1 of the University Development Guideline, the project is structured across dedicated repositories within the GitHub Organisation:

- **GitHub Organisation:** [https://github.com/darshana-opticals-osms](https://github.com/darshana-opticals-osms)
- **Project Board:** [OSMS Development Board](https://github.com/orgs/darshana-opticals-osms/projects) (Tracks Sprints 5–8 development issues, epics, and pull requests)

| Repository | Purpose | Primary Stack |
|---|---|---|
| [`documents`](https://github.com/darshana-opticals-osms/documents) *(this repository)* | Academic deliverables, specifications, ADRs, QA evidence, final report | Markdown, PDF, LaTeX |
| [`frontend`](https://github.com/darshana-opticals-osms/frontend) | Client-side responsive web application | React 18, Vite, Tailwind CSS, Vitest |
| [`backend`](https://github.com/darshana-opticals-osms/backend) | RESTful API, authentication, business logic, persistence | Node.js, Express, MongoDB, Mongoose, Jest |
| [`infra`](https://github.com/darshana-opticals-osms/infra) | Infrastructure as Code, CI/CD pipelines, Docker, E2E test suites | Docker, GitHub Actions, Playwright |

---

## Project Team & Academic Information

- **Degree Programme:** Bachelor of Science (Honours) in Software Engineering
- **Course Code & Title:** SENG 34213 – System Development Project
- **Academic Unit:** Software Engineering Teaching Unit, Faculty of Science, University of Kelaniya
- **Academic Year:** Year 3, Semester 2 (2026)
- **Client Organization:** Darshana Opticals (Pvt) Ltd
- **Development Team Members:**
  - Hiruni Imasha ([@thiruniimasha](https://github.com/thiruniimasha))
  - Kavindu Amalka ([@KavinduAmalka](https://github.com/KavinduAmalka))
  - Sachini Weerakkody ([@SachiniUresha](https://github.com/SachiniUresha))
  - Sandaru Mihiran ([@SandaruUOK](https://github.com/SandaruUOK))

---

## Documentation Index & Academic Deliverables

In compliance with Chapter 10 (Deliverables & Assessment) of the University Development Guideline, the indexed deliverables below reflect verified merged repository files:

### 1. Requirements Specifications

| Document | Version | Path | Status | Details / Traceability |
|---|---|---|---|---|
| Software Requirements Specification (SRS) | 1.1 (Final) | [`srs/srs-final.pdf`](srs/srs-final.pdf) | Merged | Implementation-aligned final revision (PR #53 / DDP-088) |
| Software Requirements Specification (SRS) | 1.0 (Baseline) | [`srs/srs-v1.0.pdf`](srs/srs-v1.0.pdf) | Approved | Historical design baseline approved in SENG 31242 |

### 2. Design Specifications

| Document | Version | Path | Status | Details / Traceability |
|---|---|---|---|---|
| Software Design Specification (SDS) | 1.1 (Final) | [`SDS/sds-final.pdf`](SDS/sds-final.pdf) | Merged | Implementation-aligned final revision (PR #53 / DDP-088) |
| Software Design Specification (SDS) | 1.0 (Baseline) | [`SDS/sds-v1.0.pdf`](SDS/sds-v1.0.pdf) | Approved | Historical design baseline approved in SENG 31242 |

### 3. Architectural Decision Records (ADRs)

Per Section 1.3 of the University Guideline, all significant deviations from the original design baseline are documented as accepted ADRs:

| ADR | Title | Path | Status | Date / Focus |
|---|---|---|---|---|
| ADR-001 | Clinical Prescription Data Storage Design | [`adr/ADR-001-clinical-prescription-storage.md`](adr/ADR-001-clinical-prescription-storage.md) | Accepted | Subdocument encapsulation for clinical optometric data |
| ADR-002 | Authoritative RBAC Role Model | [`adr/ADR-002-rbac-role-model.md`](adr/ADR-002-rbac-role-model.md) | Accepted | 5 system roles with explicit permission matrix |
| ADR-003 | Shopping Cart Persistence and Checkout Boundary | [`adr/ADR-003-shopping-cart-checkout-boundary.md`](adr/ADR-003-shopping-cart-checkout-boundary.md) | Accepted | Server-side cart validation and checkout boundaries |
| ADR-004 | Order Lifecycle and Tracking Status Model | [`adr/ADR-004-order-lifecycle-tracking.md`](adr/ADR-004-order-lifecycle-tracking.md) | Accepted | 6-state deterministic finite state machine for orders |
| ADR-005 | Loyalty Points Calculation and Persistence Model | [`adr/ADR-005-loyalty-points-model.md`](adr/ADR-005-loyalty-points-model.md) | Accepted | Tiered point accumulation and ledger persistence |
| ADR-006 | Inventory Reorder-Alert Rules and Persistence Model | [`adr/ADR-006-inventory-reorder-alert-model.md`](adr/ADR-006-inventory-reorder-alert-model.md) | Accepted | Low-stock threshold detection and automated alerting |
| ADR-007 | Appointment Scheduling, Availability, and Lifecycle Rules | [`adr/ADR-007-appointment-scheduling-lifecycle.md`](adr/ADR-007-appointment-scheduling-lifecycle.md) | Accepted | Slot concurrency control and appointment transitions |
| ADR-008 | Inventory Image Upload and Cloudinary Storage Architecture | [`adr/ADR-008-inventory-image-upload-cloudinary.md`](adr/ADR-008-inventory-image-upload-cloudinary.md) | Accepted | Cloudinary CDN integration with server-side validation |
| ADR-009 | Appointment Communication and Automated Reminder Scope Decision | [`adr/ADR-009-appointment-communication-policy.md`](adr/ADR-009-appointment-communication-policy.md) | Accepted | In-app/email notifications with automated reminder scope refinement |
| ADR-010 | AI Chatbot Architecture and Knowledge Strategy | [`adr/ADR-010-ai-chatbot-architecture-and-knowledge-strategy.md`](adr/ADR-010-ai-chatbot-architecture-and-knowledge-strategy.md) | Accepted | Domain-scoped assistance with server token budgeting |
| ADR-011 | Secure Payment Gateway Integration and Payment Lifecycle Architecture | [`adr/ADR-011-secure-payment-gateway-integration.md`](adr/ADR-011-secure-payment-gateway-integration.md) | Accepted | PayHere integration, server-side hash verification, idempotency |
| ADR-012 | Authoritative Immutable Audit-Trail Architecture | [`adr/ADR-012-immutable-audit-trail-architecture.md`](adr/ADR-012-immutable-audit-trail-architecture.md) | Accepted | Append-only audit logging for clinical & financial operations |
| ADR-013 | Authoritative Order-Status Notification Policy | [`adr/ADR-013-order-status-notification-policy.md`](adr/ADR-013-order-status-notification-policy.md) | Accepted | Event-driven customer alerts upon order milestone progression |
| ADR-014 | Authoritative Sales and Inventory Reporting Definitions, Metrics, and Access Policy | [`adr/ADR-014-sales-inventory-reporting-definitions.md`](adr/ADR-014-sales-inventory-reporting-definitions.md) | Accepted | Aggregated managerial reports and role-scoped analytics access |
| ADR-015 | Staging Backend Container Hosting Operational Fallback (Render.com) | [`adr/ADR-015-staging-backend-container-hosting-fallback.md`](adr/ADR-015-staging-backend-container-hosting-fallback.md) | Accepted | Render staging deployment fallback configuration |

### 4. Quality Assurance, Security & Testing Evidence

| Document / Artefact | Deliverable ID | Path | Status | Verification Scope |
|---|---|---|---|---|
| **Test Coverage Report (Sprint 8)** | Deliverable #6 / DDP-087 | [`testing/coverage-sprint8.pdf`](testing/coverage-sprint8.pdf) | Merged | Sprint 8 automated test coverage summary (PR #54) |
| **Test Case Register** | Deliverable #7 / DDP-087 | [`testing/test-register.pdf`](testing/test-register.pdf) | Merged | 40 representative functional & regression test cases (PR #54) |
| **Performance & Reliability Report** | Deliverable #9 / DDP-086 | [`testing/performance-report.pdf`](testing/performance-report.pdf) | Merged | Concurrency, latency (p95), and load testing results (PR #51) |
| **OWASP Top 10 Security Checklist** | Deliverable #8 / DDP-085 | [`security/owasp-checklist.pdf`](security/owasp-checklist.pdf) | Merged | Vulnerability assessment and security control evidence (PR #52) |
| **Integration & E2E Test Suite** | Guideline §6.3 / DDP-084 | Maintained in `infra` repo (`infra#11`, PR #19) | Referenced | 10 Playwright automated E2E user journeys and CI workflows |

### 5. Final Academic Reports & Standards

| Document | Deliverable ID | Path | Status | Details |
|---|---|---|---|---|
| **Final Development Report** | Deliverable #3 / DDP-089 | [`finalReport/Darshana Opticals (Pvt) Ltd Final Development Report .pdf`](finalReport/Darshana%20Opticals%20(Pvt)%20Ltd%20Final%20Development%20Report%20.pdf) | Merged | Comprehensive academic development report (PR #55 / Issue #48) |
| **Coding and Code Review Standards** | Guideline Chapter 5 | [`standards/coding-standards.md`](standards/coding-standards.md) | Active | SOLID, DRY, Clean Code, Airbnb style, PR etiquette (PR #9 / DDP-020) |
| **Sprint Retrospective Reports** | Deliverable #11 | Documented in GitHub Wiki / Report Appendices | Documented | Sprints 5–8 metrics, retrospectives, and velocity tracking |
| **Peer Evaluation Form** | Deliverable #12 | Confidential individual submission via eKelaniya LMS | Individual | Submitted individually within 48 hours per Guideline §10.2 |

---

## Testing Evidence, Scope & Technical Limitations

As required by university evaluation criteria, the testing results and verification boundaries are reported transparently:

### 1. Automated Test Results
- **3,416 Passing Automated Tests:** The combined automated test execution across the frontend (Vitest / React Testing Library) and backend (Jest / Supertest) test suites reports 3,416 passed tests in local and CI execution environments.
- **Representative Scope of DDP-087:** The 40 test cases cataloged in [`testing/test-register.pdf`](testing/test-register.pdf) represent selected primary functional paths (TC-FR-01 through TC-FR-15) and regression scenarios; they are a curated representative sample and do not represent an exhaustive list of all 3,416 unit/integration assertions.

### 2. Outstanding Verification Gaps & Operational Limitations
While core functionality is operational on the staged deployment, the following verification items remain qualified:
1. **Critical-Module Branch Coverage:** Although broad automated test volume is substantial, formal verification of the $\ge 90\%$ branch coverage threshold specifically isolated to designated critical business modules remains pending independent pipeline qualification.
2. **Endpoint-Level Integration Traceability:** 73 route endpoints are cataloged in the API source inventory; full path-specific integration assertions across all combinations remain qualified against the test register.
3. **Hosted PayHere Payment Gateway Confirmation:** Automated unit and mock integration tests pass successfully for payment handling; however, live hosted end-to-end payment confirmation via the hosted PayHere sandbox callback in the staging environment remains blocked / pending live merchant webhook verification.
4. **Acceptance Sign-Off Scope:** Local and CI automated test passes confirm software conformance and regression stability; they do not constitute formal client final acceptance or academic supervisor sign-off, which remain subject to final panel evaluation.

---

## Repository Directory Structure

```text
documents/
├── adr/                                                                 # Architectural Decision Records (ADR-001 to ADR-015)
│   ├── ADR-001-clinical-prescription-storage.md
│   ├── ADR-002-rbac-role-model.md
│   ├── ADR-003-shopping-cart-checkout-boundary.md
│   ├── ADR-004-order-lifecycle-tracking.md
│   ├── ADR-005-loyalty-points-model.md
│   ├── ADR-006-inventory-reorder-alert-model.md
│   ├── ADR-007-appointment-scheduling-lifecycle.md
│   ├── ADR-008-inventory-image-upload-cloudinary.md
│   ├── ADR-009-appointment-communication-policy.md
│   ├── ADR-010-ai-chatbot-architecture-and-knowledge-strategy.md
│   ├── ADR-011-secure-payment-gateway-integration.md
│   ├── ADR-012-immutable-audit-trail-architecture.md
│   ├── ADR-013-order-status-notification-policy.md
│   ├── ADR-014-sales-inventory-reporting-definitions.md
│   └── ADR-015-staging-backend-container-hosting-fallback.md
├── finalReport/                                                         # Capstone Development Report
│   └── Darshana Opticals (Pvt) Ltd Final Development Report .pdf
├── SDS/                                                                 # Software Design Specification
│   ├── sds-v1.0.pdf                                                     # Approved SENG 31242 design baseline
│   └── sds-final.pdf                                                    # Implementation-aligned SDS v1.1
├── security/                                                            # Security and Vulnerability Compliance
│   └── owasp-checklist.pdf                                              # DDP-085 OWASP Top 10 checklist
├── srs/                                                                 # Software Requirements Specification
│   ├── srs-v1.0.pdf                                                     # Approved SENG 31242 requirements baseline
│   └── srs-final.pdf                                                    # Implementation-aligned SRS v1.1
├── standards/                                                           # Engineering Standards
│   └── coding-standards.md                                              # Team coding and code review standards
├── testing/                                                             # Quality Assurance Evidence
│   ├── coverage-sprint8.pdf                                             # DDP-087 Sprint 8 test coverage report
│   ├── performance-report.pdf                                           # DDP-086 performance and load test report
│   └── test-register.pdf                                                # DDP-087 functional test case register
└── README.md                                                            # Repository documentation index and evaluator guide
```

---

## Evaluator Inspection & Verification Guide

For academic examiners and project evaluators reviewing this repository:

1. **Viewing Documentation:** All PDF deliverables are pre-compiled and can be viewed directly on GitHub or downloaded for local examination using any standard PDF viewer.
2. **Git Commit & PR Traceability:** Each deliverable in this repository is traceable to a specific GitHub issue and peer-reviewed pull request (e.g., PR #51 for performance reporting, PR #52 for OWASP checklist, PR #53 for final SRS/SDS, PR #54 for QA evidence, PR #55 for final report).
3. **Branch Protections & Workflow:** In accordance with the University Guideline, direct commits to `main` and `develop` are prohibited. All contributions require short-lived feature/docs branches, passing CI checks, and at least one approving peer review.
