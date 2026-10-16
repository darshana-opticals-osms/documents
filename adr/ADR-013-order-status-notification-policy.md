# ADR-013: Authoritative Order-Status Notification Policy

## 1. Title

Authoritative Order-Status Notification Architecture, Trigger Lifecycle, and Channel Delivery Policy for OSMS

## 2. Status

**Proposed** — Submitted for architectural and team review under Issue #23 on 2026-10-16.

---

## 3. Context

The Optical Shop Management System (OSMS) is an enterprise retail optical and clinical eyecare management platform for Darshana Opticals (Pvt) Ltd. 

Under **FR-014 (Real-Time Order Tracking Status)** and **FR-004 (Shopping Cart and Checkout)**, customers place prescription eyewear and optical orders, track real-time fulfillment progression, and pick up assembled glasses at designated branch locations.

The approved Software Design Specification (SDS v1.0 Section 1.2.3 & 3.2 – *Communication & Notifications Architecture*) outlines the utilization of AWS communication services (AWS SES for email and AWS SNS for SMS) to deliver automated transactional updates to customers.

Earlier architectural decisions established fundamental domain boundaries:
- **ADR-002 (Authoritative RBAC Role Model)**: Centralizes user authorization into 5 canonical roles, assigning order fulfillment progression duties to `BRANCH_ASSISTANT` and `BRANCH_MANAGER`.
- **ADR-004 (Order Lifecycle and Tracking Status Model)**: Formally defines the authoritative six-stage order state machine: `ORDER_CREATED`, `PROCESSING`, `READY_FOR_PICKUP`, `PICKED_UP`, `COMPLETED`, and `CANCELLED`.
- **ADR-009 (Appointment Communication Scope Decision)**: Confirmed that appointment reminders operate via manual staff communication, thereby leaving the automated notification provider infrastructure uncoupled from appointment reminder schedules.
- **ADR-011 (Secure Payment Gateway Integration Architecture)**: Directs order transition from `ORDER_CREATED` to `PROCESSING` upon verified PayHere IPN webhook confirmation.
- **ADR-012 (Authoritative Immutable Audit-Trail Architecture)**: Establishes compliance auditing rules for critical financial, inventory, and clinical data modifications.

### The Design Gap

While ADR-004 dictates how an order transitions through its lifecycle, OSMS does not currently define:
1. Which exact order lifecycle events warrant automated customer notifications.
2. The authoritative notification channel and external provider for Sprint 2.
3. How notifications are dispatched without introducing latency or failure coupling into core order-processing transactions.
4. How duplicate notifications are prevented when order state updates or network operations retry.
5. The data privacy, minimization, and redaction boundaries applicable to customer notifications.
6. The relationship between transient delivery records and permanent compliance audit logs.

Without a canonical ADR, individual developers risk implementing conflicting notification triggers, exposing sensitive clinical/payment data in emails, or allowing third-party email downtime to crash order checkout transactions.

---

## 4. Problem Statement

OSMS requires one authoritative, centralized notification architecture that:
- Ensures customers receive timely, relevant, and privacy-safe transactional notifications regarding meaningful order milestones.
- Strictly observes the canonical order statuses and legal transitions defined by ADR-004 without inventing synthetic intermediary states.
- Prevents external notification network outages (e.g., AWS SES downtime or throttling) from blocking or rolling back internal order status updates.
- Eliminates duplicate notifications caused by API retries, network glitches, or webhook replays.
- Minimizes customer communication fatigue by selecting only high-value lifecycle transitions rather than spamming every minor database modification.
- Keeps notification logic fully decoupled from clinical data, payment card secrets, and appointment scheduling rules.

---

## 5. Canonical Order Notification Trigger Matrix

OSMS strictly adheres to the canonical Order lifecycle defined in **ADR-004**. Notifications are triggered **only after** an authoritative status transition has successfully committed to the database.

### 5.1 Authoritative Status-Transition Notification Matrix

