# ADR-006: Inventory Reorder-Alert Rules and Persistence Model

## 1. Title

Inventory Reorder-Alert Rules and Persistence Model for OSMS

## 2. Status

**Proposed** — Team business rules have been agreed. Pending formal peer review and approval.

---

## 3. Context

The Optical Shop Management System (OSMS) requires:

- FR-005 – Real-Time Stock Synchronization
- FR-006 – Automated Inventory Reorder Alerts

Sprint 1 already provides:

- `Branch` data model
- `Inventory` data model
- Inventory management APIs
- Inventory role-based access control
- Inventory quantity updates

The existing `Inventory` model currently stores information including:

- branch relationship,
- item name,
- category,
- brand,
- price,
- quantity.

However, the current architecture does not define:

- the exact reorder threshold,
- the low-stock trigger condition,
- where the threshold is stored,
- how reorder alerts are persisted,
- how duplicate alerts are prevented,
- when an alert is resolved,
- how repeated reminders are delivered,
- which roles can manage the threshold and alerts,
- how checkout-triggered stock changes participate in reorder evaluation.

Therefore, one authoritative reorder-alert architecture is required before Sprint 2 reorder-alert implementation begins.

---

## 4. Existing Design Gap

The current `Inventory` model contains stock quantity but does not contain an approved reorder threshold or alert state.

There is currently no approved field or entity for:

- reorder threshold configuration,
- low-stock alert state,
- reminder state,
- alert resolution state.

Without one approved design, separate developers could introduce different rules for:

- low-stock detection,
- alert creation,
- duplicate handling,
- reminder scheduling,
- threshold configuration.

DDP-039 therefore defines one authoritative backend business rule.

---

## 5. Problem

OSMS must automatically identify inventory items that have reached the approved low-stock level and notify the responsible inventory staff.

The architecture must define:

1. the low-stock trigger condition,
2. the global reorder threshold,
3. threshold persistence,
4. alert persistence,
5. duplicate prevention,
6. alert resolution,
7. immediate evaluation after stock changes,
8. periodic reminder behaviour,
9. integration with manual inventory updates,
10. integration with checkout/order stock deductions,
11. role-based visibility and threshold management,
12. notification-channel scope,
13. validation requirements,
14. reporting impact,
15. implementation boundaries.

---

## 6. Confirmed Low-Stock Business Rule

The approved global reorder threshold is:

`5`

An Inventory item is considered low stock when:

`quantity <= 5`

Examples:

- quantity = 8 → not low stock
- quantity = 6 → not low stock
- quantity = 5 → low stock
- quantity = 3 → low stock
- quantity = 0 → low stock

The low-stock calculation must be performed by authoritative backend business logic.

The frontend must not independently decide whether an item is low stock.

---

## 7. Threshold Scope Options

### Option A — Global Threshold

One reorder threshold is used for all Inventory items.

Example:

`global reorder threshold = 5`

All Inventory records are evaluated against the same value.

#### Advantages

- Simple to understand.
- Easy to configure.
- Easy to maintain consistently.
- Avoids duplicated threshold values across Inventory records.

#### Disadvantages

- Different items or branches cannot use different thresholds.

---

### Option B — Item-Specific Threshold

Each Inventory item stores its own reorder threshold.

#### Advantages

- More flexible for different products.

#### Disadvantages

- More configuration is required.
- Threshold values must be maintained for every Inventory record.

---

### Option C — Category-Specific Threshold

Different inventory categories use different thresholds.

#### Advantages

- More flexible than one global value.

#### Disadvantages

- Adds category configuration and business-rule complexity.

---

## 8. Threshold Scope Decision

**Selected: Option A — Global Threshold.**

OSMS will use one global reorder threshold.

The current approved threshold value is:

`5`

Therefore:

`quantity <= 5`

means the Inventory item is low stock.

---

## 9. Threshold Persistence Options

### Option A — Store Threshold in Every Inventory Record

Each Inventory document contains its own threshold.

This is not selected because OSMS uses one global threshold.

### Option B — Dedicated Per-Item Reorder Configuration

Reorder settings are stored separately for individual items.

This is not selected because item-specific configuration is not required.

### Option C — Global Configuration

