# ADR-005: Loyalty Points Calculation and Persistence Model

## 1. Title

Loyalty Points Calculation and Persistence Model for OSMS

## 2. Status

**Accepted**

Approved through peer review and supervisor sign-off on 2026-10-05.

## 3. Context

The OSMS customer-retention scope includes a loyalty program as part of the retail and customer-relationship model. The approved SRS and SDS identify loyalty as a business capability tied to customer purchases and management oversight.

The current project state shows the following relevant constraints:

- The customer-retention requirement includes a loyalty program with point accrual and redemption behavior.
- The SRS defines the loyalty earning rule in the Business Rules section.
- The SDS documents the approval and monitoring responsibilities tied to management review.
- The RBAC model is governed by ADR-002.
- The current backend data model does not yet implement a loyalty balance/history model.
- On 2026-10-04, the team clarified the unresolved loyalty business rules and recorded the final business decisions below.

The design challenge is to define a loyalty model that can support:

- automatic point earning
- customer visibility of current balance
- auditable management review
- duplicate-award protection
- future redemption and reversal support
- robust traceability to sales and payment events

without making assumptions beyond the approved requirements.

## 4. Existing SDS/Data-Model Gap

The current backend state is limited and does not yet include any loyalty-specific persistence model.

Current Customer model:

- no loyalty balance field
- no loyalty history field
- no loyalty tier field

Current Order model:

- customerId
- orderDate
- orderAmount

Current OrderItem model:

- orderId
- itemName
- price
- quantity

Current Payment model:

- orderId
- amount
- status
- method
- providerTransactionId

Current Payment statuses include:

- pending
- processing
- completed
- failed
- cancelled
- refunded

The important data-model gap is that current order and order-item records do not include a reliable product/category/type reference that directly distinguishes loyalty-eligible frames/accessories from non-eligible purchase types.

This means the future checkout and order implementation must preserve or derive qualifying purchase information from authoritative backend product/order data. A client-provided eligible amount or qualifying spend value must not be trusted as the source of truth.

This ADR therefore documents the required architectural direction without modifying the current models.

## 5. Problem

The approved loyalty requirement establishes a business rule for point accrual and redemption, but it does not yet define the complete operational contract for how the system should persist, calculate, audit, and reverse loyalty points.

The design must therefore separate:

A. Source-confirmed customer rules
B. Architectural decisions proposed by this ADR
C. Unresolved stakeholder decisions that still require confirmation

The essential problem is to create a loyalty model that is:

- aligned with the approved SRS and SDS
- safe for customer access control
- auditable for management review
- capable of preventing double-award errors
- robust enough for future approved refund and redemption logic

without inventing unsupported business policy.

## 6. Confirmed Business Rules

The following rules were clarified during the team meeting on 2026-10-04 and are treated as the approved business decisions for this ADR while status remains Proposed.

### 6.1 Confirmed earning formula

The team confirmed the earning rule as:

> earnedPoints = eligibleSpend / 100

This is a decimal-based accumulation rule. Loyalty points support decimal values and must be represented to 2 decimal places.

Examples:

- LKR 100 eligible spend -> 1.00 point
- LKR 250 eligible spend -> 2.50 points
- LKR 320 eligible spend -> 3.20 points

Important precision:

- The previous floor-only design is not used.
- Points are not rounded down to integers.
- A value with more than two decimal places must be bounded deterministically to two decimal places using a defined decimal-rounding convention at the business/data-contract level.
- Binary floating-point arithmetic must not be treated as the authoritative financial calculation without explicit decimal-safe handling.

### 6.2 Confirmed redemption rule

SRS Business Rule 10.3 remains the source basis:

> "Redemption of loyalty points is allowed only when the customer's total point balance exceeds 500 points."

Important precision:

- the source says exceeds 500
- this ADR does not change that to 500 or more

The team also confirmed the redemption conversion rule:

- 10 loyalty points = LKR 1
- equivalent: 1 point = LKR 0.10

