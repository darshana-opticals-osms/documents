# ADR-002: Authoritative RBAC Role Model

## 1. Title

Authoritative Role-Based Access Control (RBAC) Role Model for OSMS

## 2. Status

**Accepted** — Approved through peer review on 2026-09-09.

---

## 3. Context

The Optical Shop Management System (OSMS) uses Role-Based Access Control (RBAC) to control access to customer, clinical, branch operational, branch managerial, and administrative functionality.

The approved SRS and SDS both defined role-related requirements. Originally:

The SRS identified the following system stakeholders:
- System Administrator
- Branch Manager
- Optometrist / Medical Staff
- Inventory Manager
- Sales Assistant / Cashier
- Customer
- Darshana Optical Management

The SDS Section 6.2 originally defined six RBAC authorization roles:
- System Administrator
- Inventory Manager
- Branch Manager
- Optometrist / Medical Staff
- Management / Owner
- Customer

To eliminate overlapping responsibilities, simplify authorization governance, and align with the real-world operational structure of Darshana Opticals, the project retired `MANAGEMENT`, `INVENTORY_MANAGER`, and `SALES_ASSISTANT_CASHIER`. Their responsibilities have been reassigned:
- Inventory management, stock alerts, reorder thresholds, business oversight, and reporting are consolidated under `BRANCH_MANAGER`.
- Operational front-desk, counter assistance, and order lifecycle duties are assigned to the dedicated `BRANCH_ASSISTANT` role.

This establishes an authoritative 5-role model across the system.

---

## 4. Problem

Previously, having separate `MANAGEMENT`, `INVENTORY_MANAGER`, and `SALES_ASSISTANT_CASHIER` roles introduced unnecessary architectural complexity, role fragmentation, and divergence between specifications and operational needs:
- `INVENTORY_MANAGER` and `MANAGEMENT` responsibilities overlapped significantly with the Branch Manager's operational and oversight scope.
- `SALES_ASSISTANT_CASHIER` had no dedicated functional requirements in the SRS and was absent from the SDS authorization matrix.

Without a consolidated and authoritative canonical role set, individual developers risked defining conflicting role values in:
- MongoDB user/staff schemas
- JWT authentication
- Backend RBAC middleware
- Frontend route protection
- Navigation visibility
- Development seed data
- Automated authorization tests

The project therefore requires one authoritative list of five canonical roles and clear ownership of system responsibilities.

---

## 5. Current SRS Role Model

The SRS stakeholder model identifies the following relevant roles:

| SRS Role | Main Responsibility / Interest |
|---|---|
| System Administrator | User-account management, system configuration, and data security |
| Branch Manager | Oversight of branch operations, sales reports, staff performance, and inventory levels |
| Optometrist / Medical Staff | Eye-examination scheduling and prescription history recording/access |
| Inventory Manager | Stock tracking, purchase orders, and stock transfers |
| Sales Assistant / Cashier | Daily sales transactions, customer inquiries, and point-of-sale loyalty activities |
| Customer | Online shopping, appointment booking, and order tracking |
| Darshana Optical Management | Business analytics and strategic decision-making |

The SRS therefore treats `Sales Assistant / Cashier` as a distinct internal stakeholder.

---

## 6. Current SDS Role Model

The SDS Section 6.2 defines six main RBAC roles:

| SDS Role | Main Responsibility |
|---|---|
| System Administrator | System integrity, authentication, registration, and configuration |
| Inventory Manager | Stock management and reorder alerts |
| Branch Manager | Branch operations, product catalog management, and appointment reminders |
| Optometrist / Medical Staff | Appointment management and clinical prescription recording |
| Management / Owner | Payment oversight, loyalty monitoring, and business reports |
| Customer | Customer-owned shopping, profile, chatbot, and order-tracking functionality |

The SDS does not define `Sales Assistant / Cashier` as a separate RBAC role.

This difference is considered a design ambiguity that must be resolved before the production authorization model is finalized.

---

## 7. Considered Options

### Option A — Create a Dedicated Sales Assistant / Cashier Role

