# ADR-007: Appointment Scheduling, Availability, and Lifecycle Rules

## 1. Title

Appointment Scheduling, Availability, and Lifecycle Rules for OSMS

## 2. Status

**Proposed** — Team business rules have been confirmed. Pending formal peer review and approval.

---

## 3. Context

The Optical Shop Management System (OSMS) requires online appointment booking under FR-007.

The current backend already contains an `Appointment` model with:

- `customerId`,
- `staffId`,
- `dateTime`,
- `status`.

The current controlled appointment statuses are:

- `CONFIRMED`
- `CANCELLED`

The current system also defines the canonical roles:

- `CUSTOMER`
- `SYSTEM_ADMIN`
- `INVENTORY_MANAGER`
- `BRANCH_MANAGER`
- `OPTOMETRIST`
- `MANAGEMENT`
- `SALES_ASSISTANT_CASHIER`

The approved SDS requires appointment bookings to be made at least 24 hours in advance.

However, the current model does not define the complete scheduling rules needed for Sprint 2, including:

- doctor session creation,
- session duration,
- session capacity,
- availability,
- conflict prevention,
- customer cancellation,
- session rescheduling,
- branch restrictions,
- concurrency,
- timezone handling,
- operational appointment-management permissions.

Therefore, one authoritative scheduling and lifecycle design is required before the appointment APIs and frontend booking interface are implemented.

---

## 4. Existing Model and Design Gap

The existing `Appointment` model represents an individual customer booking.

It currently contains:

- customer reference,
- staff / optometrist reference,
- appointment date/time,
- appointment status.

This is sufficient for representing a simple one-customer appointment, but the confirmed business workflow requires a doctor to provide a time period that can accept multiple patients.

Example:

`Dr. Silva – 1:00 PM to 2:00 PM – maximum 10 patients`

The current model does not directly represent:

- session start time,
- session end time,
- session capacity,
- number of available places,
- staff-created doctor sessions.

Therefore, the Sprint 2 architecture requires a controlled extension while preserving the existing Appointment responsibility where possible.

---

## 5. Problem

The appointment architecture must define:

1. booking lead-time,
2. doctor session structure,
3. session capacity,
4. double-booking and overlap rules,
5. doctor selection,
6. branch restriction,
7. availability behaviour,
8. initial appointment status,
9. customer cancellation,
10. staff appointment management,
11. customer rescheduling,
12. staff session rescheduling,
13. past-appointment behaviour,
14. customer visibility,
15. staff visibility,
16. concurrent booking protection,
17. timezone interpretation,
18. communication/reminder boundary,
19. authorization rules,
20. impact on the existing Appointment model.

---

## 6. Confirmed Booking Lead-Time Rule

A customer must make an appointment booking at least:

`24 hours`

before the selected doctor session begins.

The backend is authoritative for this rule.

The frontend may prevent invalid selections for usability, but the backend must validate the rule again when the booking request is submitted.

Conceptually:

`sessionStart >= currentServerTime + 24 hours`

The calculation must use the authoritative appointment timezone defined in this ADR.

---

## 7. Appointment Scheduling Options

### Option A — One Customer per Fixed Time Slot

Each customer receives an individual fixed-duration appointment.

Example:

- 1:00 PM – 1:15 PM
- 1:15 PM – 1:30 PM
- 1:30 PM – 1:45 PM

#### Advantages

- Individual appointment times are very precise.
- Conflict checking is straightforward.

#### Disadvantages

- Does not match the confirmed operational workflow.
- Staff would need to create many small appointment slots.

---

### Option B — Arbitrary Customer Date/Time

Customers select an arbitrary appointment date/time and the backend validates conflicts.

#### Advantages

- Flexible.

#### Disadvantages

- Does not match the staff-managed doctor-session workflow.
- More difficult to control capacity and available periods.

---

### Option C — Staff-Created Doctor Sessions with Capacity

Staff creates an available doctor session with:

- assigned Optometrist,
- date,
- start time,
- end time,
- maximum patient capacity.

Multiple customers may book the same session until its capacity is reached.

Example:

`Dr. Silva – 1:00 PM to 2:00 PM – Capacity 10`

---

## 8. Appointment Scheduling Decision

**Selected: Option C — Staff-Created Doctor Sessions with Capacity.**

