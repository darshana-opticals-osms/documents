# ADR-004: Order Lifecycle and Tracking Status Model

## 1. Title

Order Lifecycle and Tracking Status Model for OSMS

## 2. Status

**Proposed**

This ADR requires team / stakeholder review before its status is changed to `Accepted`.

---

## 3. Context

The Optical Shop Management System (OSMS) requires real-time order tracking under **FR-014**.

The current backend already contains:

- an `Order` model,
- an `OrderItem` model,
- Customer authentication,
- Product / Catalog functionality,
- Inventory functionality.

The existing `Order` model currently stores:

- `customerId`,
- `orderDate`,
- `orderAmount`,
- standard Mongoose timestamps.

It does not currently contain an authoritative order status or order-status history.

The existing `OrderItem` model remains responsible for individual line items belonging to an Order.

ADR-003 defines the shopping-cart and checkout boundary and establishes that successful checkout creates persisted `Order` and `OrderItem` records.

However, ADR-003 intentionally does **not** define payment sequencing. It does not determine whether payment must occur before Order persistence or after Order persistence.

Therefore, this ADR defines the order-tracking lifecycle without assuming a payment lifecycle.

---

## 4. Existing Design Gap

FR-014 requires Customers to track the status of their Orders, but the current implementation does not define:

- canonical order status values,
- the initial status of a new Order,
- valid status transitions,
- final states,
- which staff role may update an Order status,
- how Customers access their own Order tracking information,
- whether status history is persisted,
- how invalid transitions are rejected.

Without one authoritative decision, separate developers could introduce incompatible status values such as:

- `pending`,
- `processing`,
- `shipped`,
- `completed`,
- `delivered`,
- `cancelled`.

Order lifecycle rules must therefore be defined centrally before tracking APIs and frontend tracking functionality are implemented.

---

## 5. Problem

The system needs a lifecycle model that is:

- simple enough for the current OSMS scope,
- sufficient for Customer order tracking,
- consistent across backend, frontend, tests, and API documentation,
- protected by the authoritative RBAC model,
- traceable when a status changes,
- resistant to arbitrary or invalid transitions.

The lifecycle must also avoid introducing payment-related order states before payment sequencing is separately approved.

---

## 6. Considered Lifecycle Scope

The order lifecycle should represent the operational progress of a persisted Order.

Payment-specific values such as:

- `PAYMENT_PENDING`,
- `PAID`,
- `PAYMENT_FAILED`,
- `REFUNDED`

are not defined by this ADR.

ADR-003 leaves payment sequencing unresolved. Therefore, payment lifecycle decisions must be handled by the dedicated payment / checkout architecture.

The status values defined here describe only the operational Order lifecycle.

---

## 7. Proposed Canonical Order Status Values

The following minimal status set is proposed for OSMS:

| Canonical Value    | Human-Readable Meaning | Lifecycle Meaning                                                                   |
| ------------------ | ---------------------- | ----------------------------------------------------------------------------------- |
| `ORDER_CREATED`    | Order Created          | The Order has been successfully persisted by the backend after checkout processing. |
| `PROCESSING`       | Processing             | Staff are processing or preparing the Order.                                        |
| `READY_FOR_PICKUP` | Ready for Pickup       | The Order is prepared and ready for Customer collection.                            |
| `COMPLETED`        | Completed              | The Order lifecycle has been successfully completed.                                |
| `CANCELLED`        | Cancelled              | The Order has been cancelled and will not continue through the normal lifecycle.    |

These values are intentionally limited to the currently required order-tracking workflow.

Statuses such as:

- `SHIPPED`,
- `OUT_FOR_DELIVERY`,
- `DELIVERED`,
- payment-specific statuses

are not introduced because the currently reviewed requirements do not clearly require those workflows.

The team must approve this canonical set before this ADR becomes `Accepted`.

---

## 8. Initial Order Status

### Proposed Decision

When the backend successfully persists a new `Order` as part of the approved checkout flow, its initial tracking status will be:

`ORDER_CREATED`

This status means only that the Order record has been successfully created.

It does **not** mean:

- payment is successful,
- payment is complete,
- the Order is already being prepared,
- inventory processing is complete.

Payment sequencing remains outside this ADR.

---

## 9. Allowed Status Transitions

### Proposed Transition Model

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
COMPLETED
```
