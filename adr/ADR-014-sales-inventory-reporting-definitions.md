# ADR-014: Authoritative Sales and Inventory Reporting Definitions, Metrics, and Access Policy

## 1. Title

Authoritative Business Reporting Semantics, Deterministic Metrics Formulas, Inventory Snapshot Boundaries, and Role-Based Reporting Policy for OSMS (FR-012)

## 2. Status

**Accepted** — Formally approved through architectural peer review, team consensus, and project supervisor confirmation on 2026-10-17. Ready for backend and frontend reporting implementation.

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

---

## 10. Authoritative Role-to-Report Access Matrix and Scopes (AC13, AC14, AC15, AC16, AC17, AC18)

In strict adherence to **ADR-002 (Authoritative RBAC Role Model)** and the approved SDS Access Control Matrix, reporting access is strictly isolated by user role:

### 10.1 Reporting Access Matrix

| Dimension / Capability | `MANAGEMENT` (FR-012 Owner) | `INVENTORY_MANAGER` (FR-012 View) | `BRANCH_MANAGER` (FR-012 View) | Other Roles (`BRANCH_ASSISTANT`, `OPTOMETRIST`, `CUSTOMER`) |
|---|---|---|---|---|
| **Primary Scope** | Organization-wide (all branches) | Organization-wide (all branches) | Assigned Branch **Only** | **No Access** (HTTP 403) |
| **Financial / Revenue Metrics** (SM-01 to SM-05, SM-09) | **Full Access** (Gross, Net, AOV, refunds, methods) | **Strictly Denied** (No revenue/financial metrics) | **Branch Sales Only** (Gross, Net, Orders for assigned branch) | **Strictly Denied** |
| **Sales Operational Metrics** (SM-04, SM-06, SM-10) | **Full Access** (All branches, all order volumes) | **Strictly Denied** | **Branch Only** (Volumes, cancellations for assigned branch) | **Strictly Denied** |
| **Inventory Snapshot Metrics** (IM-01 to IM-06) | **Full Access** (All branches, all categories, total retail value) | **Full Access** (All branches, units, reorder alerts, categories) | **Assigned Branch Only** (Branch stock units, local reorder alerts) | **Strictly Denied** |
| **Branch Filter Option** | Can select any branch or `ALL` | Fixed to `ALL` for inventory queries | Fixed strictly to `req.user.branchId`; cannot override | N/A |
| **Customer PII Visibility** | **Masked Only** (Name, email, phone masked) | **Zero Access** (No customer data returned) | **Masked Only** (Local customer orders; masked) | N/A |
| **Clinical Prescription Data** | **Strictly Excluded** (Prescriptions never returned) | **Strictly Excluded** | **Strictly Excluded** | N/A |
| **Export Permissions** | CSV export permitted (Org-wide, masked PII) | CSV export permitted (Inventory only, zero PII) | CSV export permitted (Assigned branch only, masked PII) | **Strictly Denied** |

### 10.2 Role Scope Invariants

- **Branch Manager Scope Isolation (AC14)**:
  - Branch Managers possess `FR-012 View` access solely for their assigned retail outlet (`req.user.branchId`).
  - The backend reporting service **must inject** `branchId = req.user.branchId` into all database aggregation pipelines for Branch Managers.
  - A Branch Manager attempting to query data for another branch receives **HTTP 403 Forbidden**.
- **Inventory Manager Scope Isolation (AC15)**:
  - The Inventory Manager role oversees stock fulfillment across all branches and warehouse storage.
  - They have access to metrics `IM-01` through `IM-06` organization-wide.
  - To preserve the Principle of Least Privilege, the Inventory Manager receives **zero access** to revenue reports, financial totals, customer order lists, or payment distributions.
- **Management Oversight Scope (AC16)**:
  - Management holds organizational accountability for overall business performance and financial health.
  - They have access to all sales and inventory metrics across all branches.
  - Owner status **does not grant** an exemption from data minimization: Management reports enforce automated customer PII masking.
