# ADR-001: Clinical Prescription Data Storage Design

## 1. Title

Clinical Prescription Data Storage Design for OSMS

## 2. Status

**Proposed** — Pending peer review and team approval.

> Once approved by the team, update this status to **Accepted** and record the approval date.

---

## 3. Context

The Optical Shop Management System (OSMS) is required to support two prescription-related functional requirements:

- **FR-002** — Profile management and prescription history: customers must be able to view their own prescription history through their profile.
- **FR-013** — Clinical prescription data recording: authorized Optometrists / Medical Staff must be able to record and securely store clinical prescription data.

The approved SDS (v1.0) confirms that the MongoDB data layer must support detailed clinical prescription histories (Section 1.2.1) and that sensitive medical data requires restricted access and long-term protection (Section 6.3).

### Design Gap in the Approved SDS

The SDS Section 2.2 presents the Final Normalized Database Schema (3NF) with the following entities:

| Entity | Purpose |
|--------|---------|
| Customer | Customer profile data |
| Staff | Staff and branch assignment |
| Admin | Administrative accounts |
| Branch | Physical branch details |
| Inventory | Stock per branch |
| Order | Customer purchase records |
| OrderItem | Line items per order |
| Payment | Transaction records |
| Appointment | Booking records |
| Chatbot | Chat session records |
| KnowledgeArea | Chatbot knowledge base |

**A dedicated `Prescription` entity is not defined in this schema.**

Because FR-002 and FR-013 require prescription storage and retrieval, and the SDS data model does not define how prescriptions are represented, a design decision is needed before implementation begins. This decision must be documented as an ADR so that the implementation remains fully traceable to the approved SRS and SDS.

---

## 4. Problem

Before FR-002 and FR-013 can be implemented, the team must agree on:

1. **Where** clinical prescription records will be stored (dedicated collection, embedded document, or other).
2. **How** a prescription record is related to a `Customer` and to the responsible Optometrist (`Staff`).
3. **How** prescription history is maintained (historical records vs overwrite).
4. **Which roles** are permitted to create, modify, or view prescription data.
5. **Which security and retention obligations** apply to prescription records.
6. **Which clinical fields** are required and confirmed by stakeholders.

---

## 5. Considered Options

### Option A — Dedicated `Prescription` MongoDB Collection (Recommended)

A separate MongoDB collection is created for prescription records. Each document references the owning `Customer` and the recording `Staff` (Optometrist) by their respective IDs.

**Illustrative document shape (confirmed fields only):**

```json
{
  "_id": "<ObjectId>",
  "customerId": "<ObjectId — ref: Customer>",
  "recordedBy": "<ObjectId — ref: Staff (Optometrist)>",
  "recordedAt": "<ISODate>",
  "notes": "<string — general clinical notes, if approved>",
  "isArchived": false,
  "createdAt": "<ISODate>",
  "updatedAt": "<ISODate>"
}
```

> ⚠️ **Open Requirement — see Section 8:** Specific clinical fields (sphere, cylinder, axis, PD, lens type, and detailed clinical notes) are not defined in the approved SRS or SDS. These fields must be confirmed by stakeholders before implementation. The schema above intentionally omits them pending that confirmation.

**Advantages:**

- Each prescription is an independent document — full historical records are preserved naturally without overwriting.
- `customerId` reference satisfies AC3 (customer relationship).
- `recordedBy` reference satisfies AC4 (optometrist traceability).
- The collection can be independently queried, indexed, and secured at the authorization layer.
- Supports retention and archival policies independently of the `Customer` document (SDS Section 6.3).
- Audit metadata fields (`recordedAt`, `recordedBy`, `createdAt`, `updatedAt`) provide record-level traceability. Note that these fields alone do not constitute an immutable audit trail; separate immutable logging or version history must be implemented if required by the SDS.
- Independent authorization middleware can restrict collection access to approved roles without modifying the `Customer` model.

**Disadvantages:**

- Requires an additional collection and additional query (or population) when loading a customer profile.
- Slightly more complex backend service compared to embedding.

---

### Option B — Prescription Data Embedded Within the `Customer` Document

Prescription records are stored as an array field directly inside each `Customer` document.

**Illustrative shape:**

```json
{
  "_id": "<ObjectId>",
  "name": "...",
  "email": "...",
  "prescriptions": [
    {
      "recordedBy": "<ObjectId — ref: Staff>",
      "recordedAt": "<ISODate>",
      "notes": "..."
    }
  ]
}
```

**Advantages:**

- No separate collection; prescription data is co-located with the customer record.
- Single document fetch returns the customer and all their prescriptions together.

**Disadvantages:**

- The `Customer` document grows unboundedly as prescriptions accumulate over time, which violates MongoDB's recommended document-size discipline for high-frequency append patterns.
- Authorization becomes more complex: the backend must retrieve the full `Customer` document before applying role-based field-level access control to the embedded prescription array.
- Audit trails and archival are harder to manage independently; archiving a prescription means mutating the `Customer` document.
- Clinical data (sensitive medical information) is co-located with general profile data (PII), creating a larger blast radius if a data exposure occurs.
- Indexing on prescription fields (e.g., date range queries across all customers) is more expensive with embedded arrays.

---

## 6. Decision

**Selected: Option A — Dedicated `Prescription` MongoDB Collection.**

A new `Prescription` collection will be introduced into the OSMS data model as an extension to the approved SDS schema (Section 2.2). This is a documented addition to the schema, not a silent deviation.

The collection will hold one document per prescription record, referencing the owning `Customer` and the responsible `Staff` (Optometrist) by ObjectId.

New prescription records are **always inserted** (never overwrite an existing record) to preserve the full historical prescription trail required by FR-002.

---

## 7. Rationale

