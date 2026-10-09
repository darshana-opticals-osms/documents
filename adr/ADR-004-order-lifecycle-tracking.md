# ADR-004: Order Lifecycle and Tracking Status Model

## 1. Title

Order Lifecycle and Tracking Status Model for OSMS

## 2. Status

**Proposed**

This ADR must be reviewed and approved before its status is changed to `Accepted`.

---

## 3. Context

The Optical Shop Management System (OSMS) requires real-time Order tracking under **FR-014**.

The current backend already contains:

- an `Order` model,
- an `OrderItem` model,
- Customer authentication,
- Product / Catalog functionality,
- Inventory functionality.

The current `Order` model stores:

- `customerId`,
- `orderDate`,
- `orderAmount`,
- standard Mongoose timestamps.

The current model does not yet contain an authoritative Order status or Order-status history.

The existing `OrderItem` model remains responsible for individual line items belonging to an Order.

ADR-003 defines the shopping-cart and checkout boundary and establishes that successful checkout creates persisted `Order` and `OrderItem` records.

However, ADR-003 intentionally does not define payment sequencing.

Payment lifecycle and Order/Payment coordination are handled separately by **DDP-054 – Secure Payment Gateway Integration and Payment Lifecycle**.

Therefore, this ADR defines the operational Order-tracking lifecycle without independently defining Payment states.

---

## 4. Existing Design Gap

FR-014 requires Customers to track their Orders, but the current design does not define:

- canonical Order status values,
- the initial Order status,
- valid status transitions,
- terminal states,
- which role may update Order status,
- Customer Order ownership rules,
- staff Order-tracking visibility,
- status timestamps and history,
- invalid transition behaviour.

Without one authoritative lifecycle, different developers could introduce incompatible values such as:

- `pending`,
- `processing`,
- `shipped`,
- `completed`,
- `delivered`,
- `cancelled`.

Therefore, one lifecycle model must be agreed before Sprint 2 Order and tracking APIs are implemented.

---

## 5. Problem

OSMS requires an Order lifecycle that is:

- simple enough for the current project scope,
- sufficient for Customer Order tracking,
- consistent across backend, frontend, tests, and Swagger,
- protected by the authoritative RBAC model,
- traceable when an Order progresses,
- resistant to arbitrary or invalid transitions.

The lifecycle must also remain separate from the Payment lifecycle so that Order state and Payment state do not become contradictory sources of truth.

---

## 6. Lifecycle Scope

This ADR defines the **operational lifecycle of a persisted Order**.

Payment-related values such as:

- `pending`,
- `processing`,
- `completed`,
- `failed`,
- `cancelled`,
- `refunded`

belong to the Payment lifecycle defined through DDP-054.

Therefore, this ADR does not introduce Order statuses such as:

- `PAYMENT_PENDING`,
- `PAID`,
- `PAYMENT_FAILED`,
- `REFUNDED`.

The Order statuses defined here represent only the operational progress of the Order.

---

## 7. Proposed Canonical Order Status Values

The following canonical status set is proposed for OSMS:

| Canonical Value    | Human-Readable Meaning | Lifecycle Meaning                                                                                                           |
| ------------------ | ---------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `ORDER_CREATED`    | Order Created          | The Order has been successfully persisted by the backend.                                                                   |
| `PROCESSING`       | Processing             | The shop is currently processing or preparing the Order.                                                                    |
| `READY_FOR_PICKUP` | Ready for Pickup       | Shop-side preparation is complete and the Order is ready to be handed over to the external delivery / collection party.     |
| `PICKED_UP`        | Picked Up              | The external delivery / collection party has collected the Order from the shop and the Order is on its way to the Customer. |
| `COMPLETED`        | Completed              | The Order has been successfully delivered or handed over to the final Customer.                                             |
| `CANCELLED`        | Cancelled              | The Order has been cancelled and will not continue through the normal lifecycle.                                            |

These values are intentionally limited to the currently required OSMS tracking workflow.

Statuses such as:

- `SHIPPED`,
- `OUT_FOR_DELIVERY`,
- `DELIVERED`,
- Payment-specific statuses

are not added as separate canonical values because the proposed lifecycle already represents the required stages without unnecessary duplication.

