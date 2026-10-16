# ADR-012: Authoritative Immutable Audit-Trail Architecture

## 1. Title

Authoritative Immutable Audit-Trail Architecture for Critical Stock, Financial, and Clinical Actions in OSMS

## 2. Status

**Proposed** — Submitted for architectural and security review under Issue #22 on 2026-10-16.

---

## 3. Context

The Optical Shop Management System (OSMS) is an enterprise healthcare and retail management platform for Darshana Opticals (Pvt) Ltd. The platform handles operations spanning clinical eyecare, prescription tracking, inventory supply chain, retail optical sales, and financial checkout transactions.

The approved Software Design Specification (SDS v1.0 Section 6.3 – *Data Protection Approach*) establishes mandatory non-negotiable security requirements:
- **Secure Audit Trails**: Detailed audit trails must be implemented for critical stock adjustments, financial transactions, and clinical data modifications.
- **Immutability**: Audit logs must be tamper-resistant, append-only, and protected against unauthorized modification or deletion.
- **Retention**: Critical audit logs must be preserved for a **minimum of seven (7) years** to satisfy Sri Lankan medical record-keeping standards, commercial tax statutory obligations (Inland Revenue Act), and electronic transaction evidence requirements.
- **Principle of Least Privilege**: Audit log access must be strictly segregated from general operational data and restricted exclusively to authorized administrative/compliance roles.
- **PII and Sensitive Data Minimization**: Audit logs must not become an uncontrolled backdoor repository for payment card credentials, passwords, cryptographic keys, or unredacted medical notes.

---

## 4. Problem and Existing Architecture Gap

Current OSMS backend services store transactional data across MongoDB collections (`inventories`, `orders`, `payments`, `prescriptions`, `users`, `branches`). While these domain records provide baseline operational state, they exhibit critical architectural gaps when evaluated against compliance audit requirements:

1. **Mutable Domain State**: Standard domain documents rely on Mongoose timestamps (`createdAt`, `updatedAt`). When a document is modified (e.g., inventory count updated, payment state transitioned), previous values are overwritten, leaving only the latest state without an immutable delta history.
2. **Lack of Centralized Audit Event Schema**: Existing services implement ad-hoc console logs or localized status-history arrays without a uniform cross-domain schema specifying actor identity, target resource, action taxonomy, and correlation tracking.
3. **Application Logs are Not Audit Trails**: Winston and Morgan server logs capture ephemeral HTTP traffic, error stack traces, and operational diagnostics. They are subject to debug log-level rotations, lack cryptographic integrity, contain unindexed raw strings, and are typically purged within 30 to 90 days.
4. **Vulnerability to Unauthorized Modification**: A database administrator or compromised privileged account with standard MongoDB write access could theoretically update or drop operational log entries if audit records reside in unprotected mutable collections.
5. **Absence of 7-Year Retention & Archival Strategy**: No automated lifecycle exists to guarantee that high-volume transactional audit logs can be retained cost-effectively for seven years without degrading primary database performance.

Therefore, OSMS requires one authoritative, centralized, immutable audit-trail architecture to govern critical inventory, financial, and clinical operations.

---

## 5. Audit Scope and Event Catalogue

To adhere to the Principle of Least Privilege and avoid database bloat, OSMS strictly distinguishes between **auditable critical events** (which carry financial, legal, medical, or security liability) and **routine diagnostic actions**.

### 5.1 Critical Event Catalogue Matrix

The following table defines the authoritative catalogue of auditable events across OSMS domains:

| Domain | Action Type | Trigger / Business Operation | Actor Category | Target Resource | Failure Policy |
|---|---|---|---|---|---|
| **Inventory** | `INVENTORY_STOCK_MANUAL_INCREASE` | Manual stock addition / batch restock | Authenticated Staff (`BRANCH_MANAGER`) | `Inventory` | Fail-Closed |
| **Inventory** | `INVENTORY_STOCK_MANUAL_DECREASE` | Manual write-off, damage, or discrepancy adjustment | Authenticated Staff (`BRANCH_MANAGER`) | `Inventory` | Fail-Closed |
| **Inventory** | `INVENTORY_STOCK_CHECKOUT_DEDUCT` | Confirmed customer order checkout stock deduction | System Automated Process / Customer Checkout | `Inventory` | Fail-Closed (Transactional) |
| **Inventory** | `INVENTORY_STOCK_RESERVATION_RELEASE` | Order cancellation or payment timeout stock restoration | System Expiry Cron / Staff Cancellation | `Inventory` | Fail-Closed |
| **Inventory** | `INVENTORY_PRICE_UPDATED` | Authorized unit retail price modification | Authenticated Staff (`BRANCH_MANAGER`) | `Inventory` | Fail-Closed |
| **Inventory** | `INVENTORY_REORDER_THRESHOLD_UPDATED` | Low-stock threshold modification per ADR-006 | Authenticated Staff (`BRANCH_MANAGER`) | `Inventory` | Fail-Closed |
| **Financial** | `PAYMENT_SESSION_INITIATED` | Customer initiates PayHere checkout session | Authenticated Customer (`CUSTOMER`) | `Payment` / `Order` | Fail-Closed |
| **Financial** | `PAYMENT_IPN_VERIFIED` | Cryptographically verified server-to-server IPN receipt | External Provider (`PAYHERE`) | `Payment` / `Order` | Fail-Closed |
| **Financial** | `PAYMENT_TRANSACTION_COMPLETED` | Verified funds capture (`status_code = 2`) | External Provider / System | `Payment` / `Order` | Fail-Closed (Transactional) |
| **Financial** | `PAYMENT_TRANSACTION_FAILED` | Gateway decline, card error, or signature failure | External Provider / System | `Payment` / `Order` | Fail-Open with Alert |
| **Financial** | `PAYMENT_TRANSACTION_CANCELLED` | Customer explicitly aborts gateway checkout | Authenticated Customer / Provider | `Payment` / `Order` | Fail-Open with Alert |
| **Financial** | `PAYMENT_TRANSACTION_REFUNDED` | Verified offline refund recorded by Management | Authenticated Staff (`BRANCH_MANAGER` / Executive) | `Payment` / `Order` | Fail-Closed |
| **Clinical** | `PRESCRIPTION_RECORD_CREATED` | Authorized clinical examination recorded per ADR-001 | Authenticated Optometrist (`OPTOMETRIST`) | `Prescription` | Fail-Closed (Transactional) |
| **Clinical** | `PRESCRIPTION_RECORD_ARCHIVED` | Soft-archival of prescription record per ADR-001 | Authenticated Optometrist (`OPTOMETRIST`) | `Prescription` | Fail-Closed |
| **Security/Admin** | `STAFF_USER_PROVISIONED` | Administrative creation of privileged staff account | Authenticated Admin (`SYSTEM_ADMIN`) | `User` | Fail-Closed |
| **Security/Admin** | `USER_ROLE_PERMISSIONS_MODIFIED` | Canonical RBAC role reassignment per ADR-002 | Authenticated Admin (`SYSTEM_ADMIN`) | `User` | Fail-Closed |
| **Security/Admin** | `USER_ACCOUNT_STATUS_TOGGLED` | Account suspension, lockout, or reactivation | Authenticated Admin (`SYSTEM_ADMIN`) | `User` | Fail-Closed |
| **Loyalty** | `LOYALTY_POINTS_MANUALLY_ADJUSTED` | Authorized managerial loyalty point adjustment | Authenticated Staff (`BRANCH_MANAGER`) | `Loyalty` / `Customer` | Fail-Closed |

*Excluded from Audit Trail Scope*:
- Routine HTTP `GET` queries (unless accessing audit trails themselves).
- Public product catalog browsing and cart manipulations before checkout.
- Customer chatbot conversation transcripts (governed separately by ADR-010).
- Appointment availability checks and tentative booking draft selections.

---

## 6. Canonical Audit Event Schema