Store one authoritative reorder threshold as system inventory configuration.

---

## 10. Threshold Persistence Decision

**Selected: Option C — Global Configuration.**

The reorder threshold must be stored as one authoritative configurable value rather than duplicated across Inventory documents.

The initial approved value is:

`5`

The exact implementation structure and configuration model may be finalized in the backend implementation issue, but there must be only one authoritative threshold source.

---

## 11. Alert Persistence Options

### Option A — Persist Reorder Alerts

Create and maintain a backend reorder-alert record associated with the affected Inventory item.

#### Advantages

- Supports in-application alert display.
- Supports alert status and reminder information.
- Allows duplicate prevention.
- Allows alert resolution to be tracked.

#### Disadvantages

- Requires additional persistence logic.

---

### Option B — Dynamic Calculation Only

Do not store alert records. Calculate low-stock state directly from Inventory quantity whenever data is requested.

#### Advantages

- Simple persistence model.
- No separate alert records.

#### Disadvantages

- Cannot easily maintain reminder state.
- Cannot easily track active/resolved alert state.
- Less suitable for repeated in-app reminder behaviour.

---

## 12. Alert Persistence Decision

**Selected: Option A — Persist Reorder Alerts.**

A reorder alert must be persisted for an Inventory item when:

`quantity <= 5`

The alert must be associated with the unique Inventory record identifier.

The existing Inventory MongoDB `_id` is the authoritative Inventory identifier.

This ensures that stock for the same type of item at different branches can be handled independently.

---

## 13. Minimum Reorder Alert Data Contract

A persisted reorder alert must contain enough information to identify and manage the low-stock condition.

At minimum, the design must support:

- Inventory reference / Inventory ID,
- branch relationship through the Inventory record,
- current quantity,
- threshold value,
- alert status,
- created timestamp,
- updated timestamp,
- resolved timestamp where applicable,
- last reminder timestamp where applicable.

Exact Mongoose field names and indexes remain part of the backend implementation issue.

---

## 14. Duplicate-Prevention Behaviour

OSMS must not create uncontrolled duplicate alerts while the same Inventory item remains low stock.

Only one active reorder alert may exist for one Inventory record.

When the same Inventory item is detected as low stock again:

- the existing active alert must be updated,
- the current quantity must be refreshed,
- the reminder information may be updated,
- a second duplicate active alert must not be created.

Conceptually:

`Inventory ID -> maximum one active reorder alert`

---

## 15. Alert Resolution Behaviour

When an Inventory item's quantity becomes greater than the global threshold:

`quantity > 5`

the low-stock condition no longer exists.

The corresponding active alert must therefore:

- stop appearing as an active low-stock alert,
- stop generating repeated reminders,
- be marked as resolved/inactive.

The system does not require a separate detailed alert-history interface or historical analytics for Sprint 2.

Once an active reorder alert is resolved, that alert record must remain resolved and must not be reactivated.

If the same Inventory record later becomes low stock again after the previous alert was resolved, the system must create a new active reorder alert for the new low-stock occurrence.

Duplicate prevention therefore applies while an active alert already exists for the Inventory record.

For a newly created low-stock alert after a previous resolved alert:

- a new created timestamp must be recorded,
- the resolved timestamp must initially be empty,
- the last reminder timestamp must initially be empty,
- the previous resolved alert and its timestamps must remain unchanged.

---

## 16. Immediate Reorder Evaluation

Whenever an authoritative backend stock-changing operation successfully changes Inventory quantity, reorder evaluation must occur immediately.

Examples include:

- manual Inventory quantity updates,
- checkout/order stock deductions,
- other approved backend stock-changing operations.

After the quantity is updated, the backend must evaluate:

`quantity <= global reorder threshold`

If true:

- create the alert if one does not exist,
- otherwise update the existing active alert.

If false:

- resolve/deactivate the existing alert where applicable,
- stop reminders.

The frontend is not the authority for this decision.

---

## 17. Periodic Reminder Behaviour

In addition to immediate low-stock evaluation, active low-stock conditions must generate repeated in-application reminders during the approved working period.

The agreed working period is:

`8:00 AM – 6:00 PM`

The reminder interval is:

`every 3 hours`

