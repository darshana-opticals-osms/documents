# ADR-002: Authoritative RBAC Role Model

## 1. Title

Authoritative Role-Based Access Control (RBAC) Role Model for OSMS

## 2. Status

**Proposed** — Pending peer review and team approval.

> Once approved by the team, update this status to **Accepted** and record the approval date in the Review Record.

---

## 3. Context

The Optical Shop Management System (OSMS) uses Role-Based Access Control (RBAC) to control access to customer, staff, inventory, clinical, management, and administrative functionality.

The approved SRS and SDS both define role-related requirements. However, the role sets are not fully consistent.

The SRS identifies the following relevant system stakeholders:

- System Administrator
- Branch Manager
- Optometrist / Medical Staff
- Inventory Manager
- Sales Assistant / Cashier
- Customer
- Darshana Optical Management

The SDS Section 6.2 defines the following RBAC authorization roles:

- System Administrator
- Inventory Manager
- Branch Manager
- Optometrist / Medical Staff
- Management / Owner
- Customer

The SDS therefore does **not** define `Sales Assistant / Cashier` as a separate authorization role even though it is explicitly identified as an internal stakeholder in the SRS.

The SRS describes the Sales Assistant / Cashier as being involved in daily sales transactions, customer inquiries, and point-of-sale loyalty activities.

A consistent role model is required before authentication, authorization, frontend protected routes, development seed data, and role-based automated tests are finalized.

---

## 4. Problem

The project currently contains two different interpretations of the OSMS role model.

The main ambiguity is:

> **Sales Assistant / Cashier exists in the approved SRS stakeholder model but does not exist as a separate role in the SDS RBAC model.**

If this difference is not resolved centrally, individual developers may independently define different role values in:

- MongoDB user/staff schemas
- JWT authentication
- Backend RBAC middleware
- Frontend route protection
- Navigation visibility
- Development seed data
- Automated authorization tests

This could result in inconsistent authorization behaviour and security defects.

The project therefore requires one authoritative list of canonical roles and one consistent interpretation of their responsibilities.

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

**Selected: Option A — Create a dedicated Sales Assistant / Cashier RBAC role.**

The authoritative OSMS RBAC model will contain seven application roles.

This ADR documents the addition of the Sales Assistant / Cashier role to the SDS authorization model.

The role is retained because it is explicitly defined as an internal stakeholder in the SRS and represents responsibilities that are different from those of Branch Manager and Management.

The role must follow the principle of least privilege.

The addition of this role does **not** automatically introduce new functional requirements. Any Sales Assistant functionality that is not already supported by the approved SRS must be clarified and approved before implementation.

---

## 9. Canonical Roles

The following values are the authoritative programmatic role names for OSMS:

| Business Role | Canonical Programmatic Value |
|---|---|
| System Administrator | `SYSTEM_ADMIN` |
| Inventory Manager | `INVENTORY_MANAGER` |
| Branch Manager | `BRANCH_MANAGER` |
| Optometrist / Medical Staff | `OPTOMETRIST` |
| Management / Owner | `MANAGEMENT` |
| Sales Assistant / Cashier | `SALES_ASSISTANT_CASHIER` |
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

Alternative strings such as `admin`, `Admin`, `manager`, `cashier`, or `doctor` must not be independently introduced as authorization-role values.

Human-readable labels displayed in the user interface may use friendly names such as "System Administrator" or "Sales Assistant / Cashier", while authorization logic must use the canonical values defined above.

---

## 10. Permission Mapping

The following mapping defines the main ownership of protected OSMS functional areas.

| Functional Area | Primary Role(s) | Important Restrictions |
|---|---|---|
| User and account administration | `SYSTEM_ADMIN` | Privileged accounts must not be created through unrestricted public registration |
| Customer profile and customer-owned functions | `CUSTOMER` | Customers may access only their own protected personal data |
| Product/catalog maintenance | `BRANCH_MANAGER` | Other roles may receive read access only where allowed by the SDS or public catalog design |
| Inventory management | `INVENTORY_MANAGER` | Handles approved stock-management operations. Inventory adjustments and manual price changes requiring managerial override are restricted to `BRANCH_MANAGER` or `SYSTEM_ADMIN` as defined by the SRS business rules |
| Inventory/report viewing | `BRANCH_MANAGER`, `MANAGEMENT` where permitted | Read-only access must not imply inventory modification rights |
| Appointment management | `OPTOMETRIST`, with `BRANCH_MANAGER` responsible for approved reminder operations | Customers may use the booking functionality but do not receive staff management permissions |
| Clinical prescription recording | `OPTOMETRIST` | Customers may view their own approved prescription history but cannot create clinical records |
| Payment/business oversight | `MANAGEMENT` | Management access must not automatically grant System Administrator privileges |
| Sales and inventory reporting | `MANAGEMENT` | Other roles receive only the read access explicitly allowed by the approved design |
| Operational sales-assistance duties | `SALES_ASSISTANT_CASHIER` | Limited to approved sales-assistance responsibilities; no automatic admin, management, clinical, or inventory-modification access |
| Shopping, checkout, chatbot, and order tracking | `CUSTOMER` | Customer access remains scoped to customer-facing functionality |