The backend is authoritative for any redemption monetary value calculation.

### 6.3 Confirmed purchase eligibility

The team clarified that the loyalty calculation applies to merchandise/product items purchased through an Order, while clinical eye-test / doctor examination fees do not earn points.

This means the design must distinguish:

- Product / merchandise value
- Clinical service / eye-test fee value

Confirmed eligible:

- merchandise/product items purchased through an Order

Confirmed excluded:

- clinical eye-test / doctor examination fees

The ADR must therefore preserve the original SRS exclusion for clinical eye-test fees and treat any future product-category decision as a separate implementation/data-contract matter when the Order domain has sufficient authoritative data.

### 6.4 Product and category boundary requirement

The current OrderItem model does not yet preserve sufficiently authoritative product/category/type information to distinguish all merchandise items from excluded clinical service lines.

This is therefore recorded as an implementation/data-contract requirement for the Order/Checkout domain: the backend must derive eligible spend from authoritative Order/payment/item data, not from a frontend-supplied eligible amount or earned-points value.

The future Order/Checkout domain must preserve authoritative information sufficient for the backend to distinguish:

- merchandise/product value
- excluded clinical eye-test / doctor-fee value

This may later be implemented through an authoritative item/service classification, eligibility marker, or equivalent domain metadata, but this ADR does not prescribe an exact field name such as isLoyaltyEligible unless a dedicated Order/Checkout ADR or implementation requirement explicitly approves that contract.

The important architectural requirement is that the backend can derive eligible spend from authoritative persisted Order/OrderItem/service data, rather than trusting a frontend-supplied eligibleAmount or points value.

### 6.5 Automatic loyalty behavior

The source states that FR-010 requires loyalty points to be calculated and updated automatically based on customer purchase value.

This ADR therefore treats loyalty accumulation as an automatic backend business function, not a manual or client-driven value.

### 6.6 Confirmed non-eligible categories

The source does not assign explicit loyalty eligibility to categories such as:

- contact lenses
- prescription lenses
- services
- other future categories

Those remain outside the confirmed scope unless a separate, approved domain decision states otherwise.

### 6.7 Team decision summary

The team clarified the following business rules on 2026-10-04:

- decimal reward values are allowed and must be stored/returned to 2 decimal places
- loyalty points are confirmed only after a successful Payment and an Order in the approved COMPLETED state
- a valid full refund reverses the previously earned loyalty points from that refunded eligible purchase
- a valid partial refund reverses points proportionally to the refunded eligible amount
- post-payment customer self-cancellation is not available; loyalty impact is decided by the refund/cancellation outcome
- arbitrary staff manual point adjustments are prohibited
- the tiered-loyalty program is out of current implementation scope

## 7. Considered Persistence Options

### Option A — Customer balance only

Summary:

A single current loyalty balance is stored directly on the Customer record.

Evaluation:

- Simplicity: high
- Auditability: weak
- Query performance: strong for current balance reads
- Reporting: weak for historical review
- Duplicate prevention: weak unless extra enforcement is added elsewhere
- Refund/reversal support: limited
- Redemption support: possible but not robust for history
- Consistency: relatively simple but poor for traceability
- Maintainability: moderate, but loses history
- Compatibility with current SDS schema: easy to add a field, but incomplete from a business-audit perspective

This option does not provide enough traceability to support audit and reversal decisions.

### Option B — Dedicated loyalty transaction/history collection only

Summary:

Only a transaction history is retained, and the current balance is derived from the event log.

Evaluation:

- Simplicity: moderate
- Auditability: strong
- Query performance: weaker for frequent customer balance reads unless specifically optimized
- Reporting: strong
- Duplicate prevention: strong if event uniqueness is enforced
- Refund/reversal support: strong because compensating entries can be added
- Redemption support: strong if negative transactions are represented
- Consistency: can be made consistent but requires careful aggregation logic
- Maintainability: moderate to high, depending on event-query complexity
- Compatibility with current SDS schema: needs a new collection/model, but conceptually fits the normalized model