All audit records written to the authoritative audit store must adhere strictly to the following normalized JSON schema (Mongoose `AuditEventSchema`):

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "OSMS_AuditEvent",
  "type": "object",
  "required": [
    "auditEventId",
    "schemaVersion",
    "timestamp",
    "actionType",
    "actor",
    "target",
    "outcome",
    "correlation",
    "integrity"
  ],
  "properties": {
    "auditEventId": {
      "type": "string",
      "format": "uuid",
      "description": "Unique immutable identifier for this audit event (UUID v4)."
    },
    "schemaVersion": {
      "type": "integer",
      "enum": [1],
      "description": "Version of the audit schema specification."
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "Authoritative UTC timestamp generated by backend server."
    },
    "actionType": {
      "type": "string",
      "enum": [
        "INVENTORY_STOCK_MANUAL_INCREASE",
        "INVENTORY_STOCK_MANUAL_DECREASE",
        "INVENTORY_STOCK_CHECKOUT_DEDUCT",
        "INVENTORY_STOCK_RESERVATION_RELEASE",
        "INVENTORY_PRICE_UPDATED",
        "INVENTORY_REORDER_THRESHOLD_UPDATED",
        "PAYMENT_SESSION_INITIATED",
        "PAYMENT_IPN_VERIFIED",
        "PAYMENT_TRANSACTION_COMPLETED",
        "PAYMENT_TRANSACTION_FAILED",
        "PAYMENT_TRANSACTION_CANCELLED",
        "PAYMENT_TRANSACTION_REFUNDED",
        "PRESCRIPTION_RECORD_CREATED",
        "PRESCRIPTION_RECORD_ARCHIVED",
        "STAFF_USER_PROVISIONED",
        "USER_ROLE_PERMISSIONS_MODIFIED",
        "USER_ACCOUNT_STATUS_TOGGLED",
        "LOYALTY_POINTS_MANUALLY_ADJUSTED",
        "AUDIT_TRAIL_ACCESSED"
      ]
    },
    "actor": {
      "type": "object",
      "required": ["actorType", "actorId", "actorRole"],
      "properties": {
        "actorType": {
          "type": "string",
          "enum": ["AUTHENTICATED_USER", "SYSTEM_PROCESS", "EXTERNAL_PROVIDER"]
        },
        "actorId": {
          "type": "string",
          "description": "MongoDB ObjectId string of staff/customer, or system process name, or provider identifier."
        },
        "actorRole": {
          "type": "string",
          "enum": [
            "SYSTEM_ADMIN",
            "BRANCH_MANAGER",
            "OPTOMETRIST",
            "BRANCH_ASSISTANT",
            "CUSTOMER",
            "SYSTEM",
            "PAYHERE_GATEWAY"
          ]
        },
        "ipAddress": {
          "type": "string",
          "description": "Client IP address (masked to subnet e.g. 192.168.1.xxx for privacy compliance)."
        },
        "userAgent": {
          "type": "string",
          "description": "Sanitized client user agent header."
        }
      }
    },
    "target": {
      "type": "object",
      "required": ["targetType", "targetId"],
      "properties": {
        "targetType": {
          "type": "string",
          "enum": ["INVENTORY", "PAYMENT", "ORDER", "PRESCRIPTION", "USER", "LOYALTY", "AUDIT_LOG"]
        },
        "targetId": {
          "type": "string",
          "description": "Authoritative primary key (MongoDB ObjectId or unique business number)."
        },
        "targetIdentifier": {
          "type": "string",
          "description": "Human-readable reference where applicable (e.g. Order Number, SKU code, Customer Code)."
        }
      }
    },
    "branchId": {
      "type": ["string", "null"],
      "description": "Associated Branch ObjectId where applicable; null for system-wide events."
    },
    "outcome": {
      "type": "string",
      "enum": ["SUCCESS", "FAILURE"],
      "description": "Outcome of the attempted business operation."
    },
    "correlation": {
      "type": "object",
      "required": ["requestId", "correlationId"],
      "properties": {
        "requestId": { "type": "string", "format": "uuid" },
        "correlationId": { "type": "string", "format": "uuid" }
      }
    },
    "stateDelta": {
      "type": ["object", "null"],
      "properties": {
        "before": { "type": ["object", "null"] },
        "after": { "type": ["object", "null"] }
      },
      "description": "Specific changed fields only; never full raw entity copies."
    },
    "safeMetadata": {
      "type": "object",
      "description": "Sanitized, domain-specific contextual metadata."
    },
    "integrity": {
      "type": "object",
      "required": ["eventHash", "prevEventHash"],
      "properties": {
        "eventHash": {
          "type": "string",
          "description": "SHA-256 cryptographic digest of this event's canonical fields."
        },
        "prevEventHash": {
          "type": "string",
          "description": "SHA-256 cryptographic digest of the immediately preceding event in the chain."
        }
      }
    }
  }
}
```

---

## 7. Actor and Target Representation

### 7.1 Backend-Authoritative Actor Derivation
- **Client Untrusted**: The frontend client is strictly prohibited from supplying actor identity.
- **JWT Context**: For all authenticated API calls, `actor.actorId` and `actor.actorRole` are extracted exclusively by backend authentication middleware (`req.user.id`, `req.user.role`) verified against the cryptographically signed JWT.
- **System Actions**: Background cron jobs (e.g., payment expiry, reorder alert scanning) record `actorType: 'SYSTEM_PROCESS'`, `actorId: 'CRON_RESERVATION_EXPIRY'`, `actorRole: 'SYSTEM'`.
- **External Webhooks**: Verified PayHere IPN callbacks record `actorType: 'EXTERNAL_PROVIDER'`, `actorId: 'PAYHERE_IPN_SERVICE'`, `actorRole: 'PAYHERE_GATEWAY'`.

### 7.2 Target Identification
Every audit event must pin the authoritative resource target using:
- `targetType`: An explicit controlled entity enum (e.g., `INVENTORY`, `PAYMENT`, `PRESCRIPTION`).
- `targetId`: The immutable primary key (`_id.toString()`).
- `targetIdentifier`: Optional business identifier (e.g., `ORD-2026-1014-001`, `SKU-RAYBAN-501`) to support rapid operational correlation without joining collections.

---

## 8. Data Minimization and Redaction Policy

Audit records must never duplicate unnecessary Personally Identifiable Information (PII) or sensitive operational credentials.

### 8.1 Strictly Prohibited Audit Fields
Under no circumstances may the following fields be written into `stateDelta` or `safeMetadata`:
1. **Passwords and Hashes**: Plaintext passwords, bcrypt/argon2 hashes, password reset tokens.
2. **Authentication Secrets**: JWT tokens, bearer tokens, refresh tokens, session secrets.
3. **Payment Credentials**: Primary Account Numbers (PAN / 16-digit card numbers), CVV/CVC security codes, card expiration dates, bank PINs.
4. **Gateway Secrets**: PayHere merchant secret, HMAC keys, API secret keys.
5. **Raw Clinical Measurements**: Raw refraction data (Sph, Cyl, Axis, VA, Add) must **not** be duplicated in the audit store. The clinical audit record notes only that a prescription was created/archived for a customer by an optometrist.

### 8.2 Safe Metadata Examples by Domain

#### Inventory Adjustment Example
```json
{
  "stateDelta": {
    "before": { "quantity": 15 },
    "after": { "quantity": 10 }
  },
  "safeMetadata": {
    "delta": -5,
    "reason": "CHECKOUT_DEDUCTION",
    "orderId": "670c538f9b1e2a001f3e4b11"
  }
}
```

#### Financial Transaction Example
```json
{
  "stateDelta": {
    "before": { "status": "pending" },
    "after": { "status": "completed" }
  },
  "safeMetadata": {
    "amount": 14500.00,
    "currency": "LKR",
    "paymentMethod": "online_gateway",
    "providerTransactionId": "320025981244",
    "ipnStatusCode": "2"
  }
}
```

#### Clinical Prescription Example
```json
{
  "stateDelta": null,
  "safeMetadata": {
    "prescriptionId": "670c538f9b1e2a001f3e4c99",
    "customerId": "670c538f9b1e2a001f3e4a05",
    "recordedByOptometristId": "670c538f9b1e2a001f3e4012",
    "action": "PRESCRIPTION_ISSUED"
  }
}
```

---

## 9. Storage Architecture Evaluation and Decision

Four architectural options were evaluated for storing OSMS audit records:

### Option A — Dedicated Unconstrained MongoDB Collection
A standard MongoDB collection (`audit_events`) with default read/write operations.
- *Pros*: Easiest to implement; standard Mongoose models.
- *Cons*: Fails compliance; any database user with write permissions can execute `db.audit_events.deleteMany({})` or alter records; no tamper evidence.

### Option B — Dedicated Append-Only Collection with Cryptographic Hash Chaining and Database Privilege Segregation (Selected)
A dedicated MongoDB collection (`audit_trails`) fortified by four complementary defense layers:
1. **Application-Level Immutability**: Mongoose schema prevents updates (`immutable: true`), and the internal `AuditService` exposes only an `insert` interface (no update, replace, or delete methods).
2. **Database-Level Role Restrictions**: Production application connections use a restricted MongoDB user whose role permits `insert` and `find` on `audit_trails`, with `delete`, `update`, `dropCollection`, and `dropDatabase` explicitly denied.
3. **Tamper-Evident SHA-256 Hash Chaining**: Each audit record stores `eventHash = SHA256(prevEventHash + canonicalEventPayload)`. Any post-facto modification or deletion of a past record breaks the cryptographic chain and is detected during integrity verification audits.
4. **Automated Tiered Archival**: Active records remain in MongoDB for 2 years (hot tier); records aged 2 to 7+ years are batched, GZIP-compressed, digitally signed, and offloaded to encrypted immutable cold storage (e.g. AWS S3 Glacier with Object Lock Compliance Mode / GCP Cloud Storage with Bucket Lock).
- *Pros*: Zero external infrastructure overhead for primary write latency; fits cleanly within the MERN architecture; provides mathematical tamper-evidence; fully satisfies 7-year retention.
- *Cons*: Requires initial setup of restricted MongoDB roles and periodic chain-verification jobs.

### Option C — External Real-Time WORM / Immutable Cloud Logging (AWS CloudTrail / Google Cloud Logging / Datadog)
Streaming every audit event directly to an external managed compliance log.
- *Pros*: Off-site immutability managed by cloud vendor.
- *Cons*: Introduces external network latency and outbound network dependencies during critical transactions; high cost for a 7-year retention footprint; complex local mock setup for development and CI testing.

### Option D — Full Event Sourcing and CQRS (e.g., EventStoreDB)
Re-architecting the entire OSMS database to derive all business state exclusively from an immutable event stream.
- *Pros*: Complete system history is naturally an event log.
- *Cons*: Massive violation of YAGNI; extreme architectural overhaul of existing Sprint 1/2 MERN codebase; unjustifiable operational complexity.

### Decision
**Option B is selected**. It provides robust, mathematically verifiable tamper-resistance, enforces strict append-only semantics at both application and database layers, and achieves cost-effective 7-year statutory retention without introducing external runtime failure dependencies during checkout.

---

## 10. Immutability, Append-Only Controls, and Tamper-Evidence

### 10.1 Multi-Layer Immutability Controls

```text
+-------------------------------------------------------------------------+
|                        OSMS Audit Immutability                          |
+-------------------------------------------------------------------------+
| 1. Code Level      | AuditService exposes ONLY recordEvent().           |
|                    | No update(), delete(), or remove() methods exist.  |
+--------------------+----------------------------------------------------+
| 2. Mongoose Schema | All schema attributes set to { immutable: true }.  |
|                    | Strict schema validation rejects unknown fields.   |
+--------------------+----------------------------------------------------+
| 3. Database Role   | MongoDB app user role grants ONLY insert & find.   |
|                    | db.createRole({ role: "auditAppendOnly", ... })    |
|                    | Drop, delete, and update permissions REVOKED.      |
+--------------------+----------------------------------------------------+
| 4. Cryptographic   | SHA-256 Hash Chain:                                |
|    Integrity       | Event[N].hash = SHA256(Event[N-1].hash + payload)  |
|                    | Tampering with any historical block breaks chain.  |
+-------------------------------------------------------------------------+
```

### 10.2 Cryptographic Hash Chaining Specification

Each audit record calculates `eventHash` using SHA-256:

$$\text{payloadString} = \text{auditEventId} + \text{schemaVersion} + \text{timestamp} + \text{actionType} + \text{actor.actorId} + \text{target.targetType} + \text{target.targetId} + \text{JSON.stringify}(\text{safeMetadata})$$

$$\text{eventHash} = \text{SHA256}(\text{prevEventHash} + \text{payloadString})$$

- For the very first audit record in the system, `prevEventHash` is fixed to the canonical **Genesis Hash**:
  `"0000000000000000000000000000000000000000000000000000000000000000"`.
- A scheduled background verification job (`auditIntegrityChecker`) periodically verifies the blockchain-like hash consistency. Any missing sequence or mismatched digest immediately triggers a high-severity P1 security alert.

---

## 11. Central Audit Service and Write Consistency Boundary

### 11.1 Centralized Service Interface
Domain controllers must never interact directly with the `AuditTrail` MongoDB collection. All interactions must pass through the singleton `AuditService`:

```typescript
interface IAuditService {
  recordEvent(params: {
    actionType: AuditActionType;
    actor: {
      actorType: ActorType;
      actorId: string;
      actorRole: string;
      ipAddress?: string;
      userAgent?: string;
    };
    target: {
      targetType: TargetType;
      targetId: string;
      targetIdentifier?: string;
    };
    branchId?: string | null;
    outcome: 'SUCCESS' | 'FAILURE';
    correlation: {
      requestId: string;
      correlationId: string;
    };
    stateDelta?: {
      before: Record<string, unknown> | null;
      after: Record<string, unknown> | null;
    };
    safeMetadata?: Record<string, unknown>;
    session?: ClientSession; // Optional MongoDB transaction session
  }): Promise<string>; // Returns auditEventId
}
```

### 11.2 Atomicity and Multi-Document Transactions
For critical operations that alter financial, inventory, or clinical state:
1. **Transactional Execution**: Where a replica set is active (staging, production), the domain operation and the `AuditService.recordEvent` call execute within the **same MongoDB ClientSession transaction**:
   ```javascript
   const session = await mongoose.startSession();
   session.startTransaction();
   try {
     await inventoryModel.updateOne({ _id: id }, { $inc: { quantity: -deductQty } }, { session });
     await auditService.recordEvent({ ..., session });
     await session.commitTransaction();
   } catch (error) {
     await session.abortTransaction();
     throw error;
   } finally {
     session.endSession();
   }
   ```
2. **Atomicity Guarantee**: If the audit event write fails, the business state modification is rolled back. A critical business action can never commit without its corresponding audit record.

### 11.3 Audit Failure Policy
- **Fail-Closed Operations** (Stock reductions, checkout deductions, price changes, verified payment completions, refunds, clinical prescriptions, admin user provisioning):
  - If audit persistence fails, the entire transaction **aborts and throws a 500 Internal Server Error**.
  - Rationale: Financial and medical accountability takes precedence over optimistic execution.
- **Fail-Open Operations with Alerting** (Payment transaction failures, user checkout cancellations, login attempt logging):
  - If audit write fails for an already failed gateway transaction, the operation logs a critical operational alert (`AUDIT_WRITE_FAILED`) via Winston to disk/monitoring, but does not crash the response to the user.

---

## 12. Correlation, Timestamps, and Time Standards

### 12.1 Request & Correlation Tracking
- **`requestId`**: Generated as a UUID v4 by Express middleware (`correlationMiddleware`) at the boundary of every incoming HTTP request. Attached to `req.headers['x-request-id']` and returned in the HTTP response header.
- **`correlationId`**: Inherited from parent workflows (e.g., across checkout -> payment initiate -> IPN webhook -> order completion). Allows auditors to reconstruct the entire multi-step lifecycle of an order across hours or days.

### 12.2 Authoritative Timestamp and Time Standard
- **Server Authority**: The audit timestamp is exclusively generated by the backend runtime via `new Date().toISOString()`. Client-supplied timestamps are strictly rejected.
- **Canonical Storage**: Timestamps are stored in UTC ISO 8601 format (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- **Presentation**: Any internal reporting or compliance review display converts UTC timestamps to Sri Lanka Standard Time (Asia/Colombo, `UTC+05:30`).

---

## 13. Retention, Archival, and Disaster Recovery

### 13.1 Seven-Year Statutory Retention Plan
The SDS mandates a minimum 7-year retention period. To optimize database storage and performance, OSMS implements a two-tier data lifecycle:

```text
+-------------------------------------------------------------------------+
|                  OSMS 7-Year Audit Data Lifecycle                       |
+-------------------------------------------------------------------------+
| Years 0 - 2 : HOT TIER (Active MongoDB Collection)                      |
| - Fast indexing, operational queryability, tamper-hash validation.      |
| - Queryable by authorized compliance officers via backend API.          |
+-------------------------------------------------------------------------+
                                    |
                                    v (Automated Yearly Archival Job)