| Source Status | Target Status | Trigger Event / Meaning | Notify Customer? | Channel | Value to Customer / Rationale |
|---|---|---|---|---|---|
| `*` | `ORDER_CREATED` | Order checkout finalized; stock reserved (awaiting payment). | **No** | N/A | For online checkout, payment is pending. Sending an email before payment capture creates confusion if the customer abandons or cancels payment within the 30-minute reservation window. |
| `ORDER_CREATED` | `PROCESSING` | Verified payment confirmed via PayHere IPN (ADR-011) or staff POS confirmation. | **Yes** (Milestone 1) | Email (AWS SES) | **Order Confirmation & Receipt**: Confirms payment receipt, order number, items purchased, and fulfillment initiation. |
| `PROCESSING` | `READY_FOR_PICKUP` | Optical workshop / branch staff complete lens cutting, mounting, and QA. | **Yes** (Milestone 2) | Email (AWS SES) | **Action Required**: Alerts customer that their glasses are assembled and ready for collection at the assigned branch, including branch hours and address. |
| `READY_FOR_PICKUP` | `PICKED_UP` | Customer physically collects order at branch counter. | **No** | N/A | Frontline counter interaction; customer is physically present at the shop and receives paper receipt/glasses. Immediate email is redundant. |
| `PICKED_UP` | `COMPLETED` | Order finalized and closed out in system. | **Yes** (Milestone 3) | Email (AWS SES) | **Fulfillment Complete**: Formal closing notification thanking the customer, providing digital order summary, and warranty/care instructions. |
| `ORDER_CREATED` | `CANCELLED` | Payment reservation expired (30-min window) or explicit customer cancellation. | **Yes** (Milestone 4) | Email (AWS SES) | **Cancellation Notice**: Informs customer that the pending order was cancelled and inventory reservation was released. |
| `PROCESSING` | `CANCELLED` | Managerial exception / stock defect cancellation after payment. | **Yes** (Milestone 5) | Email (AWS SES) | **Critical Exception**: Alerts customer of cancellation, reason, and offline refund reconciliation processing. |

### 5.2 Invalid Transitions and Boundary Invariants
- **Invalid Transitions**: If an order status change request violates the ADR-004 state machine (e.g., `ORDER_CREATED` $\rightarrow$ `COMPLETED`), the Order domain rejects the request. **No notification is ever dispatched for rejected or failed state changes.**
- **Terminal States**: `COMPLETED` and `CANCELLED` are terminal. No transitions out of these states can ever occur or trigger notifications.
- **Execution Order Invariance**: The notification trigger is invoked **downstream** of successful database persistence. A notification must never claim an order status has updated if the database transaction failed.

---

## 6. Recipient Resolution and Contact Policy

### 6.1 Authoritative Recipient Derivation
- **Backend-Enforced Relationship**: The notification recipient is resolved exclusively from the database relationship:
  $$\text{Order.customerId} \longrightarrow \text{Customer.email}$$
- **Client Untrusted**: Frontline staff or client applications submitting an order status update (`PATCH /api/v1/orders/:id/status`) cannot supply a custom recipient email in the request payload. The backend always queries the registered `Customer` record.

### 6.2 Missing or Invalid Contact Information Policy
If a customer account lacks an email address or contains a malformed email:
1. The notification dispatcher marks the notification attempt as `PERMANENTLY_FAILED` with reason `MISSING_RECIPIENT_EMAIL`.
2. A warning is logged to the operational logger (`Winston`).
3. **The core Order status update remains 100% successful**.
4. The system **never** attempts to guess an email address, never sends to a fallback developer inbox, and never blocks order progression.

---

## 7. Notification Channel and Provider Strategy

### 7.1 Sprint 2 Channel Selection: Transactional Email
For OSMS Sprint 2, **Transactional Email** is selected as the authoritative customer communication channel.
- **Rich Informational Context**: Optical orders involve multiple line items (frame models, lens coatings, prescription linkages, branch addresses). Email provides a structured, formatted medium for complete transactional clarity.
- **Cost Effectiveness**: Email incurs minimal operating expense compared to per-message carrier SMS tariffs.
- **No SMS Character Constraints**: Avoids multi-part SMS concatenation, carrier delivery filtering, or truncated messages.
- **Future SMS Extension (Sprint 3+)**: The notification service utilizes a generic channel adapter interface (`INotificationChannelAdapter`). SMS delivery via AWS SNS can be plugged in for urgent pickup reminders without rearchitecting the order notification pipeline.