This option is auditable but less efficient for frequent balance reads.

### Option C — Customer current balance + dedicated loyalty transaction/history

Summary:

The customer carries a current loyalty-balance field, while a dedicated loyalty event/history collection records the transactional details for review, tracing, reversals, and future redemption logic.

Evaluation:

- Simplicity: moderate
- Auditability: strong
- Query performance: strong for current balance reads
- Reporting: strong
- Duplicate prevention: strong when combined with enforced idempotency keys
- Refund/reversal support: strong because compensating transactions can be recorded
- Redemption support: strong, since negative deductions can be represented as transaction events
- Consistency: requires transactional discipline to keep balance and history aligned
- Maintainability: moderate, with extra operational complexity
- Compatibility with current SDS schema: requires extension but fits normalized transaction-model principles

This option best matches the expected operational needs for OSMS.

## 8. Selected Architectural Direction

This ADR proposes Option C:

**Customer balance + dedicated loyalty transaction/history**

This is a proposed architectural direction for review. It is not yet Accepted.

The proposed rationale is:

- fast balance retrieval for customer display
- auditable history for Management review
- traceability to source transaction details
- support for compensating reversals
- support for future approved redemption deductions
- enforceable duplicate-award protection

The trade-off is a consistency cost:

- the current balance and the transaction history must remain in sync
- therefore local writes should use transactional consistency where multiple related updates occur together

## 9. Proposed Persistence Model

At the architectural level, the design should include:

### 9.1 Customer balance field

Customer records may include a loyalty-balance field representing the current confirmed point total.

This field is intended for fast customer-facing reads and efficient management overview.

### 9.2 Dedicated loyalty transaction/history record

A dedicated loyalty event or transaction record should store the minimum required audit trail for each loyalty change.

The backend implementation must use a centralized controlled set of loyalty event/transaction types rather than arbitrary free-form strings. The semantic categories are architectural and not intended to be invented independently by each service or domain component.

The required semantic event categories currently include:

- purchase earning
- refund/reversal
- redemption deduction

These may conceptually correspond to backend-defined identifiers such as:

- EARN_PURCHASE
- REFUND_REVERSAL
- REDEMPTION_DEDUCTION

However, the exact identifier names are implementation-level details and are not frozen as a final enum in this ADR. The important architectural requirement is that the event types are centrally defined, controlled, and consistent across the loyalty domain.

Proposed concepts:

- customerId
- orderId
- paymentId where relevant
- transaction/event type
- points delta
- balance-after value or equivalent audit state
- idempotency/source reference
- created timestamp

This is a proposed event model for future design work.

Exact schema details, indexing, and Mongoose-type decisions are implementation-level follow-up and are intentionally not locked in this ADR.

No arbitrary manual-adjustment event is required because manual adjustment is prohibited by the clarified business rules.

## 10. Customer Ownership and Access

Every loyalty balance and loyalty-history record belongs to a Customer.

Customer access must be:

- authenticated
- own-data only
- read-only for approved loyalty information

The backend must derive or verify authenticated ownership before returning loyalty information. Client-supplied customer IDs must not be used to authorize cross-customer access.

Frontend visibility is not an authorization control.

This ADR adopts the own-data access principle required by the approved auth policy and by the design requirement that customer records are customer-owned.

## 11. Management Access

ADR-002 defines the Management / Owner role as the authorization basis for business oversight and reporting.

In this ADR, Management may:

- review approved loyalty information
- monitor loyalty updates
- inspect audit context for customer balance/history records

Management does not receive unrestricted point mutation authority by default.

The phrase “Owner” in the SDS FR-010 matrix does not automatically grant arbitrary manual editing rights.

## 12. Sales Assistant / Cashier Role

ADR-002 identifies the role:

- SALES_ASSISTANT_CASHIER