+-------------------------------------------------------------------------+
| Years 3 - 7+ : COLD TIER (Encrypted Immutable WORM Archive)             |
| - Batched yearly JSONL archives, compressed with GZIP.                  |
| - Sealed with SHA-256 digital signature manifest.                       |
| - Stored in Cloud Object Lock (WORM storage with 7-year retention lock).|
| - Evicted from hot MongoDB collection with tombstone record.            |
+-------------------------------------------------------------------------+
```

### 13.2 Archival Verification & Retrieval
- Archival jobs generate a cryptographic manifest (`archive_2026_manifest.sha256`) containing individual archive file hashes.
- Retrieval of archived records for legal/tax discovery is performed through a controlled administrative utility that decrypts the archive, verifies the digital signature, and restores the records to a temporary isolated audit query database.

### 13.3 Relationship to Backups
Standard daily database backups (e.g., MongoDB Atlas automated snapshots, `mongodump`) protect against hardware failure and disaster recovery. However:
- A backup is **not** an immutable audit trail. Backups can be overwritten, restored out of order, or wiped during snapshot retention cycles.
- The authoritative audit trail resides in the append-only `audit_trails` collection and cold WORM archives, which are backed up alongside domain data.

---

## 14. Access Control, Querying, and Pagination

### 14.1 Principle of Least Privilege for Audit Access
Audit trail querying exposes sensitive operational and forensic information. Access is governed as follows:

| Role | Audit Read Access | Permitted Scope |
|---|---|---|
| `SYSTEM_ADMIN` | **Granted (Full)** | System-wide search, compliance review, integrity verification. |
| `BRANCH_MANAGER` | **Denied** | Must rely on sanitized operational reports (FR-012), not raw audit trails. |
| `OPTOMETRIST` | **Denied** | Clinical staff review clinical history via FR-002/FR-013, not audit trails. |
| `BRANCH_ASSISTANT` | **Denied** | Strictly zero access. |
| `CUSTOMER` | **Denied** | Strictly zero access. |

### 14.2 Audit Read Auditing (Meta-Auditing)
Whenever a `SYSTEM_ADMIN` queries the audit trail API (`GET /api/v1/admin/audit-logs`), the action is itself recorded as an immutable audit event:
`actionType: 'AUDIT_TRAIL_ACCESSED'`.

### 14.3 Query Filters and Bounded Pagination
To prevent Denial of Service and excessive memory consumption, the audit query API enforces:
- **Mandatory Date Bounding**: Maximum query date range of 90 days per request.
- **Supported Query Filters**:
  - `startDate` / `endDate` (ISO 8601)
  - `actionType` (Controlled enum)
  - `targetType` (Controlled enum)
  - `targetId` (Exact string match)
  - `actorId` (Exact string match)
  - `branchId` (Exact string match)
- **Bounded Pagination**:
  - `page`: default `1`
  - `limit`: default `20`, maximum enforced cap of `100`.
- **Deterministic Sorting**:
  - Primary sort: `timestamp: -1` (chronological descending).
  - Secondary tie-breaker: `_id: -1`.

---

## 15. Operational Logging vs. Audit Trail vs. Domain History

To maintain clear separation of concerns, OSMS strictly distinguishes these three concepts:

```text
+------------------+-----------------------+-------------------------+-------------------------+
| Feature          | Operational Logs      | Domain History          | Audit Trail             |
+------------------+-----------------------+-------------------------+-------------------------+
| Primary Purpose  | Debugging, monitoring,| Business workflows,     | Legal, regulatory, and  |
|                  | diagnostics, errors.  | customer history view.  | forensic accountability.|
+------------------+-----------------------+-------------------------+-------------------------+
| Primary Tool     | Winston / Morgan      | Domain Collections      | AuditService /          |
|                  | (Rotating log files)  | (Order, Payment, etc.)  | AuditTrail Collection   |
+------------------+-----------------------+-------------------------+-------------------------+
| Immutability     | Low (Rotated/purged)  | Medium (Insert-only for | High (Cryptographic     |
|                  |                       | Prescriptions, updates) | hash chain, no deletes) |
+------------------+-----------------------+-------------------------+-------------------------+
| Retention        | 30 to 90 days         | Active lifetime         | Minimum 7 Years         |
+------------------+-----------------------+-------------------------+-------------------------+
| User Visibility  | Developers / DevOps   | Staff & Customers       | System Admin / Auditor  |
+------------------+-----------------------+-------------------------+-------------------------+
| Sensitive Data   | Sanitized error stack | Business attributes     | Redacted safe metadata  |
+------------------+-----------------------+-------------------------+-------------------------+
```

Domain models must never derive their current operational state from the audit trail (no reverse dependency). Event Sourcing is explicitly rejected as unnecessary complexity.

---

## 16. Performance and Indexing Strategy

To guarantee that audit writing adds minimal overhead (<15ms) to API response times, the `audit_trails` collection requires the following compound indexes:

1. **Chronological Sorting & Event Lookups**:
   `{ timestamp: -1, actionType: 1 }`
2. **Resource-Specific Forensic Auditing**:
   `{ "target.targetType": 1, "target.targetId": 1, timestamp: -1 }`
3. **Actor Activity Auditing**:
   `{ "actor.actorId": 1, timestamp: -1 }`
4. **Correlation & Workflow Traceability**:
   `{ "correlation.correlationId": 1 }`
5. **Branch Forensic Querying**:
   `{ branchId: 1, timestamp: -1 }`

Write performance is optimized by keeping `stateDelta` and `safeMetadata` strictly lean and avoiding unnecessary unbounded sub-document growth.

---

## 17. Environment and Testing Strategy

### 17.1 Environment Differentiation
- **Development / Local**: Uses local MongoDB instance. In-memory or Docker MongoDB handles transactional audit writes. Hash chaining is enabled and validated.
- **CI / Automated Testing**: Test suites run against isolated test databases. Tests verify that audit records are created upon critical operations and that transactions abort when an audit write fails. No external cloud WORM storage dependencies exist in CI.
- **Staging / Production**: Full database-level role segregation (`auditAppendOnly` user), replication, automated hash chain verification jobs, and scheduled cloud archival.

### 17.2 Automated Verification Requirements
Future backend implementation issues (DDP-066) must implement automated unit and integration tests verifying:
1. **Append-Only Rejection**: Direct attempts to execute `updateOne`, `deleteMany`, or `findOneAndUpdate` on `AuditTrail` throw authorization/schema errors.
2. **Transactional Rollback**: Simulating a database failure on `auditService.recordEvent()` during checkout guarantees that the corresponding `inventory.updateOne()` is rolled back.
3. **Redaction Sanitization**: Test asserting that payloads containing `password`, `cvv`, `cardNumber`, or `merchantSecret` are completely stripped from saved audit records.
4. **Hash Chain Continuity**: Test asserting that generating 10 sequential audit events produces valid cryptographic linkages, and modifying an intermediate record triggers an integrity alert.

---

## 18. Consequences

### 18.1 Positive Consequences
- **Full Regulatory Compliance**: Fulfills the SDS Section 6.3 requirement for secure, immutable audit logs with a 7-year retention lifecycle.
- **Robust Tamper Resistance**: Multi-layer security (Mongoose immutability, MongoDB role segregation, and SHA-256 hash chaining) guarantees forensic defensibility.
- **Transactional Consistency**: Multi-document transactions guarantee that stock deductions, payment confirmations, and clinical records never occur without an audit record.
- **Data Privacy by Design**: Strict sanitization and minimization protocols prevent PII, payment cards, and raw clinical details from leaking into audit storage.
- **Separation of Concerns**: Decouples application error logging, domain business history, and compliance auditing into dedicated, purpose-built mechanisms.

### 18.2 Negative / Neutral Trade-offs
- **Write Overhead**: Adding an audit write within a MongoDB transaction introduces a slight computational overhead (estimated 10–25ms per critical transaction), which is acceptable for OSMS throughput.
- **Storage Growth**: Retaining logs for 7 years requires periodic maintenance and cloud storage costs (mitigated by compressing cold archives into low-cost WORM storage).
- **Administrative Maintenance**: Requires initial MongoDB database role provisioning and scheduled cron jobs for hash chain integrity scanning.

---

## 19. References

- **SRS v1.0**: Relevant security, traceability, and medical accountability requirements.
- **SDS v1.0 Section 6.3**: *Data Protection Approach* — Secure Audit Trails, 7-year retention, stock, financial, and clinical operations.
- **ADR-001**: Clinical Prescription Data Storage Design (dedicated insert-only collection).
- **ADR-002**: Authoritative RBAC Role Model (canonical roles, `SYSTEM_ADMIN` privilege).
- **ADR-006**: Inventory Reorder-Alert Rules and Persistence Model (stock adjustments and reorder states).
- **ADR-011**: Secure Payment Gateway Integration and Payment Lifecycle Architecture (canonical payment states, verified PayHere IPN).

---

## 20. Review Record

| Date | Reviewer | Role | Notes / Comments |
|---|---|---|---|
| 2026-10-16 | Senior Full Stack Architect | Author / Technical Lead | Initial architectural draft satisfying all AC1–AC48 requirements for Issue #22. |