A doctor session represents one period during which an Optometrist is available.

At minimum, the scheduling design must support:

- Optometrist reference,
- session date,
- start time,
- end time,
- maximum patient capacity.

Each customer appointment must be associated with an approved doctor session.

The exact MongoDB schema and field names remain part of the backend implementation issue.

---

## 9. Session Persistence Decision

The doctor session must be represented as a persisted scheduling concept separate from an individual customer appointment.

Conceptually:

`Doctor Session`
- Optometrist
- Date
- Start time
- End time
- Capacity

`Appointment`
- Customer
- Doctor session
- Status

This separation avoids duplicating session-level data across every customer booking.

The existing Appointment model should be extended only as required to associate an appointment with its session.

Exact model names, references, indexes, and implementation details remain part of the backend implementation issue.

---

## 10. Session Capacity Rule

Each doctor session has a maximum patient capacity configured by authorized staff.

Example:

`1:00 PM – 2:00 PM`
`Capacity = 10`

The session is available while:

`confirmedBookings < capacity`

The session is full when:

`confirmedBookings >= capacity`

A customer must not be allowed to create a booking that causes the confirmed booking count to exceed the configured capacity.

Cancelled bookings do not consume active session capacity.

---

## 11. Doctor Session Conflict Rule

One Optometrist must not have overlapping doctor sessions.

For example:

`1:00 PM – 2:00 PM`

and

`1:30 PM – 2:30 PM`

for the same Optometrist are not allowed.

Multiple customers within one session are allowed because the session itself has an approved capacity.

Therefore:

- overlapping sessions for the same doctor are prohibited,
- multiple customer bookings inside one session are permitted up to capacity.

---

## 12. Optometrist Selection Rule

Customers may select their preferred Optometrist from available doctor sessions.

Only Staff records with the canonical role:

`OPTOMETRIST`

may be used as appointment providers.

A customer must not be allowed to submit an arbitrary Staff account as an Optometrist.

The backend must validate that the selected session belongs to a valid `OPTOMETRIST`.

---

## 13. Branch Decision

Doctor appointment/channeling functionality is available only at:

**Gampaha Main Branch**

Other OSMS branches do not provide online doctor-channeling functionality under the current approved scope.

The implementation must resolve the approved Gampaha branch through authoritative branch data.

The frontend and backend must not independently invent appointment availability for other branches.

No branch-selection workflow is required for customers because the appointment service is restricted to the Gampaha Main Branch.

---

## 14. Initial Appointment Status

A successfully created appointment is immediately assigned:

`CONFIRMED`

No `PENDING` status is introduced.

The existing controlled statuses remain:

- `CONFIRMED`
- `CANCELLED`

Additional statuses such as:

- `COMPLETED`
- `NO_SHOW`
- `RESCHEDULED`

are not introduced by this ADR.

---

## 15. Customer Cancellation Rule

A customer may cancel their own `CONFIRMED` appointment before the booked doctor session begins.

The customer must not be allowed to cancel the appointment after the session start time has been reached.

Conceptually:

`currentTime < sessionStart` → cancellation allowed

`currentTime >= sessionStart` → customer cancellation not allowed

Cancellation changes the appointment status to:

`CANCELLED`

The appointment record should not be deleted solely because it was cancelled.

---

## 16. Cancellation and Capacity

When a confirmed customer appointment is cancelled before the session begins, that booking no longer consumes session capacity.

Therefore one place becomes available for another customer.

Example:

`Capacity = 10`

`Confirmed bookings = 10`

One customer cancels:

`Confirmed bookings = 9`

The session therefore has one available place again.

---

## 17. Customer Rescheduling Decision

Sprint 2 does not provide direct customer editing of an existing appointment date/time.

If a customer wants a different doctor session, the customer must:

1. cancel the existing appointment before its session begins,
2. select another available session,
3. create a new booking.

Conceptually:

`Cancel old appointment -> Book new appointment`

This avoids introducing a separate customer-side rescheduling lifecycle.

---

## 18. Staff Session Rescheduling

Authorized `SALES_ASSISTANT_CASHIER` staff may change or reschedule a doctor session where required.

Any changed session must still satisfy:

- valid Gampaha Main Branch scope,
- valid Optometrist,
- no overlapping session for the same Optometrist,
- valid start/end time,
- capacity consistency.

