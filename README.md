# documents

Project documentation repository for the **Optical Shop Management System (OSMS)** — Darshana Opticals (Pvt) Ltd.

---

## Documentation Index

### Requirements

| Document | Version | Path | Status |
|----------|---------|------|--------|
| Software Requirements Specification (SRS) | 1.0 | [`srs/srs-v1.0.pdf`](srs/srs-v1.0.pdf) | Approved |

### Design

| Document | Version | Path | Status |
|----------|---------|------|--------|
| Software Design Specification (SDS) | 1.0 | [`SDS/sds-v1.0.pdf`](SDS/sds-v1.0.pdf) | Approved |

### Architectural Decision Records (ADRs)

| ADR | Title | Path | Status |
|-----|-------|------|--------|
| ADR-001 | Clinical Prescription Data Storage Design | [`adr/ADR-001-clinical-prescription-storage.md`](adr/ADR-001-clinical-prescription-storage.md) | Proposed |
| ADR-002 | Authoritative RBAC Role Model | [`adr/ADR-002-rbac-role-model.md`](adr/ADR-002-rbac-role-model.md) | Accepted |
| ADR-003 | Shopping Cart Persistence and Checkout Boundary | [`adr/ADR-003-shopping-cart-checkout-boundary.md`](adr/ADR-003-shopping-cart-checkout-boundary.md) | Accepted |
| ADR-005 | Loyalty Points Calculation and Persistence Model | [`adr/ADR-005-loyalty-points-model.md`](adr/ADR-005-loyalty-points-model.md) | Accepted |
| ADR-006 | Inventory Reorder-Alert Rules and Persistence Model | [`adr/ADR-006-inventory-reorder-alert-model.md`](adr/ADR-006-inventory-reorder-alert-model.md) | Proposed |
| ADR-010 | AI Chatbot Architecture and Knowledge Strategy | [`adr/ADR-010-ai-chatbot-architecture-and-knowledge-strategy.md`](adr/ADR-010-ai-chatbot-architecture-and-knowledge-strategy.md) | Accepted |

### Development Standards

| Document | Path | Status |
|----------|------|--------|
| Coding and Code Review Standards | [`standards/coding-standards.md`](standards/coding-standards.md) | Active |

---

## Repository Structure

```text
documents/
├── srs/
│   └── srs-v1.0.pdf          # Approved SRS v1.0
├── SDS/
│   └── sds-v1.0.pdf          # Approved SDS v1.0
├── adr/
│   ├── ADR-001-clinical-prescription-storage.md
│   ├── ADR-002-rbac-role-model.md
│   ├── ADR-003-shopping-cart-checkout-boundary.md
│   ├── ADR-005-loyalty-points-model.md
│   ├── ADR-006-inventory-reorder-alert-model.md
│   └── ADR-010-ai-chatbot-architecture-and-knowledge-strategy.md
└── README.md
```

---

## Notes

- Implementation decisions must remain traceable to the approved SRS and SDS.
- Significant deviations from the approved SDS must be documented as an ADR before implementation.
- The final implemented SRS will be maintained as `srs/srs-final.pdf` after project completion.
