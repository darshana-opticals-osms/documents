# ADR-014: Authoritative Sales and Inventory Reporting Definitions, Metrics, and Access Policy

## 1. Title

Authoritative Business Reporting Semantics, Deterministic Metrics Formulas, Inventory Snapshot Boundaries, and Role-Based Reporting Policy for OSMS (FR-012)

## 2. Status

**Proposed** — Drafted to establish canonical sales and inventory reporting semantics, deterministic metric formulas, and privacy-compliant access controls under Issue #21.

---

## 3. Context

The Optical Shop Management System (OSMS) serves Darshana Opticals (Pvt) Ltd, managing omnichannel optical retail sales, eyewear customization, clinic operations, and multi-branch inventory.

Under **FR-012 (Comprehensive Sales / Inventory Reports)**, OSMS must provide management and authorized store personnel with reliable, timely, and secure operational and business intelligence reporting. 

The approved Software Design Specification (SDS v1.0 Section 3.1 & 3.2 – *Access Control Matrix & Data Protection*) establishes the following foundational role assignments for reporting:
- **Management**: FR-012 Owner (organization-wide business, financial, and analytical reporting oversight with customer PII masking).
- **Inventory Manager**: FR-012 View (operational inventory levels, stock reorder alerts, and catalog status across branches).
- **Branch Manager**: FR-012 View (branch-scoped sales performance, order counts, and localized inventory levels).
- **Other Roles** (`BRANCH_ASSISTANT`, `OPTOMETRIST`, `CUSTOMER`): No FR-012 access.

### Existing Domain Decisions and Boundaries

Reporting in OSMS aggregates data produced by previously accepted domain architectures:
- **ADR-002 (Authoritative RBAC Role Model)**: Defines canonical roles (`MANAGEMENT`, `INVENTORY_MANAGER`, `BRANCH_MANAGER`, `BRANCH_ASSISTANT`, `OPTOMETRIST`, `CUSTOMER`) and least-privilege principles.
- **ADR-003 (Shopping Cart Persistence & Checkout Boundary)**: Governs cart-to-order checkout boundaries and inventory reservation mechanics.
- **ADR-004 (Order Lifecycle and Tracking Status Model)**: Defines canonical order states: `ORDER_CREATED`, `PROCESSING`, `READY_FOR_PICKUP`, `PICKED_UP`, `COMPLETED`, and `CANCELLED`.
- **ADR-005 (Loyalty Points Calculation and Persistence Model)**: Governs customer loyalty point earning (1 point per LKR 100 on paid optical items) and redemption rules.
- **ADR-006 (Inventory Reorder-Alert Rules and Persistence Model)**: Governs authoritative reorder rules, safety thresholds, and active alert lifecycles (`OK`, `LOW_STOCK`, `REORDER_TRIGGERED`).
- **ADR-011 (Secure Payment Gateway Integration Architecture)**: Formalizes the payment lifecycle (`PENDING`, `COMPLETED`, `FAILED`, `REFUNDED`), PayHere IPN webhook verification, and payment data schemas.
- **ADR-012 (Authoritative Immutable Audit-Trail Architecture)**: Regulates immutable compliance records for financial, inventory, and status mutations.
- **ADR-013 (Authoritative Order-Status Notification Policy)**: Dictates customer notification delivery upon key lifecycle milestones.

### The FR-012 Design Gap

Although the underlying operational transactional entities exist or are defined, FR-012 does not provide exact mathematical, temporal, or privacy definitions for reporting:
1. **Ambiguous Sale Definition**: Does a "sale" occur when an order is created, when online payment succeeds, when items are picked up, or when an order is marked completed?
2. **Revenue Recognition Rules**: How are partial refunds, full refunds, and order cancellations accounted for against gross sales?
3. **Date Attribution Inconsistencies**: When an order is placed on day $T$ but payment is confirmed on day $T+1$ (or refunded on day $T+10$), which timestamp governs date-range reporting?
4. **Inventory Valuation Ambiguity**: Does "inventory value" mean retail selling value or wholesale cost, given the schema's available attributes?
5. **Snapshot vs. Historical Inventory**: The current inventory document represents real-time state; historical movement logs do not currently exist as an immutable transaction ledger.
6. **Privacy and PII Exposure**: Management reports require customer privacy protection, but the exact masking rules for sales aggregations remain undefined.
7. **RBAC Scope Creep**: "View" access for Branch Managers and Inventory Managers must be constrained strictly to authorized branch boundaries and operational scopes to prevent financial or PII leakage.

Without a canonical ADR, independent backend endpoints and frontend dashboards will implement contradictory aggregation queries, resulting in fragmented metrics and audit failures.

---

## 4. Problem Statement

