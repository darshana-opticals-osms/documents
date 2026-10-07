# ADR-003: Shopping Cart Persistence and Checkout Boundary

## 1. Title

Shopping Cart Persistence and Checkout Boundary for OSMS

## 2. Status

**Proposed** — Team architecture decision agreed on 2026-10-05. Pending formal peer review and approval.

---

## 3. Context

The Optical Shop Management System (OSMS) requires shopping-cart and checkout functionality for online eyewear purchases under **FR-004**.

The approved SDS already includes:

- Cart Management
- Checkout and Order Confirmation
- Order persistence
- OrderItem persistence
- Inventory management
- Customer authentication and JWT-based identity

However, the approved database schema does not define a dedicated `Cart` entity or collection.

The current implementation already contains:

- Product/catalog functionality
- Customer authentication
- JWT-based identity
- Inventory model
- Order model
- OrderItem model
- Shared frontend API client

There is currently no implemented `Cart` model or cart persistence mechanism.

Therefore, a clear architectural decision is required before Sprint 2 cart and checkout implementation begins.

---

## 4. Design Gap

The SRS states that a secure shopping cart and checkout process must be provided.

The SDS defines the cart and checkout workflow, but the final normalized database schema contains entities such as:

- Customer
- Inventory
- Order
- OrderItem
- Payment

It does not define a `Cart` entity.

Therefore, the existing design does not clearly specify:

- where temporary cart data is stored,
- whether cart data is stored only in the frontend or also in the backend,
- when authentication becomes mandatory,
- how frontend cart state is synchronized,
- when temporary cart data becomes permanent Order and OrderItem data,
- what happens to cart data after successful or failed checkout.

---

## 5. Problem

Before implementing the shopping cart and checkout functionality, the team must define one authoritative architecture.

The system must answer the following questions:

1. Where should cart data exist before checkout?
2. Should a Cart collection be added to MongoDB?
3. When must the customer be authenticated?
4. How should frontend cart state and backend cart data remain consistent?
5. Which cart values can be trusted?
6. When does cart data become permanent Order and OrderItem data?
7. What happens when checkout fails?
8. What happens when the cart becomes empty?

Without one agreed decision, frontend and backend developers could implement incompatible cart approaches.

---

## 6. Considered Options

### Option A — Frontend / Browser-Only Cart

The cart is stored only in frontend state and/or browser storage such as `localStorage`.

The cart is submitted to the backend only when checkout begins.

#### Advantages

- Simple architecture.
- Fast user-interface updates.
- No new MongoDB Cart collection is required.
- Lower implementation effort.
- Closely follows the existing SDS database schema.

#### Disadvantages

- Cart persistence is limited to the current browser/device.
- Cross-device cart access is difficult.
- Backend has no persistent cart state before checkout.
- Cart recovery depends on browser storage.

---

### Option B — Backend-Persisted Cart

A new `Cart` model and `carts` collection are added to the existing OSMS MongoDB database.

All cart operations are persisted through backend APIs.

#### Advantages

- Cart survives browser refresh and device changes.
- Backend maintains one persistent authoritative cart.
- Logged-in customers can potentially access the same cart from different devices.
- Easier server-side recovery of cart state.

#### Disadvantages

- Requires a new database entity not currently defined in the approved SDS schema.
- Requires additional cart APIs.
- More backend and database operations are required for cart changes.
- User-interface interactions may depend more heavily on network requests.

---

### Option C — Hybrid Authenticated Cart

The frontend maintains cart state for responsive user experience while cart data is also synchronized with a backend `Cart` collection.

Only authenticated `CUSTOMER` users may add products to the cart.

The backend persists the customer's cart using the authenticated identity from the JWT.

#### Advantages

- Fast frontend cart interaction.
- Persistent backend cart storage.
- Cart can be recovered after refresh or later login.
- Backend can maintain persistent customer-owned cart data.
- Provides a clear path from cart to checkout.

#### Disadvantages

- More complex than frontend-only storage.
- Frontend and backend cart state must be synchronized carefully.
- Requires new Cart APIs and a Cart database model.
- Requires handling synchronization failures safely.

---

### Option Comparison

| Criterion | Option A — Frontend / Browser-Only | Option B — Backend-Persisted | Option C — Hybrid Authenticated |
|---|---|---|---|
| Complexity | Low | Medium | High |
| SDS Compatibility | High — no new Cart entity | Medium — introduces a new Cart entity | Medium — introduces a new Cart entity and synchronization |
| Security | Backend still validates checkout, but cart exists only on client | Strong server-side ownership for authenticated carts | Strong server-side ownership with responsive frontend state |
| Customer Experience | Fast, but limited persistence | Persistent, but cart actions depend more on network requests | Fast frontend interaction with persistent backend recovery |
| Persistence | Browser/device only | Backend database | Frontend state plus backend database |
| Implementation Effort | Low | Medium | High |
| Maintainability | Simple | Moderate | More synchronization logic to maintain |

Option C has the highest implementation complexity, but it was selected because the team requires both responsive frontend cart behaviour and persistent customer-owned backend cart data.

---

## 7. Decision

