# ADR-005: Loyalty Points Calculation and Persistence Model

## 1. Title

Loyalty Points Calculation and Persistence Model for OSMS

## 2. Status

**Proposed**

This ADR does not yet carry Accepted status. Several required business rules remain unresolved and require stakeholder/team confirmation before the design may be finalized as an accepted implementation standard.

This ADR intentionally does not claim:

- stakeholder confirmation
- reviewer approval
- supervisor approval
- approval date
- final acceptance

## 3. Context

The OSMS customer-retention scope includes a loyalty program as part of the retail and customer-relationship model. The approved SRS and SDS identify loyalty as a business capability tied to customer purchases and management oversight.

The current project state shows the following relevant constraints:

- The customer-retention requirement includes a loyalty program with point accrual and redemption behavior.
- The SRS defines the loyalty earning rule in the Business Rules section.
- The SDS documents the approval and monitoring responsibilities tied to management review.
- The RBAC model is governed by ADR-002.
- The current backend data model does not yet implement a loyalty balance/history model.

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

The following rules are source-confirmed and must be preserved exactly:

### 6.1 Source-confirmed earing rule

SRS Business Rule 10.3:

> "Loyalty points are earned at a rate of 1 point for every 100 LKR spent on frames and accessories; however, points are not accrued for clinical eye test fees."

This means:

- 1 loyalty point is earned for every LKR 100 spent
- eligible spend is on frames and accessories
- clinical eye-test fees do not earn points

### 6.2 Source-confirmed redemption rule

SRS Business Rule 10.3:

> "Redemption of loyalty points is allowed only when the customer's total point balance exceeds 500 points."

Important precision:

- the source says exceeds 500
- this ADR does not change that to 500 or more

### 6.3 Confirmed eligible purchase categories

Confirmed eligible:

- frames
- accessories

Confirmed excluded:

- clinical eye-test fees

### 6.4 Confirmed non-eligible / unresolved categories

The approved source does not explicitly confirm the following categories as eligible:

- contact lenses
- prescription lenses
- services
- other future categories

Those remain unresolved unless an approved source explicitly defines them.

### 6.5 Automatic loyalty behavior

The source states that FR-010 requires loyalty points to be calculated and updated automatically based on customer purchase value.

This ADR therefore treats loyalty accumulation as an automatic backend business function, not a manual or client-driven value.

### 6.6 Unresolved rule for non-multiples of LKR 100

The SRS does not define how to handle partial LKR 100 blocks, including examples such as:

- LKR 50
- LKR 150
- LKR 250
- LKR 10,050

This is a required business decision and remains unresolved.

Recommendation only — not approved:

The system could choose a consistent integer policy such as floor-based earning for partial blocks, but this is not source-approved and must not be presented as a confirmed business rule.

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

The authoritative SRS/SDS/ADR sources do not currently approve arbitrary manual loyalty adjustments.

This ADR therefore records:

Manual loyalty adjustments are NOT approved for implementation by this ADR until an explicit requirement and stakeholder decision defines:

- whether they are allowed
- which role may perform them
- what evidence/reason must be captured
- what audit requirements apply

This ADR distinguishes clearly between:

- automatic earning
- automatic reversal or compensating transaction
- redemption deduction

and:

- arbitrary staff manual adjustment

The latter is not approved by current source material, and the design must not collapse the categories.

## 14. Award Trigger

The source material references loyalty activities and the operational flow around point-of-sale behavior, but it does not clearly define the authoritative persistence event that creates a confirmed award.

Possible unresolved alternatives include:

- point of sale
- payment completion
- order completion
- another approved event

This ADR does not finalize a trigger as a confirmed business rule.

Recommendation only — stakeholder/team confirmation required:

A recommended operational design is to award points only after a payment reaches the authoritative successful state and the sale/order is confirmed.

This recommendation should not be treated as an approved source rule until stakeholder review confirms it.

## 15. Failed Payment Behaviour

The ADR must satisfy the intent of AC9:

A failed or non-qualifying payment must not create a confirmed loyalty award.

Therefore loyalty processing must reject or prevent confirmed earning when the authoritative payment or sale does not qualify.

The system must not award points for:

- failed transactions
- cancelled transactions

This does not resolve later refund/cancellation behavior after a previously awarded point event; that is a separate unresolved question.

## 16. Refund / Cancellation Handling

The SRS defines a 7-day return policy for eyewear frames, but it does not define the loyalty effect after a return, cancellation, or partial refund.

The following remain unresolved:

- full refund treatment
- cancellation after award
- partial refund treatment

Reasonable approaches include:

A. retain points
B. directly reverse existing award
C. append a compensating negative loyalty transaction

Recommendation only — not approved:

Option C is the most auditable because it preserves the original award history and records a compensating reversal as a separate event. This is a design recommendation, not a source-confirmed business rule.

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

The following remain out of scope unless explicitly approved:

- point-to-LKR conversion
- redemption percentage
- expiry policy
- redemption cap
- tier-specific discounts or offers

If future redemption implementation deducts points, the proposed Option C model supports auditable negative transactions by storing the deduction as a transaction/event with an opposite signed delta.

## 22. Tiered Loyalty Program Scope

The SRS high-level customer-retention section mentions a tiered loyalty program with discounts and exclusive offers.

However, the approved source material does not define:

- tier names
- tier thresholds
- tier benefits

This ADR therefore does not invent values such as Bronze, Silver, Gold, or any threshold schedules.

Tier details are outside the source-confirmed scope of ADR-005 and remain future work requiring clarification.

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

## 26. Unresolved Decisions Requiring Stakeholder Confirmation

The following questions remain open and must be answered by the team before the ADR can become Accepted:

1. How are non-LKR-100 multiples rounded?
2. What exact business event awards points?
3. Are contact lenses, prescription lenses, or other categories eligible?
4. What happens to points after a full refund?
5. What happens to points after cancellation following an award?
6. How should partial refunds affect points?
7. Are manual adjustments ever allowed, and if so under what authority?
8. What is the approved point-to-currency redemption conversion?
9. Are loyalty tiers part of the current phase, and if so what are their rules?
10. What authoritative order/product data will preserve item eligibility for backend calculation?

These questions are not answered in the approved source material and must remain clearly visible in the ADR.

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
| — | — | Team / Stakeholder | Pending | Business-rule clarification required |
| — | — | Peer Reviewer | Pending | — |

No reviewer names, approval dates, or supervisor approvals are included because this ADR remains Proposed and is not yet accepted.

## 29. AC1–AC15 Traceability

- AC1: PARTIAL — earning rate confirmed; rounding unresolved
- AC2: PENDING — award trigger requires confirmation
- AC3: PROPOSED — Option C selected for review
- AC4: SATISFIED IN PROPOSED DESIGN
- AC5: SATISFIED IN PROPOSED DESIGN
- AC6: SATISFIED IN PROPOSED DESIGN
- AC7: PENDING / NOT APPROVED
- AC8: SATISFIED IN PROPOSED DESIGN
- AC9: SATISFIED IN PROPOSED DESIGN
- AC10: PENDING — refund/cancellation business rule
- AC11: SATISFIED IN PROPOSED DESIGN
- AC12: SATISFIED IN PROPOSED DESIGN
- AC13: SATISFIED IN PROPOSED DESIGN
- AC14: SATISFIED IN PROPOSED DESIGN
- AC15: SATISFIED — documentation only

## 30. Consequence of Current Draft State

This ADR is ready for stakeholder review as a Proposed design.

It is not ready to become Accepted until the unresolved business decisions listed above are resolved.

This issue remains incomplete until those stakeholder decisions are resolved and the ADR is accepted through the normal review process.

## 31. Final Draft Status

This ADR is intentionally a documentation-only proposal and does not alter any application code, backend models, APIs, frontend behavior, database schema, tests, or infrastructure configuration.