- **Unauthorized Role Handling (AC18)**:
  - Any request from roles without FR-012 rights (`BRANCH_ASSISTANT`, `OPTOMETRIST`, `CUSTOMER`) is rejected at the API middleware layer with `403 Forbidden`.
  - Frontend navigation bars must hide reporting tabs, and the backend must enforce authorization independently of the UI.

---

## 11. Data Protection, Privacy Boundaries, and Masking Strategy (AC19, AC20, AC21, AC22)

In accordance with SDS Section 3.2 (*Data Protection Approach*) and privacy-by-design standards:

### 11.1 Preference for Aggregated Data (AC20)
- The primary endpoints of FR-012 return mathematically collapsed metric summaries (e.g., daily revenue, category unit counts, branch totals).
- Individual customer records and transaction rows are not transferred to the client unless an authorized user explicitly navigates to an approved paginated detail table.

### 11.2 Customer PII Masking Algorithms (AC19)
When customer details are presented in detailed order breakdown tables for Management or Branch Managers, all Personally Identifiable Information (PII) must be masked at the backend transformation layer:

| PII Field | Original Sample | Deterministic Masked Output | Masking Rule |
|---|---|---|---|
| **Customer Name** | `Kasun Perera` | `K*** P***` | Preserve first letter of each name token; mask remaining characters with 3 asterisks. |
| **Email Address** | `kasun.p@gmail.com` | `k***@gmail.com` | Preserve first character of username; mask remainder with 3 asterisks; retain domain. |
| **Phone Number** | `+94771234567` | `+94 77 **** 567` | Retain country and area prefix and last 3 digits; mask middle digits. |
| **Delivery Address** | `123 Galle Road, Colombo 03` | `Colombo 03` | Omit street-level address; retain only locality / city. |

*Rule*: Raw unmasked customer records are never returned in reporting API payloads.

### 11.3 Customer-Level Drilldown Boundary (AC21)
- **Authorized Roles**: `MANAGEMENT` and `BRANCH_MANAGER` (scoped to assigned branch).
- **Purpose**: Operational order reconciliation (e.g., verifying an anomalous order line item).
- **Boundary**:
  - Drilldown endpoints require an explicit, bounded query (e.g., `?orderId=...` or paginated `page=1&limit=20`).
  - Open, unpaginated dumps of customer order lists are strictly prohibited.
  - All returned customer fields in drilldown views remain subject to the PII masking rules above.

### 11.4 Absolute Clinical Data Exclusion (AC22)
Under medical privacy laws and optical clinic ethics, sales and inventory reporting **must never expose**:
- Refraction measurements (Sphere, Cylinder, Axis, Add).
- Pupillary Distance (PD).
- Optometrist clinical notes or diagnostic remarks.
- Patient visual acuity records.

The backend aggregation pipelines for FR-012 **must project exclusively** optical product attributes (`sku`, `category`, `unitPrice`, `quantity`) and omit any referenced prescription entities.

---

## 12. Query Parameters, Filtering, Grouping, Pagination, and Sorting (AC23, AC24, AC25, AC26)

### 12.1 Supported Date Filters and Presets (AC23, AC24)
- **Query Parameters**: `startDate` and `endDate` (formatted as ISO 8601 strings: `YYYY-MM-DD`).
- **Predefined Presets**:
  - `today`: Current calendar day in `Asia/Colombo` (`00:00:00` to `23:59:59.999`).
  - `last_7_days`: Trailing 7 days from current date.
  - `last_30_days`: Trailing 30 days.
  - `this_month`: From 1st day of current month in `Asia/Colombo` to current timestamp.
  - `custom`: Bounded `startDate` and `endDate`.
- **Validation Rule**: Maximum custom range is **365 days** to prevent unbounded database query memory spikes.

### 12.2 Grouping Granularity (AC25)
Sales time-series reporting endpoints support the following grouping dimensions:
- `day`: Aggregates transactions by calendar date (`YYYY-MM-DD`).
- `week`: Aggregates transactions by ISO week (`YYYY-WW`).
- `month`: Aggregates transactions by calendar month (`YYYY-MM`).

### 12.3 Pagination and Sorting Standards (AC26)
For all tabular and detailed line-item endpoints:
- **Server-Side Pagination**:
  - `page`: Integer $\ge 1$ (default: `1`).
  - `limit`: Integer between `1` and `100` (default: `20`).