This canonical set must be confirmed during review before this ADR becomes `Accepted`.

---

## 8. Initial Order Status

### Proposed Decision

When a new Order is successfully persisted by the backend, its initial tracking status will be:

`ORDER_CREATED`

`ORDER_CREATED` means only that the Order record has been successfully created.

It does **not** automatically mean that:

- Payment has succeeded,
- Payment is complete,
- the Order is being prepared,
- the Order is ready for collection,
- the Customer has received the Order.

Payment sequencing remains the responsibility of DDP-054.

---

## 9. Allowed Status Transitions

### Proposed Normal Lifecycle

```text
ORDER_CREATED
      |
      v
PROCESSING
      |
      v
READY_FOR_PICKUP
      |
      v
PICKED_UP
      |
      v
COMPLETED
```

The meaning of the later stages is:

```text
READY_FOR_PICKUP
      |
      | External delivery / collection party
      | collects the Order from the shop
      v
PICKED_UP
      |
      | Final Customer receives the Order
      v
COMPLETED
```

Cancellation is proposed from the early lifecycle stages:

```text
ORDER_CREATED -----> CANCELLED

PROCESSING --------> CANCELLED
```

The proposed valid transitions are:

| Current Status     | Allowed Next Status             |
| ------------------ | ------------------------------- |
| `ORDER_CREATED`    | `PROCESSING`, `CANCELLED`       |
| `PROCESSING`       | `READY_FOR_PICKUP`, `CANCELLED` |
| `READY_FOR_PICKUP` | `PICKED_UP`                     |
| `PICKED_UP`        | `COMPLETED`                     |
| `COMPLETED`        | No further transition           |
| `CANCELLED`        | No further transition           |

`COMPLETED` and `CANCELLED` are terminal states.

A terminal Order must not return to an earlier lifecycle state.

The backend must prevent arbitrary lifecycle skipping such as:

```text
ORDER_CREATED -> COMPLETED
```

or:

```text
PROCESSING -> PICKED_UP
```

unless a future approved business requirement changes this lifecycle.

The cancellation rules remain subject to team / stakeholder review before this ADR becomes `Accepted`.

---

## 10. Invalid Transition Behaviour

Every requested Order-status change must be validated against the authoritative transition rules.

If a transition is invalid:

- the backend must reject the operation,
- the existing Order status must remain unchanged,
- no false status-history entry must be created,
- the frontend must receive an appropriate error response.

Unknown or unsupported status values must also be rejected.

The backend must not silently accept or normalize arbitrary client-supplied status values.

Exact HTTP status codes and response schemas will be defined by the backend implementation issue and documented in Swagger.

---

## 11. Status Update Authorization

Status-update authorization must follow **ADR-002 – Authoritative RBAC Role Model**.

Customers must have read-only Order-tracking access and must not directly update operational Order status.

The currently reviewed requirements do not explicitly identify which staff role is responsible for all operational Order-status updates.

### Proposed Decision Requiring Review

`BRANCH_MANAGER` is proposed as the role responsible for operational Order-status updates because the Branch Manager is responsible for approved branch operational activities.

Under this proposal:

- `BRANCH_MANAGER` may update Order status,
- `CUSTOMER` may view only their own tracking information and cannot update status,
- `INVENTORY_MANAGER` may receive tracking visibility where permitted by the approved access-control model,
- read permission does not automatically provide update permission,
- no additional role receives Order-status update permission by assumption.

This authorization decision must be explicitly confirmed before this ADR becomes `Accepted`.

If future implementation requires an external delivery partner to report collection or delivery events, that integration and its authorization mechanism must be defined separately rather than granting a new OSMS role by assumption.

---

## 12. Customer Tracking and Ownership

An authenticated Customer may access only Orders owned by that Customer.

The backend must:

- derive Customer identity from the authenticated backend context,
- verify Order ownership server-side,
- prevent a Customer from retrieving another Customer's Order,
- never treat a client-supplied Customer ID as proof of ownership.

A Customer must not gain access to another Customer's Order by modifying an Order ID or Customer ID in a request.

Frontend route protection alone is not sufficient.

Backend authorization remains authoritative.