**Selected: Option C — Hybrid Authenticated Cart Architecture.**

OSMS will use a hybrid cart architecture.

Only an authenticated user with the canonical role `CUSTOMER` may add, remove, or update products in a cart.

The frontend will maintain cart state to provide immediate and responsive user-interface updates.

The backend will persist the cart in a new `Cart` model / `carts` collection inside the existing OSMS MongoDB database.

The authenticated customer identity must be obtained from the verified JWT and must not be trusted from a client-supplied `customerId`.

The backend-persisted cart is the persistent cart record.

The frontend cart state must remain synchronized with backend-confirmed cart data.

---

## 8. Rationale

The hybrid approach was selected because the team requires both:

- a fast and responsive cart experience in the frontend, and
- persistent cart data in the backend.

A frontend-only cart would be simpler but would provide weaker persistence and would make recovery across sessions or devices more difficult.

A backend-only cart would provide strong persistence but would make every cart interaction more dependent on network and backend response time.

The hybrid approach provides responsive frontend behaviour while maintaining a persistent customer-owned cart in MongoDB.

Authentication is required before adding items because the selected architecture associates every persisted Cart directly with an authenticated Customer.

This avoids introducing anonymous Cart ownership, temporary guest identifiers, or guest-cart merge rules that are not currently required by the approved SRS or SDS.

---

## 9. Consequences

### Positive Consequences

- Cart data is persistently associated with the authenticated Customer.
- Frontend interactions can remain responsive.
- Cart state can be recovered after refresh.
- The backend has a clear persistent source for cart data.
- Checkout can consume a backend-associated cart.
- Customer ownership can be enforced using JWT identity.

### Negative Consequences

- A new `Cart` model and MongoDB collection must be implemented.
- Cart APIs must be created.
- Frontend/backend synchronization must be handled carefully.
- More implementation and testing effort is required than with a frontend-only cart.
- Network or synchronization failures must be handled without misleading the user.

---

## 10. Authentication and Security Boundary

Browsing the product catalogue may remain publicly accessible where already supported.

However, cart mutation requires authentication.

A user must be authenticated as a `CUSTOMER` before performing actions such as:

- Add to Cart
- Remove from Cart
- Update quantity
- Retrieve persistent cart
- Proceed to checkout

The frontend must not submit a customer identifier as authoritative ownership information.

The backend must derive the customer identity from the verified JWT.

Conceptually:

`JWT -> authenticated userId -> customer Cart`

The backend must also not trust client-supplied:

- authoritative product price,
- available stock quantity,
- final order total,
- customer identity.

These values must be validated or derived from authoritative backend/database data.

---

## 11. Cart Data and Synchronization Boundary

The frontend maintains temporary cart state for responsive user interaction.

Cart-changing operations must be synchronized with the backend.

The backend-persisted Cart represents the persistent state associated with the authenticated Customer.

### Minimum Persisted Cart Contract

The persisted Cart must contain only the minimum information required to represent the Customer's intended purchase before checkout.

At minimum, the persistence contract must include:

- Customer ownership derived from the authenticated JWT,
- the referenced product/inventory item,
- the Customer's requested quantity.

The client must not control the authoritative Customer identity.

The persisted Cart must not treat the following as authoritative checkout values:

- product price,
- current stock availability,
- calculated line totals,
- calculated order total.

These values must be retrieved, validated, or recalculated from authoritative backend/database data during checkout.

Detailed Mongoose schema design, indexes, and implementation-specific field names remain the responsibility of the dedicated Cart/backend implementation issue.

After a successful backend cart mutation, the frontend should update itself using backend-confirmed cart data.

If synchronization fails, the frontend must not falsely indicate that the change has been successfully persisted.

---

## 12. Empty Cart Behaviour

An empty cart must not be stored as an unnecessary Cart document.

The selected rule is:

- If the Customer has one or more cart items, a Cart document may exist.
- If the final item is removed, the Cart document must be deleted.
- Therefore, a Customer with an empty cart has no active Cart document in MongoDB.

This avoids storing unnecessary empty cart records.

---

## 13. Checkout and Order Boundary

Cart data is temporary shopping state.

`Order` and `OrderItem` represent confirmed transactional data.

When checkout begins, the backend must:

1. Verify the authenticated Customer.
2. Retrieve or validate the Customer's cart.
3. Validate that each referenced product/inventory item still exists.
4. Retrieve authoritative current prices.
5. Validate available stock.
6. Calculate the authoritative final order total.
7. Create the Order.
8. Create the related OrderItem records.
9. Confirm successful order creation.
10. Clear/delete the Cart only after successful checkout.

The frontend must not create or treat an Order as confirmed before the backend confirms successful persistence.

### Checkout Consistency / Atomicity Boundary

Checkout persistence must behave as one consistent all-or-nothing business operation.

The required checkout changes include, where applicable:

- creation of the `Order`,
- creation of all related `OrderItem` records,
- inventory changes required to confirm the Order,
- removal of the temporary Cart after successful checkout.

The system must not expose a confirmed Order if only part of the required checkout persistence has succeeded.

