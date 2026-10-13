# ADR-009: Appointment Communication and Automated Reminder Scope Decision

## 1. Title

Appointment Communication and Automated Reminder Scope Decision for OSMS

## 2. Status

**Proposed** — Pending formal review and approval.

---

## 3. Context

The approved Software Requirements Specification (SRS) defines FR-008 as automated appointment reminders sent to customers by email or SMS. The approved Software Design Specification (SDS) carries that direction into the backend integration-services design, the AWS SES/SNS communication and notification architecture, and the authorization model that assigns ownership of FR-008 to the Branch Manager.

The project team subsequently agreed that the current appointment workflow will use manual staff communication rather than an automated reminder subsystem. This decision is already recorded in ADR-007, Section 29, **Communication / Reminder Decision**.

This ADR formalizes the appointment-communication consequences of that decision and records a controlled, appointment-specific deviation from the original automated-reminder direction. The approved SRS and SDS remain preserved as historical baselines. This ADR does not claim to alter those baselines before completing the project's required review and change-control process.

---

## 4. Problem

The original automated-reminder direction would require decisions and implementation for:

- automated email and/or SMS delivery,
- external notification providers,
- reminder timing and scheduling,
- scheduled jobs or queue workers,
- duplicate prevention,
- delivery-attempt persistence,
- retry and final-failure handling,
- delivery-status tracking,
- provider credentials and operational monitoring.

Those capabilities conflict with the currently agreed manual communication scope. Leaving both directions active would allow teams to introduce unnecessary reminder models, services, infrastructure, APIs, or user interfaces and would create uncertainty about appointment roles and lifecycle behavior.

An authoritative and narrowly scoped decision is therefore required to keep the current appointment implementation consistent with ADR-007.

---

## 5. Considered Options

### Option A — Automated Appointment Reminders

Send appointment reminders through email and/or SMS using an external provider and an automated scheduling mechanism.

#### Advantages

- Reduces routine staff effort.
- Can provide consistent scheduled delivery.
- Can support automated delivery tracking and analytics.

#### Disadvantages

- Conflicts with the current manual-contact decision in ADR-007.
- Requires provider integration, scheduling, persistence, retry, monitoring, and security work.
- Introduces architecture and operating costs that are not required by the current appointment scope.

### Option B — Hybrid Communication

Use manual staff contact together with automated reminders.

#### Advantages

- Retains human contact while adding automated coverage.
- May provide redundancy if one communication method is unsuccessful.

#### Disadvantages

- Retains nearly all automated-reminder complexity.
- Creates a risk of duplicate or inconsistent customer communication.
- Requires unapproved policies for timing, channel, delivery ownership, and staff follow-up.

### Option C — Manual Staff Communication Only

Authorized branch staff handle appointment-related communication without an automated appointment-reminder subsystem.

#### Advantages

- Aligns with ADR-007's current appointment workflow.
- Avoids unnecessary scheduling, provider, persistence, retry, and delivery-tracking infrastructure.
- Keeps appointment booking independent of an external notification provider.
- Preserves the existing appointment lifecycle without adding communication states.

#### Disadvantages

- Communication depends on staff availability and operational follow-through.
- Automated delivery guarantees, analytics, and status tracking are unavailable.
- Future automation will require a separate approved architectural decision and implementation scope.

---

## 6. Decision

**Selected: Option C — Manual Staff Communication Only.**

The following rules define the current OSMS appointment-communication architecture:

- The current appointment workflow uses manual staff communication.
- `BRANCH_ASSISTANT` is the operational appointment-management role under ADR-002 and ADR-007.
- Successfully created appointments immediately receive `CONFIRMED` status as defined by ADR-007.
- No additional appointment confirmation, communication, reminder, delivery, or contact-result status is introduced.
- OSMS does not require automated appointment email or SMS reminders in the current scope.
- No appointment reminder scheduling service, cron job, queue worker, provider scheduling feature, or cloud scheduling job is required.
- No AWS SES or AWS SNS integration is required for appointment reminders.
- No dedicated reminder collection, notification collection, delivery-attempt record, reminder marker, notification status, or reminder history is required.
- No automated reminder retry mechanism, duplicate-prevention mechanism, or delivery-status tracking is required.
- Manual communication does not automatically change an Appointment or Doctor Session status.
- Appointment cancellation, capacity, rescheduling, timing, status, and lifecycle rules remain governed by ADR-007.
- The manual operational procedure is managed by authorized branch staff and is outside the architectural scope of this ADR.

This decision does not define staff call scripts, contact frequency, number of contact attempts, operational checklists, or call-result recording.

---

## 7. Supersession and Document Precedence

The following earlier directions are affected only for the current **appointment reminder** scope:

- SRS FR-008 — automated appointment reminders dispatched to customers through email or SMS.
- SRS Section 6.2, Clinical Appointment Scheduling — automated notifications and reminders intended to reduce no-shows.
- SRS appointment-booking use case and activity flow — scheduling or dispatching an automated reminder after booking.
- SRS Section 16.3, Software Interface Requirements — SMS/email gateways for automated reminders and booking confirmations.
- SDS Section 1.2.1, Detailed Description of Layers — backend integration services for FR-008 SMS/email notifications.
- SDS Section 1.3, Cloud-Native Architecture, **Communication & Notifications (AWS SES/SNS)** — SES or SNS for automated appointment reminders and order updates.
- SDS Section 1.5.1, Scalability — asynchronous handling of automated SMS/email reminders.
- SDS Section 6.2, Authorization Levels — Branch Manager ownership of automated appointment reminders under FR-008.
- SDS Section 7.3, Network Diagram, **Connectivity Logic** — outbound Twilio/Nodemailer integration for automated SMS/email reminders.