Introduce `Sales Assistant / Cashier` as an independent RBAC role.

The role would receive only the permissions required for approved operational sales-assistance responsibilities.

**Advantages:**

- Preserves the stakeholder role explicitly identified in the SRS.
- Supports the principle of least privilege.
- Avoids giving Sales Assistants unnecessary Branch Manager or Management permissions.
- Allows future sales-assistant functionality to be protected independently.
- Makes staff responsibilities clearer in authorization tests and audit records.

**Disadvantages:**

- Adds one role that is not currently present in the SDS Section 6.2 authorization table.
- Requires the SDS authorization model to be updated.
- Exact Sales Assistant permissions must remain limited to approved requirements and must not be invented during implementation.

---

### Option B — Map Sales Assistant / Cashier to Branch Manager

Sales Assistant / Cashier accounts would internally use the `BRANCH_MANAGER` authorization role.

**Advantages:**

- No additional RBAC role would be required.
- Maintains the existing number of SDS authorization roles.

**Disadvantages:**

- A Sales Assistant could inherit permissions intended for Branch Managers.
- This could include product-catalog management, branch-level visibility, and other permissions unnecessary for cashier duties.
- Violates the least-privilege principle.
- Removes the distinction explicitly made by the SRS stakeholder model.

---

### Option C — Exclude Sales Assistant / Cashier from the Implemented RBAC Model

The system would implement only the six roles currently defined by the SDS.

**Advantages:**

- Follows the SDS role list without modification.
- Simplifies the initial RBAC implementation.

**Disadvantages:**

- The SRS-defined Sales Assistant / Cashier stakeholder would have no corresponding system authorization identity.
- Operational staff requirements could later require an unplanned role-model change.
- The difference between the SRS and implementation would remain unresolved.

---

## 8. Decision

**Selected: Consolidated 5-Role RBAC Model with `BRANCH_ASSISTANT` and Consolidated `BRANCH_MANAGER` Responsibilities.**

The authoritative OSMS RBAC model will contain five application roles:
1. `SYSTEM_ADMIN`
2. `BRANCH_MANAGER`
3. `OPTOMETRIST`
4. `BRANCH_ASSISTANT`
5. `CUSTOMER`

Key decisions and reassignments:
- `INVENTORY_MANAGER` and `MANAGEMENT` roles are retired from canonical authorization.
  - Stock management, inventory adjustments, and reorder alert threshold configuration are assigned to `BRANCH_MANAGER`.
  - Payment/business oversight, sales reports, and inventory reporting are consolidated under `BRANCH_MANAGER`.
- `SALES_ASSISTANT_CASHIER` is replaced by the canonical role `BRANCH_ASSISTANT`, representing frontline operational staff at the branch (handling customer operational inquiries, sales assistance, order lifecycle progression, and operational appointment scheduling coordination).
- Principle of least privilege is strictly enforced: `BRANCH_ASSISTANT` does not inherit managerial override, product catalog maintenance, or administrative configuration rights.

---

## 9. Canonical Roles

The following values are the authoritative programmatic role names for OSMS:

| Business Role | Canonical Programmatic Value |
|---|---|
| System Administrator | `SYSTEM_ADMIN` |
| Branch Manager | `BRANCH_MANAGER` |
| Optometrist / Medical Staff | `OPTOMETRIST` |
| Branch Assistant | `BRANCH_ASSISTANT` |
| Customer | `CUSTOMER` |

These values must be used consistently by:

- Backend user/staff models
- Authentication services
- JWT role claims
- RBAC middleware
- Development seed data
- Automated tests
- Frontend protected routes
- Frontend role-based navigation

Alternative strings such as `admin`, `Admin`, `manager`, `cashier`, `doctor`, or retired legacy roles (`INVENTORY_MANAGER`, `MANAGEMENT`, `SALES_ASSISTANT_CASHIER`) must not be independently introduced as authorization-role values.

Human-readable labels displayed in the user interface may use friendly names such as "System Administrator", "Branch Manager", or "Branch Assistant", while authorization logic must use the canonical values defined above.