This role is relevant to operational point-of-sale loyalty activity.

However, this ADR does not grant that role any arbitrary manual loyalty-adjustment authority. The role must not:

- manually add or deduct points without explicit approval
- bypass automatic loyalty rules
- invent a new point-of-sale API or privilege outside the approved requirement set

Any manual-adjustment capability requires separate explicit approval.

## 13. Manual Adjustment Policy

The team clarified on 2026-10-04 that staff must not have arbitrary manual loyalty-point adjustment capability.

Therefore:

- no normal staff operation may manually add arbitrary points
- no normal staff operation may manually subtract arbitrary points
- no normal staff operation may manually overwrite the balance
- no normal staff operation may bypass the automatic calculation

This ADR distinguishes clearly between:

- automatic earning
- automatic redemption
- approved refund reversal
- compensating reversal history

and:

- arbitrary staff manual adjustment

The latter is prohibited and is not approved by this ADR or by the clarified business rules.

## 14. Award Trigger

The team clarified the award trigger on 2026-10-04.

Confirmed rule:

Loyalty points are confirmed only after both of the following are true:

1. Payment has succeeded
2. The related Order reaches the approved COMPLETED state

This means:

- successful payment alone does not immediately produce confirmed loyalty points
- order creation alone does not produce loyalty points
- failed payment does not produce confirmed points
- cancelled or unsuccessful payment does not produce confirmed points
- an Order not yet in the approved COMPLETED state does not produce confirmed points

The implementation must therefore be retry-safe and idempotent because the qualifying event may be observed more than once.

## 15. Failed Payment Behaviour

The ADR satisfies the intent of AC9:

A failed or non-qualifying payment must not create a confirmed loyalty award.

Therefore loyalty processing must reject or prevent confirmed earning when the authoritative payment or sale does not qualify.

The system must not award points for:

- failed transactions
- cancelled transactions
- unsuccessful payment states

If a previously awarded loyalty event is later affected by a refund or cancellation outcome, that is addressed under the refund-reversal rule below.

## 16. Refund / Cancellation Handling

The clarified team rule for refunds and cancellations is as follows.

### 16.1 Full refund

If a purchase that previously earned loyalty points receives a valid full refund, the loyalty points earned from that refunded eligible purchase must be reversed.

The reversal must be recorded as an auditable loyalty-history event. The original earning history is retained; the reversal is appended as a subsequent compensating event.

### 16.2 Partial refund

If a valid partial refund is performed, the loyalty points corresponding to the refunded eligible amount must be reversed proportionally.

Example conceptually:

- original eligible purchase: LKR 1,000 -> 10.00 points
- eligible refunded amount: LKR 300
- reversal: 3.00 points

The reversal must reflect only the refunded eligible portion and not the non-refunded portion.

### 16.3 Post-payment cancellation workflow

After payment, a Customer does not receive an online self-service ability to cancel the Order directly.

If a post-payment cancellation or refund situation arises:

- the Customer must contact staff
- the request follows the applicable business/refund policy
- some items may not qualify for refund
- only an approved refund/cancellation outcome affects loyalty points

If an item or amount is not refunded, no loyalty reversal is triggered merely because the Customer requested cancellation.

### 16.4 Recommendation on reversal mechanism

Recommendation only — not a final business-approval claim:

A compensating negative loyalty transaction is the preferred technical representation because it preserves audit history while reflecting the reversal. This is a recommendation only and does not replace the team-approved business rule.

## 17. Duplicate-Award Protection

The architecture requires that the same qualifying source transaction must never award loyalty points twice.

This is essential for:

- gateway retry callbacks
- API retries
- background worker retries
- duplicate payment notifications

The design must use a persisted idempotency/uniqueness strategy tied to the authoritative source event.

Conceptually:

- source transaction + loyalty event type = one award identity

The architecture must not rely only on a simple application-side check before insert, because concurrent reprocessing can still cause duplicate inserts.

The requirement is:

- an enforceable persistence-level uniqueness/idempotency mechanism must protect the award event

This ADR does not define the exact storage/index implementation details in this document.

## 18. Order / Payment Traceability

The current authoritative relationship is:

- Order -> Customer
- Payment -> Order

A loyalty transaction should therefore be traceable to the sale lifecycle, especially when the award event is tied to a successful payment.

Proposed traceability model:

- customerId
- orderId
- paymentId where the event is payment-related

This avoids unnecessary duplication of PII and keeps the loyalty event anchored to the authoritative sales and payment records.

Client-derived customer identity must not be treated as the source of truth.

## 19. Backend Calculation Ownership

The authoritative calculation must occur in backend business logic.

The backend determines:

- qualifying spend
- earned points
- resulting balance changes

The frontend may display the returned values, but it must not authoritatively submit:

- earnedPoints
- qualifyingSpend
- loyaltyBalance
- arbitrary adjustment values

This preserves backend control and keeps rewards consistent with approved business logic.

## 20. Consistency and Atomicity

This ADR distinguishes two different concerns.

### A. External gateway processing

An external payment gateway and MongoDB cannot share a single ACID transaction.

Therefore gateway retries, callback duplication, and external-provider failure modes must be handled idempotently at the local application layer.

### B. Local persistence consistency

If multiple local updates are made as part of one loyalty action, for example:

- Customer balance update
- loyalty history/event insert

then the local persistence must be atomic.

Architectural requirement:

Either both local changes commit, or neither commits.

A MongoDB transaction is the proposed mechanism for local write consistency when the selected model includes both balance and history.

This ADR does not implement the transaction logic; it defines the required architectural outcome.

The design must also address:

- request retry
- callback replay
- process crash
- partial local write

## 21. Redemption

The confirmed rule is:

> Redemption of loyalty points is allowed only when the customer's total point balance exceeds 500 points.

The team also confirmed the redemption conversion:

- 10 loyalty points = LKR 1
- 1 point = LKR 0.10

The following remain out of scope unless explicitly approved:

- expiry policy
- redemption cap
- tier-specific discounts or offers
- additional promotional redemption rules

If future redemption implementation deducts points, the proposed Option C model supports auditable negative transactions by storing the deduction as a transaction/event with an opposite signed delta.

## 22. Tiered Loyalty Program Scope

The SRS high-level customer-retention section mentions a tiered loyalty program with discounts and exclusive offers.

However, the team clarified that the tiered loyalty program is out of current DDP-038 / Sprint 2 implementation scope and will remain future consideration.

This ADR therefore does not invent values such as Bronze, Silver, Gold, or any threshold schedules.

The broader SRS mentions a tiered concept, but detailed tier implementation is deliberately deferred from the current scope.

## 23. Reporting Impact

The reporting scope should remain minimal and aligned with management review requirements.

The minimum data required to support review includes:

- what loyalty change occurred
- when it occurred
- which customer it applied to
- related order/payment references
- points delta
- resulting balance if persisted

This supports auditability without creating a broader analytics dashboard or KPI layer that is not defined by approved requirements.

## 24. Security and Privacy

This ADR also addresses the security and privacy expectations that must accompany the loyalty model.

The proposed model must include:

- customer own-data access only
- Management review in line with ADR-002
- least privilege
- backend-authoritative calculation
- no client-controlled balance values
- no cross-customer retrieval
- no unnecessary PII duplication
- no payment credentials or gateway secrets stored in loyalty records
- history designed to support traceability and auditability

## 25. Consequences

### Positive consequences

- one authoritative loyalty model for the project
- faster customer balance reads for display and validation
- auditable history for management review
- support for duplicate prevention and event idempotency
- traceability across customers, orders, and payments
- support for future approved redemption and reversal logic
- consistent backend and frontend behavior

### Negative consequences / trade-offs

- the SDS and data model require extension
- additional transaction-history management is required
- balance and history must remain transactionally consistent
- unresolved business rules still block final implementation acceptance