---

## 13. Staff Viewing Scope

Staff Order-tracking visibility must follow:

- the approved SRS/SDS access-control model,
- ADR-002.

Where the approved access-control rules allow `INVENTORY_MANAGER` or `BRANCH_MANAGER` to view Order-tracking information, the backend may provide that authorized read access.

This ADR does not independently expand staff permissions.

Read access must not automatically grant Order-status update permission.

---

## 14. Status History Options Considered

### Option A – Current Status Only

Store only:

- current Order status,
- latest status-update timestamp.

Conceptually:

```text
Order
 ├── status
 └── statusUpdatedAt
```

#### Advantages

- simplest implementation,
- minimum storage,
- simple current-status lookup.

#### Disadvantages

- previous status changes are lost,
- weak traceability,
- limited troubleshooting and reporting support.

---

### Option B – Embedded Status History

Store:

- current status,
- latest status-update timestamp,
- ordered status-history entries inside the Order document.

Conceptually:

```text
Order
 ├── status
 ├── statusUpdatedAt
 └── statusHistory[]
       ├── status
       ├── changedAt
       └── changedBy
```

#### Advantages

- current status remains simple to query,
- previous transitions remain traceable,
- history naturally belongs to the Order,
- no additional collection is required,
- suitable for the expected small number of lifecycle changes,
- supports Customer tracking and troubleshooting.

#### Disadvantages

- Order documents become slightly larger,
- update logic must ensure current status and history remain consistent.

---

### Option C – Separate Order Status History Collection

Store current status on `Order`, while each status change is stored in a separate collection.

Conceptually:

```text
Order
 └── status

OrderStatusHistory
 ├── orderId
 ├── status
 ├── changedAt
 └── changedBy
```

#### Advantages

- strong separation of status events,
- suitable for large histories,
- useful for complex auditing or analytics.

#### Disadvantages

- requires another collection and model,
- requires additional queries,
- adds implementation complexity,
- unnecessary for the current expected OSMS Order lifecycle.

---

## 15. Tracking History Decision

### Proposed Selection: Option B – Embedded Status History

OSMS will use:

- the current status directly on the `Order`,
- the latest status-change timestamp,
- an embedded ordered status-history structure.

Conceptually:

```text
status
statusUpdatedAt

statusHistory:
  - status
  - changedAt
  - changedBy
```

This provides a suitable balance between:

- simplicity,
- traceability,
- query complexity,
- implementation effort,
- auditability,
- maintainability.

A normal OSMS Order is expected to have only a small number of lifecycle transitions, so a separate history collection would add unnecessary complexity.

The exact Mongoose schema implementation belongs to the future backend implementation issue.

This ADR does not modify the database model directly.

---

## 16. Status History Rules

When a valid status transition succeeds:

1. the current Order status must be updated,
2. `statusUpdatedAt` must represent the successful change time,
3. a corresponding status-history entry must be appended,
4. history must preserve chronological lifecycle progression.

For example:

```text
ORDER_CREATED
PROCESSING
READY_FOR_PICKUP
PICKED_UP
COMPLETED
```

If a transition fails or is invalid:

- the current status must remain unchanged,
- no false history entry must be added.

Where `changedBy` is stored, it must be derived from authenticated backend identity.

The backend must not trust a client-supplied user or staff identifier as authoritative status-change identity.

---

## 17. API Responsibility Boundary

Sprint 2 Order and tracking APIs must provide the following high-level responsibilities.

### Order Creation

The backend must:

- use the authenticated Customer identity,
- perform checkout validation required by ADR-003,
- persist the Order,
- persist related OrderItem records,
- assign the approved initial Order status,
- initialize the approved status-history representation.

### Customer Order List Retrieval

An authenticated Customer may retrieve only Orders owned by that Customer.

### Single Order Retrieval

The backend must:

- retrieve the requested Order,
- verify Customer ownership or approved staff authorization,
- return approved Order and tracking information.

### Status Retrieval

Tracking responses must use the canonical status values defined by this ADR.

Frontend code may display human-readable labels, but it must not create a separate or incompatible Order lifecycle.

### Authorized Status Update

The backend must:

- authenticate the caller,
- authorize the approved update role,
- validate the requested canonical status,
- verify that the transition from the current status is legal,
- update the current status,
- update the status timestamp,
- append the corresponding history entry.

Exact endpoint paths, validation schemas, HTTP response codes, and response structures will be defined by the backend implementation issue and Swagger documentation.

---

## 18. Relationship with Checkout and Payment

ADR-003 defines the cart-to-Order checkout boundary.

Payment lifecycle and Order/Payment coordination are defined separately by:

**DDP-054 – Secure Payment Gateway Integration and Payment Lifecycle**

Therefore, this ADR defines only:

- canonical operational Order statuses,
- valid Order-status transitions,
- Order tracking and history rules.

This ADR does not independently define:

- Payment lifecycle states,
- when a Payment record is created,
- which Payment event confirms payment success,
- failed-Payment impact on the Order,
- Payment-cancellation impact on the Order,
- retry behaviour,
- refund behaviour.

DDP-054 must define:

- Payment lifecycle states,
- failed-Payment impact on the Order,
- Payment-cancellation impact on the Order,
- which verified Payment events cause an Order lifecycle transition,
- the distinction between Order creation, Payment initiation, Payment completion, and final paid-order confirmation.

Any Payment-to-Order transition defined by DDP-054 must remain compatible with the canonical Order statuses and allowed transitions defined by this ADR.

Payment status and Order status must remain separate authoritative domain states and must not contradict each other.

---

## 19. Existing Order and OrderItem Responsibility

The existing model responsibilities remain preserved.

### Order

`Order` remains the order-level record responsible for information such as:

- Customer ownership,
- Order date,
- Order amount,
- future approved tracking status,
- future approved status history.

### OrderItem

`OrderItem` remains responsible for individual line-item information belonging to an Order.

Order-status history must not duplicate OrderItem information unnecessarily.

---

## 20. External Delivery / Collection Boundary

The proposed lifecycle distinguishes between:

```text
READY_FOR_PICKUP
```

and:

```text
PICKED_UP
```

`READY_FOR_PICKUP` means shop-side preparation has been completed.

`PICKED_UP` means the external delivery / collection party has physically collected the Order from the shop.

`COMPLETED` means the final Customer has received the Order.

Therefore:

```text
READY_FOR_PICKUP != PICKED_UP
PICKED_UP != COMPLETED
```

This distinction provides clearer Customer tracking visibility.

This ADR does not define a separate external-delivery user role, delivery application, courier API, or third-party tracking integration.

If such integration is required later, it must be approved through its own implementation or architecture decision.

---

## 21. Security Implications

Order tracking is protected functionality.

The backend must enforce:

- JWT authentication,
- authoritative RBAC,
- Customer ownership verification,
- canonical status validation,
- valid transition validation,
- server-controlled status timestamps,
- server-controlled status-change identity.

The client must not be trusted to provide authoritative:

- Customer ownership,
- staff identity,
- status timestamps,
- transition history.

Frontend access controls may support the user experience but must not replace backend authorization.

---

## 22. Consequences

### Positive

- Backend, frontend, tests, and Swagger can use one authoritative Order lifecycle.
- Customers receive clear tracking information.
- The difference between shop preparation, external-party collection, and final delivery is explicit.
- Invalid status jumps are prevented.
- Terminal Orders cannot accidentally return to active processing states.
- Embedded status history improves traceability.
- Existing `Order` and `OrderItem` responsibilities remain clear.
- Payment state remains separate from operational Order tracking.

### Negative / Trade-offs

- The `Order` model will require a future schema extension.
- Every status update requires transition validation.
- Embedded history slightly increases the size of each Order document.
- `PICKED_UP` introduces an additional lifecycle transition that must be supported consistently.
- The proposed status-update role requires explicit approval.
- Future delivery-partner integrations may require another architecture decision.
- Future payment requirements may require coordination with this lifecycle through DDP-054.

---

## 23. SRS / SDS / ADR References