### Sales Assistant / Cashier Limitation

The SRS describes the Sales Assistant / Cashier as handling daily sales transactions, customer inquiries, and point-of-sale loyalty activities.

However, the current Functional Requirements Register does not assign a separate FR directly to the Sales Assistant / Cashier.

Therefore:

- The role is included in the RBAC model.
- It must not inherit all Branch Manager or Management permissions.
- It must not manually bypass the automatic loyalty-point behaviour defined by FR-010.
- Any new point-of-sale functionality or permission not clearly supported by the approved SRS must be handled through the appropriate requirement/design-change process before implementation.

---

## 11. Account Provisioning Rules

### Public Registration

Public self-registration must create only:

`CUSTOMER`

A public user must not be allowed to choose or submit a privileged role such as:

- `SYSTEM_ADMIN`
- `INVENTORY_MANAGER`
- `BRANCH_MANAGER`
- `OPTOMETRIST`
- `MANAGEMENT`
- `SALES_ASSISTANT_CASHIER`

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

Backend #4 and related identity models must use only the canonical roles defined in this ADR.

Backend #7 RBAC middleware must use the same values and must not create another independent role list with different names.

### Authentication

JWT generation and authenticated-user context must use the canonical role representation.

### Frontend

Frontend authentication, navigation, and protected-route logic must use the same canonical values.

The frontend must not independently introduce alternative authorization roles.

### Development Seed Data

Backend #19 and other development seed mechanisms must create users only with roles approved by this ADR.

### Testing

Authorization tests must use the canonical roles.

Tests should verify that:

- Allowed roles can access permitted operations.
- Unauthorized roles receive access-denied responses.
- Invalid arbitrary role values are rejected.
- Public registration cannot create privileged users.
- Sales Assistant / Cashier does not inherit unrelated Branch Manager or Management permissions.

---

## 14. Consequences

### Positive

- One authoritative RBAC role model is established.
- The SRS/SDS ambiguity is explicitly documented rather than silently handled in code.
- Backend, frontend, authentication, seed data, and tests can use the same role definitions.
- Sales Assistant / Cashier remains represented as required by the SRS stakeholder model.
- Least privilege can be applied without granting Sales Assistants excessive management permissions.
- Public registration cannot be used to create privileged accounts.

### Negative / Trade-offs

- The SDS authorization model must be updated because it currently defines only six main RBAC roles.
- Backend and frontend implementations must coordinate canonical role values.
- Adding another role increases the number of authorization scenarios that must be tested.
- Exact Sales Assistant / Cashier operational permissions may require additional clarification where the existing Functional Requirements Register does not define a corresponding function.

### Security Impact

The decision strengthens authorization consistency because roles are centrally defined and privileged account creation is controlled.

No authorization decision may rely only on frontend visibility.

Backend authorization checks must be applied to all protected operations.

---

## 15. Remaining Clarification

The authoritative role **identity** is resolved by this ADR, but the project must not invent new Sales Assistant / Cashier business functionality.

The SRS stakeholder description identifies operational Sales Assistant duties, while the current Functional Requirements Register does not provide a dedicated Sales Assistant-owned functional requirement.

If implementation requires new capabilities beyond the approved requirements, such as a dedicated point-of-sale workflow, manual loyalty adjustment, or additional customer-account access, those capabilities must be clarified and approved through the project requirement/design-change process before implementation.

This clarification does not prevent the canonical `SALES_ASSISTANT_CASHIER` role from being defined and used in the RBAC model.

---

## 16. SRS / SDS / Guideline References

| Reference | Relevance |
|---|---|
| SRS Section 7 — Stakeholders | Defines System Administrator, Branch Manager, Optometrist / Medical Staff, Inventory Manager, Sales Assistant / Cashier, Customer, and Management stakeholders |
| SRS FR-001 | Customer and staff registration/authentication using secure credentials |
| SRS FR-002 | Customer profile and prescription-history functionality |
| SRS FR-003 | Product catalog management |
| SRS FR-005 / FR-006 | Inventory management and reorder alerts |
| SRS FR-007 / FR-008 | Appointment booking and reminders |
| SRS FR-009 / FR-010 / FR-012 | Payment, loyalty, and management-reporting responsibilities |
| SRS FR-013 | Clinical prescription recording |
| SDS Section 6.1 — Authentication Mechanism | Defines JWT authentication and role representation |
| SDS Section 6.2 — Authorization Levels | Defines the current six-role RBAC model and access-control matrix |
| Development Guideline Section 1.3 — Project Continuity | Significant SDS deviations must be documented using an ADR and reflected in the final SDS |
| DDP-033 | Requires one authoritative OSMS RBAC role model |

---

## 17. Review Record

| Date | Reviewer | Role | Status | Notes |
|---|---|---|---|---|
| — | — | Team Member | Pending | — |
| — | — | Supervisor | Pending | — |

> At least one peer review is required before this ADR is considered approved. After approval, update the ADR status from **Proposed** to **Accepted**, record the approval details, and ensure the final SDS is updated to reflect the authoritative RBAC role model.