If any required checkout persistence step fails, the backend must use an appropriate transaction, rollback, or equivalent compensation mechanism so that:

- no partially confirmed Order remains,
- Order and OrderItem data remain consistent,
- inventory does not remain partially updated,
- the Cart is not silently lost.

The exact MongoDB transaction or compensation implementation is outside this ADR and must be defined in the dedicated Sprint 2 checkout/backend implementation issue.

---

## 14. Inventory Interaction Boundary

The checkout/order service is responsible for coordinating with the inventory layer before confirming an Order.

Before final Order creation, the backend must verify that requested quantities are still available.

The Cart itself does not reserve stock.

Adding an item to the Cart must therefore not be interpreted as guaranteeing inventory availability at checkout time.

Detailed inventory-decrement and stock-synchronization behaviour remains the responsibility of the relevant Sprint 2 inventory and checkout implementation issues.

---

## 15. Payment Boundary

The approved SDS includes Payment functionality, but this ADR is limited to shopping-cart persistence and the transition from temporary Cart data to Order and OrderItem data.

This ADR does not define the required sequencing between:

- payment authorization or payment completion,
- Order persistence,
- Order confirmation.

The approved requirements reviewed for DDP-036 do not provide enough information to select that business rule safely.

Therefore, payment sequencing must be confirmed through the appropriate stakeholder/design decision and implemented through the dedicated payment and checkout issues.

Until that decision is confirmed, implementation must not independently assume that:

- an Order is confirmed before payment,
- payment must always complete before Order creation,
- or Order creation itself proves successful payment.

The atomic checkout boundary defined by this ADR applies to the cart-to-order persistence responsibilities defined here. Payment-specific consistency requirements must follow the separately approved payment architecture.

---

## 16. Failure Behaviour

Checkout may fail because of conditions such as:

- authentication failure,
- invalid or removed products,
- changed product prices,
- insufficient stock,
- invalid quantities,
- server or database errors.

If checkout fails:

- no invalid or partially confirmed Order should be produced,
- partial Order, OrderItem, and inventory changes must not remain as a successful checkout state,
- the frontend must receive a clear failure response,
- the Cart must not be silently deleted,
- the previously persisted Cart should remain available unless an explicitly valid cart update is required,
- rollback, transaction, or equivalent compensation behaviour must preserve the consistency boundary defined in Section 13.

The frontend must not display a successful checkout until the backend confirms successful Order creation.

---

## 17. Post-Checkout Behaviour

After successful Order and OrderItem creation:

- the confirmed Order becomes the permanent transactional record,
- the temporary Cart is no longer required,
- the backend Cart document must be deleted,
- the frontend cart state must be cleared.

The Cart must only be cleared after the backend confirms that checkout completed successfully.

If checkout fails, the Cart remains available so the Customer can correct the issue and retry.

---

## 18. SRS and SDS References

### SRS

- **FR-004** — Shopping Cart and Checkout
- Customer online shopping workflow
- Relevant customer authentication requirements

### SDS

- Section 2.2 — Final Normalized Database Schema
- Cart Management sequence
- Checkout and Order Confirmation sequence
- Cart Management user flow
- Checkout and Order Confirmation user flow
- `Order` entity
- `OrderItem` entity
- Inventory design
- Authentication and JWT design
- Input validation and security design

---

## 19. Implementation Boundary

This ADR defines architecture only.

DDP-036 must not implement:

- React Cart components,
- Cart APIs,
- Cart MongoDB model,
- checkout API,
- Order creation API,
- inventory decrement logic,
- payment integration.

Payment sequencing and payment lifecycle rules are also outside the decision scope of DDP-036 and require the separately approved payment architecture or stakeholder clarification.

Those changes must be completed in their dedicated Sprint 2 implementation issues after this ADR is Accepted.

---

## 20. Review Record

| Date | Reviewer / Group | Result | Notes |
|---|---|---|---|
| 2026-10-05 | Development Team | Agreed | Hybrid authenticated cart architecture selected. Only authenticated CUSTOMER users may use cart functionality. Empty carts are not persisted. |
| Pending | Peer Reviewer | Pending | Formal pull-request review required before ADR status changes to Accepted. |
| — | — | Supervisor | Pending | — |

---

## 21. Decision Summary

OSMS will implement a **hybrid authenticated shopping cart**.

- Cart functionality requires an authenticated `CUSTOMER`.
- Frontend cart state provides responsive user experience.
- Cart data is persisted in a backend MongoDB `carts` collection.
- Customer ownership is derived from the JWT.
- Empty carts are not persisted.
- Backend authoritative product prices and stock are revalidated at checkout.
- Successful checkout creates `Order` and `OrderItem` records.
- Cart is deleted only after successful checkout.
- Failed checkout preserves the Cart.
- Persisted Cart ownership comes from the authenticated JWT and Cart items persist only the required product/inventory reference and requested quantity.
- Checkout persistence must behave as an all-or-nothing operation; partial Order, OrderItem, inventory, or Cart state must not be exposed as successful checkout.
- Payment sequencing is not decided by this ADR and requires the separately approved payment architecture / stakeholder clarification.