OSMS requires one authoritative, unified reporting specification that:
- Establishes a mathematically deterministic definition of a "sale" and revenue recognition aligned with ADR-004 and ADR-011.
- Provides closed formulas for all approved sales and inventory metrics, preventing duplicated or disparate calculations across frontend and backend.
- Enforces strict role-based data isolation (RBAC), ensuring Branch Managers only access their assigned branch data and Inventory Managers only view operational inventory metrics without unapproved financial data.
- Guarantees customer privacy by masking Personally Identifiable Information (PII) and excluding clinical prescription data.
- Accurately distinguishes current inventory snapshots from historical inventory movements within existing database schema boundaries.
- Rejects unnecessary infrastructure complexity (no premature data warehouses, OLAP engines, or AI forecasting) in accordance with YAGNI and DRY principles.

---

## 5. Current Data Boundary and Entity Sources

All reporting metrics must derive exclusively from authoritative persistent domain entities without assuming nonexistent collections:

| Domain Entity | Storage Collection | Key Available Fields Used in Reporting | Relevant Constraints / Notes |
|---|---|---|---|
| **Order** | `orders` | `_id`, `orderNumber`, `customerId`, `branchId`, `status`, `totalAmount`, `subtotal`, `discountAmount`, `createdAt`, `updatedAt` | Lifecycle states strictly governed by ADR-004. |
| **OrderItem** | Embedded in `orders.items` | `productId`, `productName`, `category`, `unitPrice`, `quantity`, `subtotal` | Linked directly to parent Order; represents purchased snapshot. |
| **Payment** | `payments` | `_id`, `orderId`, `amount`, `currency`, `method`, `status`, `paidAt`, `refundedAt`, `refundAmount`, `createdAt` | Statuses governed by ADR-011 (`PENDING`, `COMPLETED`, `FAILED`, `REFUNDED`). |
| **Inventory** | `inventories` | `_id`, `branchId`, `sku`, `name`, `category`, `brand`, `unitPrice`, `quantity`, `reorderThreshold`, `reorderStatus`, `updatedAt` | State governed by ADR-006 (`OK`, `LOW_STOCK`, `REORDER_TRIGGERED`). Represents current snapshot. |
| **Branch** | `branches` | `_id`, `branchName`, `branchCode`, `address`, `isActive` | Organizational hierarchy reference. |
| **Customer** | `customers` | `_id`, `firstName`, `lastName`, `email`, `phone` | PII governed by Data Protection standards; must be masked. |

---

## 6. Canonical Definition of a "Sale" and Revenue Recognition Architecture

### 6.1 What Qualifies as a Sale (AC1, AC2)

In an optical retail environment, orders may exist in draft or pending payment stages before financial clearance occurs. 

**Architectural Rule**:
> **A transaction legally and authoritatively qualifies as a "Recognized Sale" if and only if:**
> 1. An associated `Payment` entity exists with `status == 'COMPLETED'`, **AND**
> 2. The associated `Order` entity is in a non-cancelled lifecycle state (`status IN ['PROCESSING', 'READY_FOR_PICKUP', 'PICKED_UP', 'COMPLETED']`).

- **Order Creation (`ORDER_CREATED`)**: Does **not** qualify as a recognized sale. An order awaiting customer online payment capture is an uncommitted reservation.
- **Completed Lifecycle (`COMPLETED`)**: Order lifecycle completion signifies customer handover; it confirms physical delivery, but financial recognition occurs upon verified payment receipt.
- **Authoritative Revenue Anchor**: Revenue recognition in OSMS is **cash-basis/payment-anchored**. Monies are recognized when captured via verified PayHere IPN webhook (for online transactions) or confirmed POS cashier tender (for counter sales).

### 6.2 Financial Treatment of Edge Cases

#### A. Cancelled Orders (AC3)
- Orders with `Order.status == 'CANCELLED'` are **strictly excluded** from recognized sales and revenue totals.
- Cancelled transactions are reported under separate operational monitoring metrics:
  - `Cancelled Orders Count`: Count of orders where `status == 'CANCELLED'`.
  - `Cancelled Orders Value`: Sum of `totalAmount` for cancelled orders.
- If an order is cancelled prior to payment capture (e.g., checkout timeout), it produces zero recognized financial impact.
- If an order is cancelled after payment capture (exception flow), it triggers a refund workflow and is reconciled via the refund policy below.

#### B. Failed and Abandoned Payments (AC4)
- Payment attempts with `Payment.status IN ['FAILED', 'PENDING']` are **completely excluded** from all sales metrics and revenue aggregations.
- Failed payments are visible solely to system administrators and management within technical payment health monitoring metrics (`Failed Payment Count`, `Payment Gateway Failure Rate`).
- Under no circumstances shall failed, abandoned, or pending payment attempts inflate sales, volumes, or item quantities.