---

## 10. Permission Mapping

The following mapping defines the main ownership of protected OSMS functional areas.

| Functional Area | Primary Role(s) | Important Restrictions |
|---|---|---|
| User and account administration | `SYSTEM_ADMIN` | Privileged accounts must not be created through unrestricted public registration |
| Customer profile and customer-owned functions | `CUSTOMER` | Customers may access only their own protected personal data |
| Product/catalog maintenance | `BRANCH_MANAGER` | Other roles may receive read access only where allowed by the SDS or public catalog design |
| Inventory management & reorder alert thresholds | `BRANCH_MANAGER` | Handles stock-management operations, inventory adjustments, and threshold configurations previously assigned to `INVENTORY_MANAGER` |
| Inventory/report viewing & business oversight | `BRANCH_MANAGER` | Payment/business oversight and sales/inventory reporting previously assigned to `MANAGEMENT` are consolidated under `BRANCH_MANAGER` |
| Appointment management & doctor sessions | `BRANCH_ASSISTANT`, with `BRANCH_MANAGER` operational oversight | Staff appointment coordination and doctor session management per ADR-007; customers use self-service booking |
| Clinical prescription recording | `OPTOMETRIST` | Customers may view their own approved prescription history but cannot create clinical records |
| Branch operational & sales-assistance duties | `BRANCH_ASSISTANT` | Frontline sales assistance, order status progression, and customer counter support; no administrative, clinical, or catalog/pricing override access |
| Shopping, checkout, chatbot, and order tracking | `CUSTOMER` | Customer access remains scoped to customer-facing functionality |

### Branch Assistant Limitation

The `BRANCH_ASSISTANT` role handles daily branch operational duties, customer inquiries, order processing, and frontline assistance.

Therefore:

- The role is included in the RBAC model as the canonical operational staff role.
- It must not inherit Branch Manager permissions (such as product-catalog editing, user provisioning, or managerial price overrides).
- It must not manually bypass the automatic loyalty-point behaviour defined by FR-010.
- Any operational functionality must follow approved requirements and least privilege.

---

## 11. Account Provisioning Rules

### Public Registration

Public self-registration must create only:

`CUSTOMER`

A public user must not be allowed to choose or submit a privileged role such as:

- `SYSTEM_ADMIN`
- `BRANCH_MANAGER`
- `OPTOMETRIST`
- `BRANCH_ASSISTANT`

The backend must assign or enforce the `CUSTOMER` role for public registration rather than trusting a role value supplied by the client.

### Staff Account Provisioning

Privileged staff accounts must be provisioned through a controlled administrative process.

The selected approach is:

> **System Administrator creates or provisions staff accounts and assigns an approved canonical role.**

This is consistent with the SRS and SDS responsibility assigned to the System Administrator for account management, registration, authentication, and system security.

Staff role assignment must be performed only by authorized administrative functionality.

Development seed accounts may be created through controlled development/test setup, but seed credentials and privileged development behaviour must never be treated as a production account-provisioning mechanism.

---

## 12. Authentication and JWT Impact

The SDS specifies JWT-based authentication and requires the authenticated user's identity and role to be represented in the JWT.

OSMS JWT authentication must therefore include a canonical role value.

Illustrative authenticated role representation:

```json
{
  "userId": "<authenticated-user-id>",
  "role": "CUSTOMER"
}
```

The exact user-identifier claim name may follow the backend authentication implementation, but the `role` value must use one of the canonical role values defined by this ADR.

The backend must:

1. Validate the JWT before authorization.
2. Read the authenticated user's canonical role.
3. Compare the role against the roles permitted for the requested operation.
4. Reject unauthorized access using the appropriate HTTP response.

The frontend may use the authenticated role to determine navigation and protected-route behaviour, but hiding a button or page in the frontend is **not** a substitute for backend authorization.

Backend RBAC remains the authoritative security control.

---

## 13. Implementation Consistency

After this ADR is accepted:

### Backend