### 7.2 Provider Selection: AWS Simple Email Service (SES)
In accordance with SDS Section 3.2, **Amazon SES** is the authoritative external delivery provider:
- Direct support for DKIM, SPF, and DMARC domain verification under `darshanaopticals.lk`.
- High delivery reputation and built-in bounce/complaint handling.
- Environment-isolated credentials via AWS IAM.

### 7.3 Decoupling from Appointment Reminders
Per ADR-009, appointment communication is manual for staff. Order notifications **do not** interact with or depend on appointment reminder services, cron jobs, or models. Reusable AWS SDK client configuration is shared at the infrastructure library layer (`libs/aws/sesClient.js`) without coupling domain business logic.

---

## 8. Delivery Architecture Evaluation and Decision

Three dispatch architectures were evaluated:

### Option A — Direct Synchronous Dispatch
The order controller/service updates MongoDB and immediately calls AWS SES via HTTP before returning the API response.
- *Pros*: Simple, procedural flow.
- *Cons*: Severe anti-pattern. External AWS SES network latency (200ms–2000ms) directly degrades Order API responsiveness. If AWS SES times out or errors, developers face a dilemma: roll back a valid physical order change, or leave the customer with a 500 error on a successful order.

### Option B — Internal Asynchronous Domain-Event Dispatch (Selected)
The Order service updates MongoDB within its domain boundary and emits an internal in-process domain event (`order.status_changed`) via a centralized Node.js event emitter. The `OrderNotificationListener` handles event processing asynchronously:
- The Order API responds to staff/webhook immediately (<20ms).
- The notification worker formats the email, checks idempotency, calls AWS SES, and records the delivery result asynchronously.
- *Pros*: Zero latency impact on Order APIs; complete isolation between core order fulfillment and third-party email delivery; zero additional infrastructure dependencies (no Redis/RabbitMQ required for Sprint 2 scale); fully testable via mock event listeners.
- *Cons*: If the Node.js process crashes at the exact millisecond between event emission and SES call, the email could be lost (acceptable trade-off for transactional updates with web tracking fallback).

### Option C — Durable Transactional Outbox / Worker Queue
The Order service writes an `OrderNotificationTask` document into a MongoDB outbox collection within the same transaction as the order status update. A background polling worker scans and processes tasks with at-least-once delivery guarantees.
- *Pros*: Guaranteed persistence across sudden server crashes.
- *Cons*: High complexity; requires background polling intervals or change streams; excessive overhead for non-critical email updates where customers have active real-time web tracking (FR-014).

### Decision
**Option B (Internal Asynchronous Domain-Event Dispatch)** is selected. It completely protects Order transactional integrity, keeps API latency minimal, avoids complex message broker infrastructure, and fits the operational scale of OSMS Sprint 2.

---

## 9. Order Transaction vs. Notification Failure Boundary

A fundamental architectural principle of OSMS:
> **Notification delivery failure must NEVER roll back, fail, or invalidate an authoritative Order status update.**

- If branch staff mark glasses as `READY_FOR_PICKUP`, that physical and operational state is true. If AWS SES returns a `503 Service Unavailable` or the customer's mailbox is full, the order remains `READY_FOR_PICKUP`.
- The customer can view their real-time order status at any time via the Customer Web Tracking Portal (**FR-014 / DDP-044**). Notification is an auxiliary channel, not the single source of truth.

---

## 10. Duplicate Prevention and Idempotency Strategy

To prevent spamming customers if an API request is retried or a webhook is re-delivered, OSMS enforces deterministic notification idempotency:

### 10.1 Deterministic Idempotency Key
Every notification task computes a unique idempotency key:
$$\text{idempotencyKey} = \text{"ORDER\_NOTIF\_"} + \text{orderId} + \text{"\_"} + \text{targetStatus}$$
*Example*: `ORDER_NOTIF_670c538f9b1e2a001f3e4b11_READY_FOR_PICKUP`

### 10.2 Enforcement Mechanism
Before dispatching an email, the notification service checks if a notification record with that `idempotencyKey` already exists with status `SENT` or `PROCESSING`.
- If found: The duplicate event is silently dropped, and an audit log note is recorded.
- Because ADR-004 enforces a strictly forward-progressing state machine where an order cannot re-enter an identical status, each status milestone generates at most **one** customer notification.