#### C. Refund Treatment and Net Revenue Calculation (AC5)
- Under ADR-011, customer returns or post-payment order cancellations can transition payments to `REFUNDED` (or record a `refundAmount`).
- **Gross Revenue vs. Net Revenue**:
  $$\text{Gross Recognized Revenue} = \sum (\text{Payment.amount}) \quad \text{for all qualifying completed payments}$$
  $$\text{Refund Deductions} = \sum (\text{Payment.refundAmount}) \quad \text{for all refunds recorded}$$
  $$\text{Net Sales Revenue} = \text{Gross Recognized Revenue} - \text{Refund Deductions}$$
- **Refund Date Attribution**:
  - Refunds are attributed to the **date the refund was executed** (`Payment.refundedAt`), **not** retroactively to the original order creation date.
  - This prevents closed historical reporting periods (e.g., prior month reconciliations) from unpredictably changing when a refund occurs in a subsequent accounting period.
- **Partial vs. Full Refunds**:
  - Full refund: `refundAmount == Payment.amount`.
  - Partial refund: `0 < refundAmount < Payment.amount`.
  - The deterministic metric calculates exact net revenue by deducting the exact `refundAmount`.

---

## 7. Authoritative Reporting Date Basis and Timezone Boundary (AC6, AC24)

### 7.1 Authoritative Timestamp Basis by Metric Type

To eliminate discrepancies between operational order counts and financial ledger balances:

| Metric Category | Authoritative Timestamp Field | Source Entity | Rationale |
|---|---|---|---|
| **Financial / Revenue Metrics** (Gross Sales, Net Sales, AOV) | `Payment.paidAt` (fallback: `Payment.createdAt` if `paidAt` is unset) | `payments` | Financial revenue must align with actual funds capture. |
| **Operational Order Metrics** (Order Volume, Lifecycle Progression) | `Order.createdAt` | `orders` | Reflects customer demand and fulfillment intake timing. |
| **Refund Metrics** | `Payment.refundedAt` | `payments` | Prevents retroactive mutation of historical accounting periods. |
| **Inventory Snapshot Metrics** | `inventories.updatedAt` / Current Time | `inventories` | Current real-time stock levels; point-in-time snapshot. |

### 7.2 Authoritative Timezone Standard (AC24)

- **System Standard**: All database documents store timestamps in UTC (`ISO 8601`).
- **Reporting Timezone**: All business date boundaries (start-of-day `00:00:00.000` to end-of-day `23:59:59.999`) **must be evaluated in `Asia/Colombo` (UTC+05:30)**.
- **Aggregation Pipeline Rule**: The backend reporting aggregation engine must apply the `Asia/Colombo` timezone offset during MongoDB `$dateToString` or date range grouping. Frontend applications must never apply inconsistent local browser timezone shifts to aggregate boundaries.

---

## 8. Authoritative Sales Metrics Specification (AC7, AC8)

The following deterministic metrics constitute the canonical sales reporting scope for FR-012:

### Metric Definition Matrix

| Metric ID | Metric Name | Source Entity | Qualifying Filter Criteria | Deterministic Formula | Date Field | Supported Grouping Dimensions |
|---|---|---|---|---|---|---|
| **SM-01** | **Recognized Gross Revenue** | `payments` JOIN `orders` | `payments.status == 'COMPLETED'` AND `orders.status IN ['PROCESSING', 'READY_FOR_PICKUP', 'PICKED_UP', 'COMPLETED']` | $\sum (\text{payments.amount})$ | `payments.paidAt` | Total, Day, Week, Month, Branch, Payment Method |
| **SM-02** | **Refund Deductions** | `payments` | `payments.refundAmount > 0` (or `payments.status == 'REFUNDED'`) | $\sum (\text{payments.refundAmount})$ | `payments.refundedAt` | Total, Day, Week, Month, Branch |
| **SM-03** | **Recognized Net Revenue** | `payments` JOIN `orders` | Same as SM-01 and SM-02 | $\text{SM-01 (Gross Revenue)} - \text{SM-02 (Refund Deductions)}$ | `payments.paidAt` / `refundedAt` | Total, Day, Week, Month, Branch |
| **SM-04** | **Total Qualifying Orders Count** | `orders` JOIN `payments` | `payments.status == 'COMPLETED'` AND `orders.status IN ['PROCESSING', 'READY_FOR_PICKUP', 'PICKED_UP', 'COMPLETED']` | $\text{COUNT(DISTINCT orders.\_id)}$ | `orders.createdAt` | Total, Day, Week, Month, Branch, Order Status |
| **SM-05** | **Average Order Value (AOV)** | Calculated from SM-01 & SM-04 | Same as SM-01 and SM-04 | $\frac{\text{SM-01 (Recognized Gross Revenue)}}{\text{SM-04 (Total Qualifying Orders Count)}}$ | `payments.paidAt` | Total, Day, Week, Month, Branch |
| **SM-06** | **Items Sold Quantity** | `orders.items` JOIN `payments` | Same qualifying order/payment filter as SM-01 | $\sum (\text{order.items.quantity})$ | `payments.paidAt` | Total, Branch, Category, Product SKU |
| **SM-07** | **Sales by Branch** | `orders` JOIN `payments` | Same as SM-01, grouped by `orders.branchId` | $\sum (\text{payments.amount}) \quad \text{per branchId}$ | `payments.paidAt` | Branch |
| **SM-08** | **Sales by Product Category** | `orders.items` JOIN `payments` | Same as SM-01, grouped by `item.category` | $\sum (\text{item.unitPrice} \times \text{item.quantity}) \quad \text{per category}$ | `payments.paidAt` | Category (`Frames`, `Lenses`, `Sunglasses`, `Accessories`) |
| **SM-09** | **Payment Method Distribution** | `payments` JOIN `orders` | Same as SM-01, grouped by `payments.method` | Count & Volume per `payments.method` (`PAYHERE_CARD`, `CASH`, `BANK_TRANSFER`) | `payments.paidAt` | Payment Method |
| **SM-10** | **Cancelled Orders Metric** | `orders` | `orders.status == 'CANCELLED'` | Count: $\text{COUNT(orders.\_id)}$<br>Value: $\sum (\text{orders.totalAmount})$ | `orders.updatedAt` | Total, Day, Month, Branch |