Backend identity models and staff schemas must use only the canonical roles defined in this ADR (`SYSTEM_ADMIN`, `BRANCH_MANAGER`, `OPTOMETRIST`, `BRANCH_ASSISTANT`, `CUSTOMER`).

Backend RBAC middleware must use the same values and must not create another independent role list with different names.

### Authentication

JWT generation and authenticated-user context must use the canonical role representation.

### Frontend

Frontend authentication, navigation, and protected-route logic must use the same canonical values.

The frontend must not independently introduce alternative authorization roles.

### Development Seed Data

Development seed mechanisms must create users only with roles approved by this ADR.

### Testing

Authorization tests must use the canonical roles.

Tests should verify that:

- Allowed roles can access permitted operations.
- Unauthorized roles receive access-denied responses.
- Invalid arbitrary role values are rejected.
- Public registration cannot create privileged users.
- Branch Assistant does not inherit unrelated Branch Manager permissions.

---

## 14. Consequences

### Positive

- One authoritative, streamlined RBAC role model with five canonical roles is established.
- Role redundancy is eliminated by merging inventory and managerial oversight into `BRANCH_MANAGER`.
- Operational branch staff are clearly represented by `BRANCH_ASSISTANT`.
- Backend, frontend, authentication, seed data, and tests use identical role definitions.
- Least privilege is preserved without granting frontline staff managerial powers.
- Public registration cannot be used to create privileged accounts.

### Negative / Trade-offs

- The SDS authorization model and SRS stakeholder mapping must be updated to reflect the 5-role structure.
- Legacy references to retired roles (`INVENTORY_MANAGER`, `MANAGEMENT`, `SALES_ASSISTANT_CASHIER`) across existing ADRs, tests, and documentation must be updated or aligned.

### Security Impact

The decision strengthens authorization consistency because roles are centrally defined and privileged account creation is controlled.

No authorization decision may rely only on frontend visibility.

Backend authorization checks must be applied to all protected operations.

---

## 15. Remaining Clarification

The authoritative role **identity** is resolved by this ADR.

The frontline operational staff responsibilities are consolidated under `BRANCH_ASSISTANT`.

If implementation requires new capabilities beyond the approved requirements, such as a dedicated point-of-sale workflow or manual loyalty adjustments, those capabilities must be clarified and approved through the project requirement/design-change process before implementation.

---

## 16. SRS / SDS / Guideline References

| Reference | Relevance |
|---|---|
| SRS Section 7 — Stakeholders | Defines project stakeholders; streamlined to System Administrator, Branch Manager, Optometrist, Branch Assistant, and Customer |
| SRS FR-001 | Customer and staff registration/authentication using secure credentials |
| SRS FR-002 | Customer profile and prescription-history functionality |
| SRS FR-003 | Product catalog management |
| SRS FR-005 / FR-006 | Inventory management and reorder alerts |
| SRS FR-007 / FR-008 | Appointment booking and reminders |
| SRS FR-009 / FR-010 / FR-012 | Payment, loyalty, and management-reporting responsibilities |
| SRS FR-013 | Clinical prescription recording |
| SDS Section 6.1 — Authentication Mechanism | Defines JWT authentication and role representation |
| SDS Section 6.2 — Authorization Levels | Defines the RBAC model and access-control matrix |
| Development Guideline Section 1.3 — Project Continuity | Significant SDS deviations must be documented using an ADR and reflected in the final SDS |
| DDP-033 | Requires one authoritative OSMS RBAC role model |

---

## 17. Review Record

| Date | Reviewer | Role | Status | Notes |
|---|---|---|---|---|
| 2026-09-09 | thiruniimasha | Team Member | Approved | Peer review completed; no blocking issues. |
| 2026-10-12 | Development Team | Approved | Consolidated role model: retired `MANAGEMENT`, `INVENTORY_MANAGER`, and `SALES_ASSISTANT_CASHIER`. Reassigned to `BRANCH_MANAGER` and introduced canonical `BRANCH_ASSISTANT`. |
| — | — | Supervisor | Pending | — |

> The final SDS must be updated to reflect the authoritative RBAC role model as required by the project change-management process.