The approved SRS and SDS PDFs remain unchanged and are retained as historical baselines. This Proposed ADR records the intended architectural deviation but does not automatically override formal requirements before approval.

ADR-007 Section 29 already establishes manual appointment communication and remains authoritative for the accepted current appointment workflow. ADR-002 remains authoritative for canonical role names and role boundaries.

Once this ADR is accepted through the project's review and change-control process, it must be read together with ADR-007 as the authoritative current appointment-communication decision. Any future formal SRS or SDS revision should incorporate the accepted deviation.

This precedence is narrow. It does not supersede order updates or any other notification requirement outside appointment reminders.

---

## 8. Security and Privacy

Existing authentication, authorization, privacy, and audit requirements remain applicable.

- Customer contact information must remain protected by server-side authorization controls.
- Appointment-related customer data must not be exposed to unauthorized users.
- Staff access must follow the canonical roles and appointment-specific permissions defined by ADR-002 and ADR-007.
- No external appointment notification provider receives customer data under this decision.
- No automated reminder delivery logs or sensitive notification payloads are introduced.
- Manual communication does not authorize unnecessary copying, logging, or disclosure of customer or clinical information.

This ADR introduces no new security infrastructure.

---

## 9. Failure Isolation

An unsuccessful manual contact attempt must not automatically:

- cancel or delete an Appointment,
- change an Appointment status,
- change appointment ownership,
- alter a Doctor Session,
- consume or release session capacity,
- modify clinical data.

Existing appointment lifecycle rules remain authoritative. No notification-service dependency is introduced into appointment booking, cancellation, reassignment, session management, or availability processing.

---

## 10. Consequences

### Positive Consequences

- The appointment architecture remains simpler.
- No appointment reminder provider integration is required.
- No scheduling or delivery-tracking subsystem is required.
- No reminder persistence or retry workflow is required.
- Appointment booking is not operationally dependent on an external notification service.
- Customer appointment data is not transmitted to an external reminder provider.
- The implementation scope remains aligned with ADR-007.

### Negative Consequences / Trade-offs

- Appointment communication depends on branch staff.
- Automated reminder delivery guarantees are not provided.
- Automated delivery analytics, delivery receipts, and failure tracking are unavailable.
- The system does not enforce a staff contact schedule or confirm that manual contact occurred.
- Future automated reminders will require a separate approved architectural decision, clarified business policy, and dedicated implementation scope.

---

## 11. Scope Boundaries

This ADR applies only to appointment communication and the automated-reminder scope previously associated with FR-008.

It does not disable, supersede, redesign, or decide:

- order notifications or order-status updates,
- payment notifications,
- inventory or reorder alerts,
- low-stock notifications,
- marketing or promotional communications,
- chatbot messages,
- other unrelated OSMS notification features.

Those capabilities remain governed by their own requirements, ADRs, and implementation issues.

This ADR does not implement automated or manual communication functionality and does not introduce reminder APIs, staff communication screens, notification providers, credentials, database models, schedulers, or infrastructure.

---

## 12. Documentation Impact

- The original approved SRS and SDS PDFs are preserved unchanged by this decision record.
- Their appointment-specific automated-reminder direction is documented here as a controlled deviation pending approval.
- ADR-007 remains authoritative for appointment scheduling, lifecycle, cancellation, rescheduling, capacity, status, and timezone rules.
- ADR-002 remains authoritative for canonical role definitions and general authorization boundaries.
- This ADR is limited to the appointment-communication architecture.
- Future formal SRS/SDS revisions may incorporate this decision through the project's change-control process after acceptance.

---

## 13. References

### SRS

- FR-007 — Online appointment booking for eye examinations
- FR-008 — Automated appointment reminders through email or SMS
- Section 6.2 — Clinical Appointment Scheduling
- Appointment-booking use case and activity flow
- Section 16.3 — Software Interface Requirements

### SDS

- Section 1.2.1 — Detailed Description of Layers, Application Layer integration services
- Section 1.3 — Cloud-Native Architecture, Communication & Notifications (AWS SES/SNS)
- Section 1.5.1 — Scalability, asynchronous reminder processing
- Section 6.2 — Authorization Levels and Updated Access Control Matrix
- Section 7.3 — Network Diagram, Connectivity Logic

### ADRs

- ADR-002 — Authoritative RBAC Role Model
- ADR-007 — Appointment Scheduling, Availability, and Lifecycle Rules, especially Section 29

### Development Standards

- Significant deviations from the approved SDS must be recorded through an ADR.
- Business rules must remain traceable to approved project documentation.
- YAGNI — functionality not required by the approved current scope should not be introduced.
- Protected data and unnecessary PII must not be exposed or logged.

---

## 14. Review Record

| Date | Reviewer / Group | Result | Notes |
|---|---|---|---|
| — | — | Pending | Formal review and approval required. |

---

## 15. Decision Summary

OSMS will use manual staff communication for the current appointment workflow. The canonical `BRANCH_ASSISTANT` role handles operational appointment coordination according to ADR-002 and ADR-007. No automated appointment reminder provider, scheduler, persistence model, retry workflow, delivery state, or tracking subsystem is introduced. Appointment lifecycle behavior remains governed by ADR-007. The original SRS/SDS automated-reminder direction remains preserved as a historical baseline and becomes a controlled, appointment-specific deviation only after this ADR completes formal approval and change control.