| Requirement | How Option A satisfies it |
|-------------|--------------------------|
| FR-002: customers view prescription history | Query all `Prescription` docs where `customerId` matches the authenticated customer — returns full history in chronological order. |
| FR-013: Optometrist records clinical data | `POST /prescriptions` endpoint, RBAC-restricted to Optometrist role; `recordedBy` field captures the responsible staff member. |
| SDS Section 1.2.1: support detailed clinical prescription histories | Dedicated collection with append-only insert pattern preserves the full history. |
| SDS Section 6.2: RBAC — Optometrist creates/modifies, Customer views own data | Independent collection allows a focused authorization middleware that does not interfere with other customer data. |
| SDS Section 6.3: audit trails, 7-year retention | `recordedAt`, `recordedBy`, `createdAt`, `updatedAt` fields provide record-level traceability (`isArchived` flag supports soft archival). Immutable audit logging or version history must be implemented separately if required by the SDS. |
| SDS Section 6.3: sensitive data separated from general PII | Prescription collection is physically separate from `Customer` — reduces exposure surface. |
| AC5: history must not be overwritten | Insert-only pattern; no `PUT` that replaces the previous record. |

The dedicated collection approach is the standard MongoDB pattern for one-to-many relationships where the "many" side (prescriptions) grows over time and requires independent access control and lifecycle management.

---

## 8. Consequences

### Positive

- FR-002 and FR-013 can be implemented with a clear, traceable data model.
- Historical prescriptions are preserved and queryable without risk of overwrite.
- Access control can be enforced at the collection/endpoint level, consistent with the SDS RBAC model.
- Auditing and retention requirements can be satisfied without coupling prescription lifecycle to the `Customer` document lifecycle.

### Negative / Trade-offs

- The `Prescription` collection is an **addition to the approved SDS schema** — this ADR is the traceability record for that addition.
- When a customer profile is loaded with prescription history, a secondary query (or Mongoose population) is needed. This is standard practice and acceptable given the access-control benefit.

### Access Control Summary

Access control for clinical prescription data is governed by the baseline permissions explicitly defined in the SDS (Section 6.2):

- **Authorized Optometrist / Medical Staff**: Can record and modify clinical prescription data (FR-013).
- **Customers**: Can view their own prescription history (FR-002).
- **Other Roles**: Detailed role permissions for all other system roles (e.g., Admin, Branch Manager, Management) must follow the authoritative RBAC ADR and are not defined within this storage design ADR.

> **Security Control Rule:** Backend authorization middleware must be the primary security control enforcing role-based access on all prescription endpoints. Frontend UI visibility controls alone must not be treated as an authorization mechanism.

### Security and Privacy

- All prescription endpoints require a valid JWT (authentication before authorization).
- Customer-scoped read access is enforced by matching `customerId` against the authenticated user's token — customers cannot query another customer's prescriptions.
- Prescription data must never appear in backend logs.
- Prescription documents must never be returned in any API response that is not explicitly scoped to an authorized role or the owning customer.
- Input validation is required on all fields before persistence (SDS Section 6.4).

### Data Retention

- Prescription records must not be hard-deleted during normal operation.
- The `isArchived` flag supports soft-archival to meet the SDS 7-year retention requirement (SDS Section 6.3) without removing records from the database.
- Archival and purge policies (when records may be permanently removed after the retention period) are outside the scope of this ADR and must be defined separately if required.

---

## 9. Open Requirements — Stakeholder Confirmation Required

> ⚠️ The following items are **not defined** in the approved SRS or SDS. Per the copilot-instructions (Section 12 — Clinical prescription data), these fields must **not** be implemented without explicit stakeholder confirmation. They are recorded here so the implementation ticket is not blocked by an undocumented assumption.

| Item | Question |
|------|----------|
| **Clinical fields** | Which specific fields are required on a prescription record? (e.g., sphere, cylinder, axis, near PD, far PD, lens type, frame type, add power, prism, clinical notes) |
| **Left/Right eye data** | Are fields recorded separately per eye, or as a single combined record? |
| **Prescription validity / expiry** | Does a prescription have an expiry date that the system must track? |
| **Amendment vs new record** | If an Optometrist corrects an error on an existing prescription, is that a new record or an amendment to the existing one? If amendment, how is the original preserved for audit? |
| **Prescription linked to appointment** | Should a `Prescription` record reference the `Appointment` at which it was recorded? |
| **Customer-initiated prescription upload** | FR-002 mentions prescription history — does this include the ability for a customer to upload an externally issued prescription, or only Optometrist-recorded prescriptions? |

**Implementation of FR-013 must not proceed until the clinical fields question is resolved.**  
The remaining questions should be clarified before or during the FR-013 implementation sprint.

---

## 10. SRS / SDS References

| Reference | Relevance |
|-----------|-----------|
| SRS FR-002 | Customer profile management and prescription history viewing |
| SRS FR-013 | Clinical prescription data recording by Optometrist |
| SDS Section 1.2.1 — Data Layer | MongoDB must support detailed clinical prescription histories |
| SDS Section 2.2 — Final Normalized Schema | Defines the approved baseline schema; `Prescription` entity is absent |
| SDS Section 6.2 — Authorization Levels | RBAC matrix: Optometrist owns FR-013; Customer views FR-002 |
| SDS Section 6.3 — Data Protection | Audit trails, 7-year retention, sensitive data isolation |
| SDS Section 6.4 — Input Validation | All input must be validated before persistence |
| Development Guidelines Section 1.3 | Significant SDS deviations must be documented via ADR |

---

## 11. Review Record

| Date | Reviewer | Role | Status | Notes |
|------|----------|------|--------|-------|
| — | — | — | Pending | — |

> Peer review is required before this ADR is moved to **Accepted** status and before the linked implementation issue (FR-013 backend) is started.