---

## 11. Notification Persistence Model

To support operational delivery monitoring, debugging, and duplicate suppression without polluting core domain models, OSMS introduces a dedicated, lightweight MongoDB collection: `order_notifications`.

### 11.1 Document Schema (`order_notifications`)

```json
{
  "_id": "<ObjectId>",
  "orderId": "<ObjectId — ref: Order>",
  "customerId": "<ObjectId — ref: Customer>",
  "recipientEmail": "<string — e.g. 'c***@example.com'>",
  "triggerStatus": "<string — e.g. 'READY_FOR_PICKUP'>",
  "idempotencyKey": "<string — unique index>",
  "channel": "EMAIL",
  "deliveryStatus": "<string — 'PENDING' | 'SENT' | 'FAILED' | 'SKIPPED'>",
  "provider": "AWS_SES",
  "providerMessageId": "<string | null — AWS SES MessageId>",
  "attempts": "<number — default 1>",
  "lastAttemptAt": "<ISODate>",
  "errorCode": "<string | null — e.g. 'SES_BOUNCE', 'INVALID_EMAIL'>",
  "createdAt": "<ISODate>",
  "updatedAt": "<ISODate>"
}
```

*Indexes*:
- `{ idempotencyKey: 1 }` (Unique)
- `{ orderId: 1, createdAt: -1 }`
- `{ deliveryStatus: 1, createdAt: -1 }`

This schema intentionally stores only delivery metadata. Full raw email HTML bodies and unnecessary PII are excluded.

---

## 12. Retry and Failure Policy

### 12.1 Categorization of Failures
1. **Transient Failures (Retryable)**:
   - AWS SES rate throttling (`ThrottlingException`).
   - Transient TCP/DNS timeouts or HTTP 5xx gateway errors.
2. **Permanent Failures (Non-Retryable)**:
   - Invalid email syntax or unverified sender identity (`InvalidParameterException`).
   - Hard bounce or blocked recipient mailbox.
   - Missing customer email address.

### 12.2 Bounded Retry Protocol
- **Max Retries**: Up to 3 attempts.
- **Backoff Strategy**: Exponential backoff with jitter (1s $\rightarrow$ 4s $\rightarrow$ 16s).
- **Terminal State**: If all 3 attempts fail, `deliveryStatus` transitions to `FAILED`. No further automatic retries are attempted.
- **No False Positives**: An unconfirmed or failed attempt is **never** recorded as `SENT`.

---

## 13. Message Content, Minimization, and Privacy Boundaries

### 13.1 Permitted Transactional Content
Order emails must strictly contain the minimum operational information required for customer clarity:
- Customer friendly name (first name).
- Order reference code (e.g., `ORD-2026-1014-001`).
- Human-readable order status update (e.g., *"Your glasses are assembled and ready for collection"*).
- Assigned branch name, address, and operating hours (for `READY_FOR_PICKUP`).
- Secure link to the web tracking portal: `https://darshanaopticals.lk/orders/:id/track`.

### 13.2 Strictly Prohibited Content
In compliance with medical data privacy (ADR-001) and payment security (ADR-011):
- **Zero Clinical Measurements**: Emails must never include sphere, cylinder, axis, visual acuity, or optometrist diagnostic remarks.
- **Zero Payment Secrets**: Emails must never include credit card numbers (PAN), CVVs, PayHere merchant secrets, or raw webhook hashes.
- **Zero Authentication Credentials**: Emails must never include temporary passwords or JWTs.
- **Zero Promotional / Marketing Bloat**: Emails must not contain unsolicited product advertisements, loyalty upsells, or marketing banners.

### 13.3 Centralized Template Ownership
All email formatting is owned by a single service: `OrderNotificationTemplateService`. Order controllers and adapters must not assemble ad-hoc HTML strings.

---

## 14. Authorization, Trust Boundaries, and Manual Resend Policy

### 14.1 Server-Controlled Triggers Only
- No public or customer-facing API endpoint can directly command the server to send an order notification (e.g., `POST /api/v1/notifications/send` is strictly forbidden).
- Notifications are strictly triggered as a side effect of authorized order status transitions performed by `BRANCH_ASSISTANT` or `BRANCH_MANAGER` under ADR-002.