When a doctor session is rescheduled, existing customer bookings remain associated with that session and follow the updated session time.

The `SALES_ASSISTANT_CASHIER` must manually contact affected customers and inform them about the changed session.

If an affected customer cannot attend the new session time, the `SALES_ASSISTANT_CASHIER` may make the required appointment adjustment using another valid available session or cancel the booking where necessary.

Any replacement session must still satisfy the approved availability, Optometrist, capacity, branch, and scheduling rules.

The backend implementation must prevent changes that would produce an invalid scheduling state.

---

## 19. Past Appointment Behaviour

When a doctor session begins or passes, OSMS does not automatically introduce a new appointment lifecycle status.

A past appointment may therefore remain:

`CONFIRMED`

unless it was previously:

`CANCELLED`

No automatic transition to `COMPLETED`, `NO_SHOW`, or another new state is required for Sprint 2.

Historical/upcoming presentation may be determined from the session date/time rather than by inventing additional statuses.

---

## 20. Availability Responsibility

Appointment availability is backend-authoritative.

The backend appointment functionality must provide the information necessary for the customer frontend to display bookable doctor sessions.

The frontend may request and display:

- available Optometrists,
- available doctor sessions,
- session date/time,
- remaining availability where appropriate.

The backend must determine whether a session is actually bookable using:

- approved branch,
- approved Optometrist,
- session timing,
- 24-hour booking rule,
- current confirmed booking count,
- configured capacity.

The frontend must not independently decide that a session is available.

---

## 21. Final Booking Revalidation

Displaying a session as available does not guarantee that it remains available until booking is completed.

Immediately before persisting a booking, the backend must revalidate:

- the session still exists,
- the session is still valid,
- the 24-hour rule still passes,
- the session is not full,
- the Optometrist remains valid,
- the booking does not violate any approved scheduling rule.

This protects the system from stale frontend availability information.

---

## 22. Concurrent Booking Rule

Two customers may attempt to book the final available place at nearly the same time.

The required business outcome is:

> The first successfully completed booking receives the final available place. The later competing request must be rejected as unavailable/full.

The backend must enforce session capacity consistently so that concurrent requests cannot cause:

`confirmedBookings > capacity`

The exact locking, transaction, conditional-update, or database-consistency mechanism remains part of the backend implementation issue.

---

## 23. Customer Visibility

An authenticated `CUSTOMER` may access only appointments belonging to that authenticated customer.

The backend must derive customer identity from the authenticated context.

A client-supplied customer identifier must not permit access to another customer's appointments.

Customers may:

- view their own appointments,
- create appointments using available sessions,
- cancel their own eligible appointments before session start.

Customers do not receive staff appointment-management permissions.

---

## 24. Operational Appointment Management Role

The confirmed operational appointment-management role is:

`SALES_ASSISTANT_CASHIER`

For the Gampaha Main Branch appointment workflow, the authorized Sales Assistant / Cashier may:

- create doctor sessions,
- view doctor sessions,
- manage doctor sessions,
- reschedule doctor sessions,
- view customer bookings,
- perform approved booking management,
- cancel bookings where operationally required,
- manually contact customers and Optometrists regarding appointment information.

Backend authorization remains authoritative.

---

## 25. Branch Manager and System Administrator Visibility

`BRANCH_MANAGER` receives read-only visibility of relevant Gampaha appointment sessions and bookings.

Read-only access does not automatically allow the Branch Manager to:

- create doctor sessions,
- change doctor sessions,
- cancel customer appointments,
- change session capacity,
- reschedule sessions.

Operational appointment management remains assigned to `SALES_ASSISTANT_CASHIER`.

`SYSTEM_ADMIN` does not receive normal appointment or doctor-session visibility or operational appointment-management permissions.

The `SYSTEM_ADMIN` role remains responsible for system and account administration and must not receive appointment access merely because it is an administrative system role.

---

## 26. Optometrist System Visibility Decision

Direct Optometrist appointment-management access is not required for the confirmed Sprint 2 operational workflow.

Appointment/session information is communicated to the Optometrist by authorized staff.

Therefore, this ADR does not require a dedicated Optometrist appointment-management screen or direct doctor-side booking-management workflow.

The `OPTOMETRIST` role remains the authoritative identity used to determine which doctor is assigned to a session.