| Reference                                                          | Relevance                                                                     |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| FR-004 – Shopping Cart and Checkout                                | Defines checkout and Order creation context                                   |
| FR-014 – Real-Time Order Tracking Status                           | Requires Customer-facing Order tracking                                       |
| SDS Section 2.2 – `Order` and `OrderItem`                          | Defines existing persisted Order structures                                   |
| SDS Checkout and Order Confirmation flow                           | Defines checkout-to-Order context                                             |
| SDS Section 6.2 – Authorization Levels / RBAC                      | Defines protected Customer and staff access                                   |
| ADR-002 – Authoritative RBAC Role Model                            | Defines canonical OSMS roles and authorization principles                     |
| ADR-003 – Shopping Cart Persistence and Checkout Boundary          | Defines cart-to-Order responsibility and leaves Payment sequencing unresolved |
| DDP-054 – Secure Payment Gateway Integration and Payment Lifecycle | Defines Payment lifecycle and Payment-to-Order status coordination            |
| Existing backend `Order` model                                     | Existing Order-level persistence                                              |
| Existing backend `OrderItem` model                                 | Existing line-item persistence                                                |

---

## 24. Implementation Boundary

This ADR defines architecture and lifecycle rules only.

This issue must not implement:

- Order APIs,
- Order-status controllers,
- Order-status services,
- frontend Order tracking screens,
- Mongoose model changes,
- database migrations,
- Payment logic,
- inventory-decrement logic,
- courier integrations,
- external delivery APIs,
- unrelated backend or frontend functionality.

Those changes belong to dedicated Sprint 2 implementation issues after this ADR has been reviewed and accepted.

---

## 25. Review Decisions Required

Before this ADR becomes `Accepted`, reviewers must explicitly confirm:

1. the canonical status set,
2. `ORDER_CREATED` as the initial status,
3. `READY_FOR_PICKUP -> PICKED_UP -> COMPLETED` as the external-delivery tracking flow,
4. the allowed transition rules,
5. the proposed cancellation rules,
6. whether `BRANCH_MANAGER` is the correct status-update role,
7. embedded status history as the selected history strategy,
8. separation between Order lifecycle and DDP-054 Payment lifecycle.

Until these decisions are approved, dependent implementation must not independently introduce alternative statuses, transitions, or permissions.

---

## 26. Review Record

| Date       | Reviewer / Group         | Result                 | Notes                                                                                                                                                 |
| ---------- | ------------------------ | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-10-09 | Development Team         | Proposed               | Initial ADR prepared for review under DDP-037 / Issue #14. `PICKED_UP` included to distinguish external-party collection from final Customer receipt. |
| —          | Peer Reviewer            | Pending                | —                                                                                                                                                     |
| —          | Supervisor / Stakeholder | Pending where required | —                                                                                                                                                     |

---

## 27. Decision Summary

The proposed OSMS Order-tracking lifecycle is:

```text
ORDER_CREATED
      |
      v
PROCESSING
      |
      v
READY_FOR_PICKUP
      |
      v
PICKED_UP
      |
      v
COMPLETED
```

Cancellation is proposed from:

```text
ORDER_CREATED -> CANCELLED
PROCESSING    -> CANCELLED
```

The canonical statuses are:

- `ORDER_CREATED`
- `PROCESSING`
- `READY_FOR_PICKUP`
- `PICKED_UP`
- `COMPLETED`
- `CANCELLED`

The operational meaning is:

```text
ORDER_CREATED
Order exists in the system.

PROCESSING
The shop is preparing the Order.

READY_FOR_PICKUP
Shop-side preparation is complete and the Order is ready for the external delivery / collection party.

PICKED_UP
The external delivery / collection party has collected the Order from the shop.

COMPLETED
The final Customer has received the Order.

CANCELLED
The Order has been cancelled.
```

`COMPLETED` and `CANCELLED` are terminal states.

Customers have read-only tracking access to their own Orders.

Staff viewing follows ADR-002 and the approved SDS access-control model.

`BRANCH_MANAGER` is proposed as the operational Order-status update role pending review.

Embedded status history is proposed to preserve lifecycle traceability.

Payment-specific lifecycle decisions remain under DDP-054 and must coordinate with, rather than redefine, this Order lifecycle.

This ADR remains **Proposed** until the lifecycle, authorization, cancellation, history, and Order/Payment coordination decisions are reviewed and approved.