## 26. Team-Clarified Business Decisions

The team clarified the following business decisions on 2026-10-04 and these are now treated as resolved for this ADR:

1. Decimal earning values are used, with 2 decimal places retained for the business value.
2. The award trigger is: successful Payment AND Order in the approved COMPLETED state.
3. Product/merchandise value is eligible; clinical eye-test fees are excluded.
4. A valid full refund reverses previously earned points from the refunded eligible purchase.
5. A valid partial refund reverses the corresponding eligible portion proportionally.
6. Post-payment Customer cancellation is not a self-service path; loyalty impact follows the refund/cancellation outcome.
7. Arbitrary staff manual adjustment is prohibited.
8. Redemption conversion is 10 points = LKR 1, equivalent to 1 point = LKR 0.10.
9. Tiered loyalty is deferred out of current scope.
10. The backend must derive eligible spend from authoritative Order/payment/item data and must not trust frontend-supplied values.

Any remaining open items are implementation-contract dependencies rather than unresolved business rules. For example, the exact data contract or product-type association in the checkout/order workflow must be defined in the relevant domain task, but the business rule itself is resolved.

## 27. SRS / SDS / ADR References

### Source references

- SRS FR-009
- SRS FR-010
- SRS Section 10.3 — Loyalty and Customer Rules
- SRS customer-retention and tiered-loyalty scope
- SDS Section 1.2 — Application Layer / Business Logic
- SDS Section 2.2 — Normalized database schema
- SDS Section 6.2 — Authorization model
- ADR-002 — Authoritative RBAC Role Model

### Current implementation references

- current Customer model
- current Order model
- current OrderItem model
- current Payment model
- current payment status constants

### Planned dependent ADR context

- DDP-036 / ADR-003 shopping cart / checkout boundary (planned dependency context)
- DDP-037 / ADR-004 order lifecycle / tracking (planned dependency context)
- DDP-038 / Issue #15 (this ADR)

This ADR does not claim that ADR-003 or ADR-004 already exist in the merged repository state. They are referenced as planned or dependent design context only.

## 28. Review Record

| Date | Reviewer | Role | Status | Notes |
|---|---|---|---|---|
| 2026-10-04 | — | Team / Business Clarification | Completed | Business-rule clarification recorded |
| 2026-10-04 | thiruniimasha | Peer Reviewer | Approved | Peer review completed; non-blocking clarifications incorporated. |
| 2026-10-05 | Project Supervisor | Supervisor | Approved | Formal supervisor review completed and approved. |

Peer review and supervisor approval have been completed and this ADR is Accepted.

## 29. AC1–AC15 Traceability

- AC1: SATISFIED — exact earning formula, 2-decimal rule, and merchandise-vs-clinical eligibility are clarified
- AC2: SATISFIED — successful Payment AND Order COMPLETED is the confirmed award trigger
- AC3: SATISFIED — Option C selected as the authoritative architecture
- AC4: SATISFIED
- AC5: SATISFIED
- AC6: SATISFIED
- AC7: SATISFIED — arbitrary manual adjustment is prohibited
- AC8: SATISFIED
- AC9: SATISFIED — failed or non-successful Payment / non-completed Order do not create confirmed points
- AC10: SATISFIED — full and partial refund behavior are clarified; post-payment cancellation workflow is defined
- AC11: SATISFIED
- AC12: SATISFIED
- AC13: SATISFIED
- AC14: SATISFIED
- AC15: SATISFIED — documentation only

## 30. Consequence of Accepted Status

This ADR is Accepted as the authoritative Loyalty Points Calculation and Persistence Model for OSMS.

It provides the approved design baseline for backend implementation (DDP-058 / loyalty service).

## 31. Final Status

This ADR is an authoritative documentation artifact and does not alter any application code, backend models, APIs, frontend behavior, database schema, tests, or infrastructure configuration directly in this issue.