---

## 27. ADR-002 Authorization Impact

ADR-002 currently identifies `OPTOMETRIST` as the primary appointment-management role and identifies `BRANCH_MANAGER` for approved reminder operations.

The confirmed operational workflow for DDP-045 differs from that earlier general mapping.

This ADR therefore defines a controlled, more specific authorization decision for Sprint 2 appointment scheduling:

- `SALES_ASSISTANT_CASHIER` → operational session and appointment management,
- `CUSTOMER` → own booking/view/cancellation functionality,
- `BRANCH_MANAGER` → read-only Gampaha appointment visibility,
- `OPTOMETRIST` → assigned clinical provider identity; no direct appointment-management workflow required.
- `SYSTEM_ADMIN` → no normal appointment/session visibility or operational appointment-management permission.

This change must be reflected in the project's final authorization documentation/SDS through the normal change-management process.

---

## 28. Timezone Decision

The authoritative timezone for OSMS appointment scheduling is:

`Asia/Colombo`

All business-rule interpretation must use Sri Lanka local time.

This includes:

- doctor session start/end times,
- 24-hour advance-booking validation,
- customer cancellation cutoff,
- availability evaluation,
- past/upcoming appointment classification.

MongoDB/JavaScript `Date` values may be persisted using their normal UTC instant representation, but backend and frontend appointment business rules must interpret/display the appointment using `Asia/Colombo`.

The frontend and backend must not interpret the same session as different local times.

---

## 29. Communication / Reminder Decision

The project team has approved a change from automated appointment reminders to manual staff contact for the current appointment workflow.

Therefore, this ADR does not require:

- automated email reminders,
- automated SMS reminders,
- scheduled notification jobs,
- notification-provider integration,
- automated reminder delivery.

Instead, authorized operational staff manually contact customers and Optometrists where appointment communication is required.

Cancelled appointments must not be treated as valid upcoming appointments for manual contact purposes.

This is a controlled change from the previously documented FR-008 automated-reminder direction and must be reflected in the relevant requirements/design documentation.

Dedicated automated-reminder issues must not be implemented unchanged if they conflict with this approved decision.

---

## 30. No Appointment Payment

The appointment workflow defined by this ADR is a reservation workflow only.

No payment is required when a customer books a doctor session.

This ADR does not introduce:

- online appointment payment,
- payment gateway integration,
- appointment deposits,
- doctor-payment calculation.

These concerns are outside DDP-045.

---

## 31. Existing Appointment Model Impact

The existing Appointment model must be preserved where possible.

Existing concepts remain valid:

- Customer relationship,
- Staff / Optometrist relationship,
- appointment status,
- timestamps.

However, the confirmed session-based design requires a persisted doctor-session concept.

The backend implementation may therefore require a controlled extension such as:

- a dedicated appointment-session entity/model,
- a session reference from Appointment,
- supporting session timing/capacity fields.

Exact schema field names, database indexes, and migration strategy are implementation concerns for the dedicated backend issue.

This ADR does not implement those code changes.

---

## 32. Validation Requirements

Backend validation must enforce at minimum:

- appointment booking is at least 24 hours before session start,
- only the Gampaha Main Branch supports appointment sessions,
- session start time is before end time,
- capacity is a valid positive whole number,
- selected provider has canonical role `OPTOMETRIST`,
- one Optometrist cannot have overlapping sessions,
- confirmed bookings cannot exceed session capacity,
- cancelled bookings do not consume capacity,
- customer cancellation is permitted only before session start,
- customer may access only their own appointments,
- unauthorized roles cannot manage sessions,
- appointment times are interpreted consistently using `Asia/Colombo`.

Frontend validation may improve usability but must not replace backend validation.

---

## 33. Security and Authorization Boundary

All protected appointment operations require authenticated backend authorization.

The frontend may hide or display functionality according to role, but frontend visibility is not an authorization control.

The backend must ensure:

- customers cannot access another customer's appointment,
- customers cannot create sessions,
- customers cannot assign arbitrary staff as an Optometrist,
- unauthorized staff cannot modify appointment sessions,
- read-only Branch Manager access does not become modification access.

---

## 34. Consequences

### Positive Consequences