The backend scheduling implementation must ensure that active low-stock alerts are reminded at approximately three-hour intervals during the approved working period.

The exact scheduler implementation and exact execution timestamps remain part of the backend implementation issue.

A reminder must only be generated while the Inventory item remains low stock.

Before generating each reminder, the backend must confirm that:

`quantity <= 5`

If the quantity has risen above the threshold, reminders must stop.

The periodic reminder process must not create duplicate active alert records.

It updates or uses the existing persisted alert.

---

## 18. Inventory Update Integration

The existing Inventory quantity-update functionality must remain the authoritative stock-update path.

Reorder-alert evaluation must be integrated with backend Inventory business logic so that stock changes and alert evaluation do not become two independent implementations.

Conceptually:

`Inventory quantity update -> successful persistence -> reorder evaluation`

The same centralized reorder-evaluation logic must be reusable by other stock-changing workflows.

---

## 19. Checkout / Order Integration

Successful checkout and Order processing may reduce Inventory quantity.

Checkout must not contain a separate hard-coded rule such as:

`quantity <= 5`

Instead, checkout must call or use the same centralized Inventory/reorder-evaluation mechanism.

Conceptually:

`Successful checkout`
`-> Inventory quantity decreases`
`-> centralized reorder evaluation`
`-> create/update/resolve reorder alert`

This keeps DDP-041 compatible with ADR-006.

---

## 20. Authorization Model

Authorization must follow ADR-002 – Authoritative RBAC Role Model.

### Inventory Manager

`INVENTORY_MANAGER` is the primary role responsible for stock management and reorder alerts.

The Inventory Manager may:

- view active reorder alerts,
- manage reorder-alert information where required,
- view the global reorder threshold,
- modify the global reorder threshold.

### Branch Manager

`BRANCH_MANAGER` may receive approved read-only visibility of relevant Inventory/reorder-alert information.

Read access must not automatically provide permission to:

- change the global reorder threshold,
- modify Inventory quantities,
- resolve alerts manually where not approved.

### Management

`MANAGEMENT` may receive approved read-only visibility for inventory/reporting purposes.

Read access does not provide threshold-management permission.

### System Administrator

`SYSTEM_ADMIN` is not the normal business owner of reorder-threshold management.

The reorder threshold is treated as an Inventory business rule managed by `INVENTORY_MANAGER`.

---

## 21. Threshold Configuration Authorization

Only an authenticated user with the canonical role:

`INVENTORY_MANAGER`

may modify the global reorder threshold through the approved backend functionality.

Changing the threshold changes Inventory business behaviour and therefore must:

- require backend authorization,
- use server-side validation,
- reject unauthorized updates.

The frontend must not be treated as the authorization boundary.

A successful change to the global reorder threshold must trigger authoritative re-evaluation of all existing Inventory records using the same centralized reorder-evaluation logic defined by this ADR.

For each Inventory record after the new threshold is persisted:

- if `quantity <= new threshold` and no active alert exists, create a new active reorder alert,
- if `quantity <= new threshold` and an active alert already exists, update that active alert with the current quantity and threshold,
- if `quantity > new threshold` and an active alert exists, resolve that alert and stop reminders,
- if `quantity > new threshold` and no active alert exists, no alert action is required.

Threshold-change reconciliation must not be implemented as separate duplicated low-stock logic.

The backend must use the same centralized reorder-evaluation mechanism used by normal Inventory quantity updates and checkout/order stock deductions.

---

## 22. Notification Channel Decision

For Sprint 2, reorder alerts use:

**In-application notification only.**

Included:

- in-app low-stock alerts,
- in-app repeated reminders during the approved working period.

Not included:

- email,
- SMS,
- supplier messages,
- push-notification providers,
- automatic external notification services.

External notification channels may be introduced only through a separately approved requirement or architecture decision.

---

## 23. Validation Rules

The backend must validate reorder-threshold configuration and alert-related operations.

At minimum:

- threshold must be numeric,
- threshold must be finite,
- threshold must not be negative,
- threshold modification must be restricted to `INVENTORY_MANAGER`,
- invalid Inventory identifiers must be rejected,
- alert operations must reference an existing Inventory record,
- Inventory quantity must never become negative.

The current approved threshold is `5`.

---

## 24. Reporting Impact