### 14.2 Manual Resend Boundary for Sprint 2
- **Decision**: **No manual resend capability or UI buttons exist in Sprint 2**.
- Frontend interfaces must not include "Resend Notification" or "Email Customer" buttons.
- Customers can verify their order status 24/7 on the real-time tracking interface (FR-014). Adding manual resend introduces risk of staff spamming customers and is unnecessary for Sprint 2.

---

## 15. Relationship to ADR-012 Compliance Audit Trail

- **Notification Events are NOT ADR-012 Audit Events**: Sending an automated email is an auxiliary customer notification, not a financial, medical, or stock modification.
- **No Duplicate Auditing**: The underlying Order status update (e.g., marking order `READY_FOR_PICKUP`) is already recorded in the authoritative audit trail under ADR-012. Generating a second immutable audit record for the corresponding email dispatch is redundant and explicitly rejected.
- **Operational Logs Only**: Notification delivery results are recorded in `order_notifications` and Winston operational logs for 30–90 days debugging.

---

## 16. Environment-Specific Behavior and Testing Safety

To guarantee that automated tests and local developer environments never transmit real emails to customers:

| Environment | Notification Provider Implementation | Behavior |
|---|---|---|
| **Development** | `MockEmailNotificationAdapter` | Logs formatted email to terminal/Winston; records simulated success in `order_notifications`. |
| **CI / Automated Tests** | `MockEmailNotificationAdapter` | In-memory stub. Asserts that `order.status_changed` event fired with correct payload without making external network calls. |
| **Staging / QA** | `AwsSesEmailAdapter` (Sandbox) | Routes all emails strictly to a whitelisted internal developer mailbox (`qa-test@darshanaopticals.lk`). |
| **Production** | `AwsSesEmailAdapter` (Production) | Transmits live transactional emails to verified customer email addresses using DKIM-signed domain. |

---

## 17. Security and Safe Logging

- **Credential Isolation**: AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `SES_FROM_EMAIL`) are loaded exclusively from backend environment variables. Never hardcoded or committed.
- **PII-Sanitized Logging**: Log messages recording notification dispatch must mask customer email addresses (e.g., `j***@gmail.com`) and log only the `orderId`, `idempotencyKey`, and `deliveryStatus`.

---

## 18. Consequences

### 18.1 Positive Consequences
- **Architectural Determinism**: Establishes one authoritative trigger matrix mapped directly to ADR-004 canonical statuses.
- **High System Reliability**: Decoupling email delivery from database updates ensures third-party AWS outages never block retail store operations or customer checkout.
- **Zero Duplicate Spam**: Idempotency keys prevent double-messaging customers during network or webhook retries.
- **Privacy by Design**: Guarantees that sensitive medical refraction data and financial secrets never leak into customer email inboxes.
- **Zero Cost Overhead**: Leverages lightweight Node.js event listeners and affordable AWS SES without requiring complex message queue clusters.

### 18.2 Negative / Neutral Trade-offs
- **Asynchronous In-Memory Window**: In the rare event of an ungraceful server termination (e.g., kill -9) immediately following an order transition, an email event could be dropped. Customers still have real-time web tracking on the web portal.
- **No Manual Resend UI**: Staff cannot manually click a button to re-trigger an email if a customer claims they didn't receive it (mitigated by directing customers to self-service online order tracking).

---

## 19. References

- **SRS v1.0**: FR-004 (Shopping Cart & Checkout), FR-014 (Real-Time Order Tracking Status).
- **SDS v1.0**: Section 1.2.3, 3.2 (Communication & Notifications Architecture, AWS SES/SNS).
- **ADR-002**: Authoritative RBAC Role Model.
- **ADR-004**: Order Lifecycle and Tracking Status Model.
- **ADR-009**: Appointment Communication Scope Decision (manual staff contact).
- **ADR-011**: Secure Payment Gateway Integration and Payment Lifecycle Architecture.
- **ADR-012**: Authoritative Immutable Audit-Trail Architecture.

---

## 20. Review Record

| Date | Reviewer | Role | Notes / Comments |
|---|---|---|---|
| 2026-10-16 | Senior Full Stack Architect | Author / Technical Lead | Initial architectural specification satisfying all AC1–AC57 requirements for Issue #23. |