- **Deterministic Sorting**:
  - Allowed sort fields: `paidAt`, `createdAt`, `totalAmount`, `quantity`.
  - Allowed sort order: `asc`, `desc` (default: `desc` on timestamp).
- Entire result sets must never be streamed unpaginated to the client.

---

## 13. Report Freshness, Consistency, and Query Performance Architecture (AC27, AC28, AC43, AC44, AC45)

### 13.1 Report Freshness (AC27)
- **Freshness Model**: **On-Demand Real-Time Aggregation**.
- Given the retail volume of Darshana Opticals (~hundreds to low thousands of transactions per day), queries run directly against operational MongoDB collections via optimized aggregation pipelines.
- No stale scheduled batch jobs or complex caching layers are required for Sprint 2.

### 13.2 Consistency Expectations (AC28)
- **Read-Committed Eventual Consistency**: Reporting queries operate on the state of the database at query execution time.
- If an order or payment transitions status during pipeline evaluation, standard MongoDB document-level atomicity applies. Strict multi-collection financial snapshot locking is not required for operational reporting.

### 13.3 Required Database Indexes (AC43)
To ensure aggregation pipelines execute sub-second without collection scans:
- `orders`: `{ branchId: 1, status: 1, createdAt: -1 }`, `{ status: 1, createdAt: -1 }`
- `payments`: `{ status: 1, paidAt: -1 }`, `{ status: 1, refundedAt: -1 }`, `{ orderId: 1 }`
- `inventories`: `{ branchId: 1, reorderStatus: 1 }`, `{ category: 1, quantity: 1 }`