Persisted reorder alerts may provide approved inventory/reporting functionality with information about:

- currently low-stock Inventory items,
- current quantity,
- threshold,
- alert status,
- branch association.

This ADR does not introduce additional analytics, forecasting, procurement reporting, or historical alert analytics.

---

## 25. Procurement Boundary

A reorder alert indicates that stock is low.

It does not automatically mean OSMS will:

- create a supplier order,
- create a purchase order,
- contact a supplier,
- automatically purchase stock,
- perform procurement approval.

Those behaviours require separate approved requirements and implementation.

---

## 26. Consequences

### Positive Consequences

- One clear low-stock rule is used throughout OSMS.
- Threshold configuration is centralized.
- Reorder alerts are persisted.
- Duplicate active alerts are prevented.
- Inventory Managers receive repeated in-app reminders.
- Alerts stop automatically when stock returns above the threshold.
- Manual Inventory updates and checkout use the same reorder logic.
- Backend business logic remains authoritative.

### Negative Consequences

- A reorder-alert persistence mechanism must be implemented.
- A global configuration mechanism is required.
- A periodic reminder process is required.
- Alert lifecycle and scheduling logic add backend complexity.
- One global threshold is less flexible than per-item thresholds.

---

## 27. SRS / SDS / ADR References

### SRS

- FR-005 – Real-Time Stock Synchronization
- FR-006 – Automated Inventory Reorder Alerts
- Inventory Manager responsibilities
- Relevant inventory and reporting access requirements

### SDS

- `Inventory` entity
- `Branch` entity
- Backend Application Layer / business logic
- Authorization Levels / Access Control Matrix
- Input validation and data-integrity requirements

### ADR

- ADR-002 – Authoritative RBAC Role Model
- ADR-003 – Shopping Cart Persistence and Checkout Boundary

---

## 28. Implementation Boundary

DDP-039 defines architecture and business rules only.

This issue must not implement:

- Inventory model changes,
- reorder-alert model code,
- configuration model code,
- alert APIs,
- alert controllers,
- alert services,
- scheduling jobs,
- frontend alert interfaces,
- checkout stock logic,
- email/SMS integration,
- procurement functionality,
- reporting implementation.

Those changes belong to their dedicated Sprint 2 implementation issues.

---

## 29. Review Record

| Date | Reviewer / Group | Result | Notes |
|---|---|---|---|
| 2026-10-09 | Development Team | Agreed | Global threshold 5, `quantity <= 5` low-stock rule, persisted alerts, immediate evaluation, 3-hour in-app reminders during working hours, Inventory Manager threshold ownership. |
| Pending | Peer Reviewer | Pending | Formal pull-request review required before ADR status changes to Accepted. |
| — | — | Supervisor | Pending | — |

---

## 30. Decision Summary

OSMS will use the following authoritative reorder-alert model:

- Global reorder threshold is `5`.
- An Inventory item is low stock when `quantity <= 5`.
- The threshold is stored as one global Inventory configuration value.
- `INVENTORY_MANAGER` may change the global threshold.
- Reorder alerts are persisted.
- Each Inventory record may have only one active reorder alert.
- The Inventory MongoDB `_id` is used as the authoritative Inventory identifier.
- Low-stock evaluation occurs immediately after authoritative stock changes.
- Active low-stock conditions receive in-app reminders every 3 hours during the 8:00 AM–6:00 PM working period.
- Repeated reminders update/use the existing alert rather than creating duplicate active alerts.
- When quantity becomes greater than the threshold, the alert is resolved/inactive and reminders stop.
- `INVENTORY_MANAGER` manages reorder alerts.
- `BRANCH_MANAGER` and `MANAGEMENT` receive approved read-only visibility.
- Alerts are in-app only.
- Email and SMS are outside Sprint 2 scope.
- Manual Inventory updates and checkout/order deductions must use the same centralized reorder-evaluation logic.
- No automatic procurement or supplier-ordering workflow is introduced.
- A successful global threshold change triggers authoritative re-evaluation of all existing Inventory records using the centralized reorder-evaluation logic.
- Resolved alerts remain resolved; if the same Inventory record later becomes low stock again, a new active reorder alert is created for that new low-stock occurrence.