*Note: If SM-04 is 0, SM-05 (AOV) returns `0.00` to prevent division by zero.*

---

## 9. Authoritative Inventory Metrics Specification and Boundaries (AC9, AC10, AC11, AC12, AC13)

### 9.1 Current Snapshot Inventory Metrics

Inventory reports under FR-012 provide real-time visibility into branch stock levels and supply replenishment needs:

| Metric ID | Metric Name | Source Entity | Qualifying Filter Criteria | Deterministic Formula | Supported Grouping Dimensions |
|---|---|---|---|---|---|
| **IM-01** | **Total Stock on Hand (Units)** | `inventories` | All active items (`quantity >= 0`) | $\sum (\text{inventories.quantity})$ | Total, Branch, Category, Brand |
| **IM-02** | **Stock Quantity by Branch** | `inventories` | Grouped by `branchId` | $\sum (\text{inventories.quantity}) \quad \text{per branchId}$ | Branch |
| **IM-03** | **Stock by Product Category** | `inventories` | Grouped by `category` | $\sum (\text{inventories.quantity}) \quad \text{per category}$ | Category (`Frames`, `Lenses`, `Sunglasses`, `Contact Lenses`, `Solutions`) |
| **IM-04** | **Low-Stock Items Count** | `inventories` | `reorderStatus == 'LOW_STOCK'` (ADR-006 state: $0 < \text{quantity} \le \text{reorderThreshold}$) | $\text{COUNT(inventories.\_id)}$ | Branch, Category |
| **IM-05** | **Reorder-Triggered Items Count** | `inventories` | `reorderStatus == 'REORDER_TRIGGERED'` (ADR-006 state: $\text{quantity} == 0$) | $\text{COUNT(inventories.\_id)}$ | Branch, Category |
| **IM-06** | **Total Retail Inventory Valuation** | `inventories` | Active inventory items | $\sum (\text{inventories.quantity} \times \text{inventories.unitPrice})$ | Total, Branch, Category |

### 9.2 Current vs. Historical Inventory Boundary & Gap Analysis (AC10, AC11)

- **Snapshot Invariant**: The `inventories` collection stores only the **current point-in-time state** of stock.
- **Historical Stock Movement Boundary**:
  - FR-012 reporting **cannot and shall not fabricate historical inventory trends** (e.g., "what was Branch A's stock level on June 1st?") directly from the `inventories` collection.
  - **Identified Architectural Data Gap**: Historical stock movements (goods received notes, breakages, stock adjustments, internal branch transfers) require a dedicated `inventory_transactions` or `stock_movement_ledger` collection, which is outside the current Sprint 1/2 database schema.
  - **Sprint 2 Reporting Scope**: FR-012 is authoritatively defined as a **Current Inventory Snapshot Report**. Historical inventory trends are explicitly deferred to a future enhancement issue once an immutable inventory ledger is provisioned.

### 9.3 Inventory Valuation Semantics (AC12)

- **Valuation Standard**: Metric **IM-06** represents **Total Retail Selling Valuation** ($\sum \text{quantity} \times \text{unitSellingPrice}$).
- **Wholesale Cost Limitation**: The current `inventories` schema does not capture supplier purchase cost (`costPrice`). To adhere to strict data truthfulness:
  - Reporting APIs must explicitly label this metric as `totalRetailValuation` in all API responses and UI headers.
  - It must **never** be misrepresented to management as "accounting asset valuation" or "cost-of-goods-sold (COGS)".