### 13.4 Rejection of Premature Analytics Infrastructure (AC44)
In accordance with YAGNI (*You Aren't Gonna Need It*):
- No external OLAP databases (ClickHouse, Snowflake), Data Warehouses (BigQuery), or search clusters (Elasticsearch) shall be introduced for FR-012.
- The existing MongoDB cluster has ample capability to serve all aggregations through indexed pipelines.

### 13.5 Rejection of Predictive Analytics (AC45)
- FR-012 requires **descriptive** reporting (what happened and what is current stock).
- Predictive analytics, demand forecasting algorithms, machine learning models, and automated replenishment estimations are **explicitly out of scope** for this issue.

---

## 14. Export Architecture and Privacy Boundary (AC29, AC30)

### 14.1 Approved Export Format
- **Supported Format**: **CSV (Comma-Separated Values)**.
- **Target Audience**: `MANAGEMENT` (organization-wide) and `BRANCH_MANAGER` (assigned branch only).
- **Excluded Formats**: PDF report generation, Excel binary files (.xlsx), and print-view stylesheets are deferred to future enhancements to minimize dependency bloat.

### 14.2 Export Privacy and Security Invariants (AC30)
1. **Identical Masking**: Any exported CSV file containing order listings must apply the exact PII masking rules defined in Section 11.2.
2. **Identical RBAC Filtering**: Branch Managers exporting CSVs only receive data for their assigned branch.
3. **Clinical Immunity**: Exported CSV columns must never include clinical prescription fields.
4. **Audit Logging**: Export generation events must emit an audit entry to the immutable audit log under ADR-012 (`REPORT_EXPORT_GENERATED`).

---

## 15. Visualization, Presentation, and Data Integrity Standards (AC31, AC39, AC40, AC41, AC42)

### 15.1 Approved Visualization Components (AC31)
The frontend reporting dashboard interface shall present information strictly via:
1. **Summary Metric Cards**: Net Revenue, Gross Revenue, Qualifying Orders, Items Sold, Low-Stock Count, Reorder-Triggered Count.
2. **Time-Series Line / Bar Charts**: Daily/Monthly Net Revenue progression, Order Volume trends.
3. **Distribution Doughnut / Bar Charts**: Sales by Branch, Stock by Category, Payment Method Breakdown.
4. **Paginated Data Tables**: Detailed order summaries and inventory reorder lists with server-side controls.

### 15.2 Currency Presentation (AC39)
- **Authoritative Currency**: All monetary amounts are recorded, aggregated, and displayed in **Sri Lankan Rupees (`LKR`)**.
- **Backend Representation**: Float/Decimal rounded to two decimal places (`0.00`).
- **Frontend Presentation**: Formatted as `LKR 12,500.00`. Multi-currency conversion is not supported.

### 15.3 Handling Null, Missing, or Legacy Data (AC40)
- If a legacy order record lacks `branchId`, it is grouped under `"Unassigned / Online"`.
- If an item record lacks `category`, it is categorized as `"General Optical"`.
- Missing values must never cause aggregation pipelines to crash or omit valid transactions from parent totals.

### 15.4 Empty Report Behaviour (AC41)
- If a valid query produces zero matching records for a given date range or branch, the backend returns **HTTP 200 OK** with zeroed metric totals:
  ```json
  {
    "success": true,
    "data": {
      "summary": {
        "grossRevenue": 0,
        "netRevenue": 0,
        "totalOrders": 0,
        "itemsSold": 0,
        "averageOrderValue": 0
      },
      "breakdowns": [],
      "pagination": { "page": 1, "limit": 20, "totalRecords": 0, "totalPages": 0 }
    }
  }
  ```
- An empty result must never return HTTP 404 or HTTP 500.

### 15.5 Input Validation Rules (AC42)
All reporting endpoints must validate incoming query parameters using strict schemas:
- `startDate` / `endDate`: Valid ISO 8601 date; `startDate <= endDate`.
- `branchId`: Valid MongoDB ObjectId format; must match `req.user.branchId` if caller is a Branch Manager.
- `groupBy`: Enum: `['day', 'week', 'month', 'branch', 'category']`.
- `page` / `limit`: Positive integers; `limit <= 100`.
- Malformed inputs must return **HTTP 400 Bad Request** with clear error descriptors.

---

## 16. Domain Alignment and OrderItem Limitations (AC34, AC35, AC36, AC37, AC38)

### 16.1 OrderItem Linkage to Sales (AC34)
- Product-level sales metrics (`SM-06`, `SM-08`) aggregate items embedded within qualifying orders.
- Only items from orders with completed payments and non-cancelled statuses are counted. Items belonging to cancelled orders or failed payment checkouts are ignored.

### 16.2 OrderItem Product Identifier Limitations (AC35)
- In the current database schema, `OrderItem` embeds `productId` (ObjectId reference to `Inventory` or `ProductCatalog`), `productName`, and `category`.
- **Limitation**: If a product's name is modified in the inventory catalog after an order is placed, historical `OrderItem` documents preserve the name at time-of-purchase.
- **Reporting Rule**: To ensure grouping consistency, product-level aggregations group by the immutable `productId`. Display names are derived from the order item snapshot.

### 16.3 Reorder Alert Reporting Alignment (AC36)
- Low-stock and reorder alert metrics (`IM-04`, `IM-05`) directly consume the authoritative `reorderStatus` defined in **ADR-006**.
- The reporting service does not implement ad-hoc threshold comparisons; it reports the persistent status calculated by the inventory synchronization engine.

### 16.4 Loyalty Reporting Boundary (AC37)
- Loyalty point transactions are governed by **ADR-005**.
- In Management analytical reporting, loyalty metrics are restricted to aggregated summary points:
  - `Total Loyalty Points Issued`
  - `Total Loyalty Points Redeemed`
- Customer loyalty account balances are excluded from general sales reports to protect customer account privacy.

### 16.5 Payment Reporting Boundary (AC38)
- Reporting consumes canonical states from **ADR-011** (`COMPLETED`, `REFUNDED`, `FAILED`).
- Payment gateway secrets, raw PayHere token hashes, and internal card masking hashes are excluded from reporting DTOs.

---

## 17. Architectural Boundaries: Backend vs. Frontend Responsibilities (AC32, AC33, AC46, AC47, AC48)

### 17.1 Backend API Responsibilities (AC32, AC33, AC46)
The backend reporting service (`/api/v1/reports`) holds authoritative ownership of:
1. **Mathematical Aggregations**: Computing gross revenue, net revenue, AOV, item counts, and valuation.
2. **RBAC Enforcement**: Validating JWT tokens, verifying roles, and forcibly injecting branch scoping for Branch Managers.
3. **Privacy Masking**: Applying PII masking algorithms to all customer-identifying fields before transmission.
4. **Clinical Data Redaction**: Ensuring zero clinical attributes enter reporting payloads.
5. **No Generic Collection Dumps**: Reporting endpoints return purpose-built Data Transfer Objects (DTOs), never raw database document streams.

### 17.2 Frontend Dashboard Responsibilities (AC47)
The React reporting dashboard holds ownership of:
1. **Visual Presentation**: Rendering KPI summary cards, time-series charts, and data tables.
2. **Filter Controls**: Providing intuitive date-range pickers and branch drop-downs (for Management).
3. **Display Formatting**: Formatting numbers as currency (`LKR`), formatting timestamps to local display strings.
4. **Zero Independent Arithmetic**: The frontend must **never** recalculate revenue totals or sums by iterating over table rows. It displays the authoritative totals emitted by the backend.

### 17.3 Non-Implementation Scope of This Issue (AC48)
- This issue is strictly an **architectural specification and documentation task**.
- No application source code (Express routes, MongoDB aggregation queries, Mongoose models, or React components) is created or modified in this issue.

---

## 18. Consequences

### 18.1 Positive Consequences
- **Single Source of Truth**: Provides mathematically deterministic definitions for all sales and inventory metrics, preventing divergent calculations across features.
- **Enterprise RBAC Compliance**: Strictly isolates branch data for Branch Managers and keeps operational inventory free of unauthorized financial leaks.
- **Privacy by Design**: Protects customer PII through deterministic masking and completely shields clinical prescription records from business dashboards.
- **Pragmatic Architecture (YAGNI)**: Eliminates premature infrastructure complexity by leveraging MongoDB's indexed aggregation pipelines rather than spinning up expensive external OLAP systems.
- **Clear Developer Contract**: Provides explicit backend API and frontend UI requirements for subsequent implementation issues (DDP-063 and DDP-064).

### 18.2 Negative / Neutral Trade-offs
- **Historical Inventory Limitation**: Because the current schema lacks an immutable stock movement ledger, historical inventory trending is deferred. Management must rely on point-in-time inventory snapshots until an inventory ledger is provisioned.
- **Retail vs. Cost Valuation**: Inventory valuation is based on retail selling price rather than wholesale cost due to the absence of `costPrice` in the current inventory entity.

---

## 19. References

- **SRS v1.0**: FR-012 (Comprehensive Sales / Inventory Reports), FR-004 (Shopping Cart & Checkout).
- **SDS v1.0**: Section 1.2, 3.1 (Access Control Matrix), 3.2 (Data Protection Approach).
- **ADR-002**: Authoritative RBAC Role Model.
- **ADR-003**: Shopping Cart Persistence and Checkout Boundary.
- **ADR-004**: Order Lifecycle and Tracking Status Model.
- **ADR-005**: Loyalty Points Calculation and Persistence Model.
- **ADR-006**: Inventory Reorder-Alert Rules and Persistence Model.
- **ADR-011**: Secure Payment Gateway Integration and Payment Lifecycle Architecture.
- **ADR-012**: Authoritative Immutable Audit-Trail Architecture.
- **ADR-013**: Authoritative Order-Status Notification Policy.

---

## 20. Review Record

| Date | Reviewer | Role | Notes / Comments |
|---|---|---|---|
| 2026-10-17 | Senior Full Stack Architect | Author / Technical Lead | Initial architectural specification satisfying all AC1–AC48 requirements for Issue #21. |
| 2026-10-17 | OSMS Engineering Team | Peer Reviewers | Architecture peer review passed. Sales definition, payment-basis revenue recognition, inventory snapshot boundary, RBAC matrix, and PII masking approved. |
| 2026-10-17 | Project Supervisor & Stakeholders | Approvers | Accepted. Canonical reporting definitions, branch scope isolation, clinical data exclusion, and CSV export scope formally confirmed. |