- Appointment behaviour matches the confirmed real-world Gampaha workflow.
- Staff can define doctor availability using practical sessions.
- Multiple patients can book one doctor session up to an approved capacity.
- Session capacity is centrally enforced.
- Cancelled appointments release capacity.
- Customers can choose their preferred Optometrist.
- Concurrent overbooking is explicitly prevented.
- No unnecessary appointment lifecycle statuses are introduced.
- Backend remains authoritative for availability and security.
- Appointment times use one authoritative timezone.

### Negative Consequences / Trade-offs

- The existing Appointment model alone is not sufficient for the confirmed session-based workflow.
- A persisted doctor-session concept must be introduced during backend implementation.
- Session-capacity and concurrency enforcement add backend complexity.
- Restricting appointment service to Gampaha means other branches cannot offer online appointment booking under the current scope.
- Manual customer/doctor communication requires staff operational effort.
- ADR-002/SRS/SDS documentation requires alignment with the updated appointment-management and reminder decisions.

---

## 35. SRS / SDS / ADR References

### SRS

- FR-007 – Online Appointment Booking Calendar
- FR-008 – Appointment Reminder requirement, subject to the approved manual-contact change
- Customer appointment-booking workflow
- Optometrist / Medical Staff responsibilities
- Relevant staff and branch responsibilities

### SDS

- Interactive Appointment Calendar
- `Appointment` entity
- Appointment Booking Sequence
- Appointment Booking Flow
- Authorization Levels / Access Control Matrix
- Server-Side Business Rule Validation
- 24-hour advance-booking requirement

### ADR

- ADR-002 – Authoritative RBAC Role Model

---

## 36. Implementation Boundary

DDP-045 defines architecture and business rules only.

This issue must not implement:

- appointment routes,
- appointment controllers,
- appointment services,
- session database models,
- Appointment model changes,
- React booking pages,
- calendar components,
- notification jobs,
- email/SMS delivery,
- payment processing,
- clinical prescription functionality,
- attendance tracking,
- doctor-payment calculation.

Those changes belong to their respective implementation issues.

---

## 37. Review Record

| Date | Reviewer / Group | Result | Notes |
|---|---|---|---|
| 2026-10-10 | Development Team | Agreed | Confirmed Gampaha-only appointment service, staff-created capacity-based doctor sessions, customer-selected Optometrist, 24-hour booking rule, cancellation before session start, Sales Assistant/Cashier operational management, Branch Manager read-only visibility, manual communication, and Asia/Colombo timezone. |
| Pending | Peer Reviewer | Pending | Formal pull-request review required before ADR status changes to Accepted. |
| — | — | Supervisor | Pending | Documentation alignment required for changed reminder and authorization decisions. |

---

## 38. Decision Summary

OSMS will use the following authoritative appointment scheduling model:

- Appointment/channeling functionality is available only at the Gampaha Main Branch.
- Authorized `SALES_ASSISTANT_CASHIER` staff create and manage doctor sessions.
- A doctor session has an Optometrist, date, start time, end time, and patient capacity.
- Customers choose their preferred available Optometrist/session.
- Only canonical `OPTOMETRIST` Staff records may be assigned as doctors.
- One Optometrist cannot have overlapping sessions.
- Multiple customers may book the same session up to its capacity.
- Customers must book at least 24 hours before session start.
- A successful booking receives `CONFIRMED` status immediately.
- Existing statuses remain `CONFIRMED` and `CANCELLED`.
- Customers may view only their own appointments.
- Customers may cancel before session start.
- Cancellation releases one place back to the session.
- Customers reschedule by cancelling the old booking and creating a new booking.
- Authorized staff may manage/reschedule doctor sessions.
- Past appointments do not automatically receive a new status.
- `BRANCH_MANAGER` has read-only visibility of relevant Gampaha appointments/sessions.
- `SYSTEM_ADMIN` has no normal appointment/session visibility or operational appointment-management permission.
- Direct Optometrist appointment-management access is not required in the current workflow.
- Backend provides authoritative availability and performs final booking revalidation.
- Concurrent booking must never exceed session capacity; the first successful booking receives the final place.
- Appointment scheduling uses `Asia/Colombo`.
- Automated reminders are not part of the approved current workflow; staff communication is manual.
- No appointment payment is required.
- A persisted doctor-session concept is required as a controlled extension to the current Appointment architecture.
