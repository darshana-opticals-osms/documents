# ADR-008: Inventory Image Upload and Cloudinary Storage Architecture

## 1. Title

Inventory Image Upload Architecture and Cloudinary Storage Design for OSMS

## 2. Status

**Accepted** — Approved through formal peer review and supervisor sign-off on 2026-10-13.

---

## 3. Context

The Optical Shop Management System (OSMS) supports an optional `imageUrl` field on Inventory records. The current System Design Specification (SDS) references AWS S3 for image and object storage.

During Sprint 2 planning, the development team identified that requiring authorized staff to manually enter an external image URL is error-prone, insecure, and inconsistent with a professional inventory-management workflow.

The team has therefore decided that authorized staff should **upload** an Inventory image through a structured workflow rather than entering a raw URL. The approved hosted image provider for this workflow is:

**Cloudinary**

This decision introduces a controlled architecture change from the AWS S3 reference currently documented in the SDS. Adopting Cloudinary for Inventory images is a deliberate, bounded decision. It does not automatically migrate all other OSMS object-storage use cases from AWS S3 to Cloudinary, and its impact on existing SDS storage references must be explicitly documented.

### Relevant SRS References

- FR-003 – Digital Product Catalogs and Filters
- FR-005 – Real-Time Stock Synchronization
- Relevant Inventory-management requirements
- Relevant authentication and authorization requirements

### Relevant SDS References

- Inventory entity
- React Presentation Layer
- Backend Application Layer
- Cloud / object-storage architecture
- AWS S3 image/object-storage references
- Authorization Levels / Access Control Matrix
- Input Validation and Sanitization Strategy
- Data Protection / Security Design

### Relevant ADR References

- ADR-002 – Authoritative RBAC Role Model
- ADR-006 – Inventory Reorder-Alert Rules and Persistence Model

This ADR must not redefine stock, threshold, or reorder-alert business rules established by ADR-006.

---

## 4. Existing Design Gap

The current Inventory entity supports an `imageUrl` field but provides no approved:

- upload architecture or backend upload endpoint,
- image provider configuration,
- file-type or file-size validation rules,
- image metadata persistence contract (`imagePublicId`),
- image replacement or deletion strategy,
- orphan-upload compensation strategy,
- frontend upload boundary definition.

The SDS currently references AWS S3 for object storage. Without one authoritative ADR, separate developers could independently introduce conflicting image-upload approaches, expose provider credentials in frontend bundles, or skip orphan-upload handling entirely.

---

## 5. Problem

OSMS must allow authorized staff to upload an Inventory image through a safe, backend-mediated workflow. The architecture must define:

1. the selected storage provider,
2. the upload architecture pattern (client-direct vs. backend-mediated),
3. role-based authorization for image upload, replacement, and deletion,
4. approved image formats (allow-list),
5. maximum file size and where that limit is enforced,
6. controlled Cloudinary folder / provider namespace,
7. persisted image metadata fields,
8. secure URL requirements,
9. Inventory creation flow integration,
10. image replacement sequencing and safety,
11. image removal sequencing and authorization,
12. orphan-upload handling strategy,
13. backend response contract,
14. error-handling and response safety rules,
15. provider credential management,
16. frontend boundary definition,
17. backend boundary definition,
18. SDS and AWS S3 alignment impact.

---

## 6. Storage Provider Options

### Option A — Cloudinary

Cloudinary is a hosted media-management platform offering authenticated upload, automatic CDN delivery, transformation capabilities, and provider-assigned asset identifiers (`public_id`).

#### Advantages

- Managed CDN delivery without additional infrastructure.
- Built-in asset management with stable `public_id` references for safe replacement and deletion.
- Supports backend-authenticated upload, keeping secrets off the frontend.
- Well-documented Node.js SDK compatible with the OSMS Express backend.
- Generous free-tier suitable for Sprint 2 scope.

#### Disadvantages

- Introduces a new third-party provider dependency.
- SDS must be updated to reflect that Cloudinary replaces AWS S3 for Inventory images.
- Cloudinary API secrets require careful backend environment management.

---

### Option B — AWS S3 (Current SDS Reference)

Use the existing AWS S3 reference documented in the SDS for Inventory image storage.

#### Advantages

- Consistent with the current SDS storage architecture.
- No SDS architecture change is required for storage layer.

#### Disadvantages

- Requires AWS infrastructure provisioning, IAM policies, and bucket configuration.
- Pre-signed URL workflows add backend complexity.
- No built-in image CDN without additional CloudFront configuration.
- Higher setup overhead for Sprint 2 timeline.

---

### Option C — Local Filesystem Storage

Store uploaded images directly on the OSMS backend server filesystem.

#### Advantages

- No external provider dependency.
- Simple initial implementation.

#### Disadvantages

- Not suitable for a multi-instance or cloud-deployed backend.
- No CDN delivery.
- File management, backup, and cleanup are the team's responsibility.
- Violates the architecture guidance that images should not be stored as local filesystem paths.

---

## 7. Storage Provider Decision

**Selected: Option A — Cloudinary.**

Cloudinary will be used as the hosted image provider for the Inventory-image workflow.

Reasons:

- Managed CDN without additional infrastructure.
- Stable `public_id` enables safe replacement and deletion.
- Backend-mediated SDK upload keeps all credentials server-side.
- Compatible with the OSMS Express / Node.js backend stack.
- Suitable for Sprint 2 scope and timeline.

This decision does not automatically migrate every other OSMS object-storage use case from AWS S3 to Cloudinary. Existing SDS AWS S3 references for unrelated storage responsibilities remain unaffected unless separately decided.

---

## 8. Upload Architecture Options

### Option A — Backend-Mediated Authenticated Upload

The OSMS backend receives the image from the frontend as a multipart form upload, validates it, authenticates to Cloudinary using server-side credentials, uploads the image to Cloudinary, and returns normalized image metadata to the frontend.

```text
Frontend
   ↓ multipart image
OSMS Backend
   ↓ authenticated + authorized + validated upload
Cloudinary
   ↓ secure_url + public_id
OSMS Backend
   ↓ normalized safe response
Frontend
```

#### Advantages

- Cloudinary API secrets remain entirely server-side.
- Backend can enforce authorization before upload occurs.
- Backend can enforce MIME type and file-size validation before forwarding to Cloudinary.
- Frontend receives only normalized, provider-neutral metadata.
- Consistent with OSMS backend-is-authoritative security model.

#### Disadvantages

- Each upload traverses the OSMS backend, which adds latency compared to direct upload.
- Backend must handle multipart form data processing.

---

### Option B — Frontend Direct Upload with Cloudinary Widget

The frontend uses a Cloudinary Upload Widget or unsigned preset to upload directly from the browser to Cloudinary.

#### Advantages

- Reduces backend involvement in upload traffic.
- Cloudinary widget handles preview and progress natively.

#### Disadvantages

- Requires an unsigned upload preset, which weakens access control.
- Cloudinary API key or upload preset must be bundled into the frontend.
- Backend authorization is bypassed during the upload phase.
- Any authenticated or unauthenticated user who obtains the widget configuration could upload arbitrary assets to OSMS's Cloudinary account.
- Incompatible with the OSMS security model requiring backend-authoritative authorization.

---

## 9. Upload Architecture Decision

**Selected: Option A — Backend-Mediated Authenticated Upload.**

The frontend must not contain or use Cloudinary API secrets.

Cloudinary provider integration belongs entirely to the backend.

The backend must:

- receive the image as a multipart upload,
- authenticate the requesting user before processing,
- authorize the requesting user's role before uploading,
- validate MIME type and file size,
- upload to Cloudinary using server-side credentials,
- return normalized image metadata to the frontend.

---

## 10. Authorization Model

Authorization must follow ADR-002 – Authoritative RBAC Role Model.

### Permitted Roles — Inventory Image Upload and Replacement

`BRANCH_MANAGER` is the primary role authorized for Inventory metadata management, including image upload and replacement.

Under the current backend Inventory contract:

- `BRANCH_MANAGER` may upload an Inventory image during Inventory creation.
- `BRANCH_MANAGER` may replace an existing Inventory image.

### Restricted Roles

- `SYSTEM_ADMIN` is limited to approved price overrides and system administration. `SYSTEM_ADMIN` must not automatically receive Inventory-image mutation permission unless explicitly authorized through the approved requirement and design process.
- `BRANCH_ASSISTANT` must not upload or replace Inventory images.
- `OPTOMETRIST` must not upload or replace Inventory images.
- `CUSTOMER` must not upload or replace Inventory images.

Backend authorization remains authoritative. Frontend hiding of upload controls is not a substitute for backend authorization checks.

### Authorization Summary

| Operation | `BRANCH_MANAGER` | `SYSTEM_ADMIN` | `BRANCH_ASSISTANT` | `OPTOMETRIST` | `CUSTOMER` |
|---|---|---|---|---|---|
| Upload Inventory image | Permitted | Not permitted | Not permitted | Not permitted | Not permitted |
| Replace Inventory image | Permitted | Not permitted | Not permitted | Not permitted | Not permitted |
| Remove Inventory image | Permitted | Not permitted | Not permitted | Not permitted | Not permitted |

---

## 11. Supported Image Type Options

### Option A — Explicit Allow-List

Define an explicit list of approved MIME types. Reject everything else.

Approved formats:

- `image/jpeg` (JPEG / JPG)
- `image/png` (PNG)
- `image/webp` (WebP)

#### Advantages

- Predictable, auditable format set.
- Prevents upload of SVG (which may embed active scripts), PDF, executable, or arbitrary binary files.
- Backend validates actual MIME content, not only filename extension.

#### Disadvantages

- New formats require explicit ADR or implementation approval.

---

### Option B — Accept All Common Web Image Formats

Accept any browser-reported `Content-Type` that starts with `image/`.

#### Advantages

- More permissive without explicit configuration.

#### Disadvantages

- SVG files embed executable XML and script content. Accepting SVG creates an XSS vector.
- Browser-reported content type can be spoofed at the filename level.
- Does not meet the OSMS security requirement for server-side MIME validation.

---

## 12. Supported Image Type Decision

**Selected: Option A — Explicit Allow-List.**

The backend must validate the uploaded file's actual MIME type (not the filename extension alone).

Approved MIME types for Inventory images:

| Format | MIME Type |
|---|---|
| JPEG / JPG | `image/jpeg` |
| PNG | `image/png` |
| WebP | `image/webp` |

SVG, PDF, and other file formats must be rejected by the backend with an appropriate error response.

---

## 13. Maximum File Size Options

### Option A — 5 MB per Inventory Image

A 5 MB limit is appropriate for Sprint 2 Inventory product images:

- Suitable for high-quality product photography.
- Prevents accidental upload of excessively large files.
- Aligns with typical Cloudinary free-tier usage.

### Option B — No Explicit Limit

No size limit is enforced.

This is not selected because uncontrolled file sizes could exhaust Cloudinary storage, increase backend memory usage, and degrade upload performance.

---

## 14. Maximum File Size Decision

**Selected: Option A — 5 MB per Inventory image.**

- The frontend may validate file size for usability (to provide immediate user feedback).
- The backend must enforce the 5 MB limit authoritatively before forwarding the upload to Cloudinary.
- The size limit must be configured in one backend-owned location (for example, an environment variable or a shared backend configuration constant), not duplicated across controllers and middleware.

---

## 15. Cloudinary Folder / Provider Namespace

### Decision

Inventory images must be uploaded to the controlled Cloudinary folder:

`osms/inventory/`

This ensures:

- Inventory images are organized under a predictable, auditable provider namespace.
- The backend controls the destination folder; the client must not be allowed to select arbitrary Cloudinary folders as trusted input.
- Provider-side asset organization is consistent across environments.

---

## 16. Persisted Image Metadata Fields

### Decision

The Inventory entity must support two image metadata fields:

| Field | Description |
|---|---|
| `imageUrl` | The Cloudinary HTTPS `secure_url`. Used by the frontend to render the Inventory image. |
| `imagePublicId` | The Cloudinary `public_id` for the uploaded asset. Required for safe replacement and deletion through the backend. |

`imageUrl` stores the hosted HTTPS URL used to display the image.

`imagePublicId` stores the provider asset identifier required to replace or delete the Cloudinary asset through a trusted server-side call.

The client must not be allowed to supply an arbitrary `imagePublicId` value to delete or replace unrelated Cloudinary assets.

`imagePublicId` must be treated as internal server-managed metadata. It must never be used as a client-controllable deletion key without backend re-validation.

---

## 17. Secure URL Requirement

### Decision

The backend must persist only the Cloudinary HTTPS `secure_url`.

Inventory images must not be stored as:

- local filesystem paths,
- temporary upload paths,
- base64-encoded blobs,
- browser-local object URLs.

Only HTTPS Cloudinary delivery URLs are acceptable for the `imageUrl` field.

---

## 18. Inventory Creation Flow

### Decision

The approved Inventory-image creation flow is:

```text
1. Branch Manager selects an image in the frontend.
2. Frontend performs basic UX validation (type check, size check).
3. Frontend sends the file to the protected backend upload endpoint via multipart form upload.
4. Backend authenticates the requesting user.
5. Backend authorizes the requesting user's role.
6. Backend validates MIME type.
7. Backend validates file size.
8. Backend uploads the image to Cloudinary (osms/inventory/ folder).
9. Cloudinary returns secure_url and public_id.
10. Backend returns normalized imageUrl and imagePublicId to the frontend.
11. Frontend uses imageUrl and imagePublicId in the Inventory create form submission.
12. Backend validates and persists the full Inventory record including image metadata.
```

Uploading an image alone must not be treated as a successful Inventory creation.

The image upload endpoint is a preparation step. The Inventory record is not created until the full Inventory creation request is submitted and persisted successfully.

---

## 19. Image Replacement Strategy

### Decision

When a `BRANCH_MANAGER` replaces an existing Inventory image, the backend must follow failure-safe sequencing:

```text
1. Receive new image upload request.
2. Authenticate and authorize the requesting user.
3. Validate MIME type and file size of the new image.
4. Upload the new image to Cloudinary.
5. Receive new secure_url and public_id from Cloudinary.
6. Update the Inventory record with the new imageUrl and imagePublicId.
7. Only after the Inventory record is successfully persisted:
   → delete the previous Cloudinary asset using the previously persisted imagePublicId.
```

The old Cloudinary asset must not be deleted before the new image metadata is successfully persisted.

Rationale: If the Inventory record update fails after the old asset is deleted, the Inventory record would reference a Cloudinary URL that no longer exists, leaving the Inventory in a broken state.

Failure-safe sequencing means the previous asset remains valid until the new asset is confirmed and persisted.

---

## 20. Image Removal Strategy

### Decision

Where image removal is supported:

1. The requesting user must be authenticated and authorized (`BRANCH_MANAGER` only).
2. The backend must clear `imageUrl` and `imagePublicId` from the Inventory record.
3. The backend must delete the corresponding Cloudinary asset using the trusted persisted `imagePublicId` — not a client-supplied value.
4. Deletion of the provider asset must use server-side credentials.

If the backend does not implement image removal, the frontend must not expose a Remove image control or action.

The frontend must not independently call Cloudinary to delete assets.

---

## 21. Orphan Upload Handling

### Decision

An orphan upload occurs when:

- The backend successfully uploads an image to Cloudinary, and
- The subsequent Inventory creation or update operation fails.

In this case, a Cloudinary asset exists that is not referenced by any Inventory record.

The backend must address orphan uploads through the following strategies:

### Strategy A — Immediate Compensation

If Inventory persistence fails after a successful Cloudinary upload, the backend must immediately attempt to delete the orphaned Cloudinary asset using the `public_id` returned from the upload response.

This is the preferred first-line strategy.

### Strategy B — Periodic Cleanup Job

A scheduled backend job may periodically identify Cloudinary assets in `osms/inventory/` that are not referenced by any active Inventory record and delete them.

This provides a safety net for cases where immediate compensation fails (for example, due to a backend crash after the upload but before the compensation attempt).

At minimum, immediate compensation must be attempted on Inventory persistence failure. The backend implementation issue must define the final cleanup strategy.

---

## 22. Backend Response Contract

### Decision

The backend upload endpoint must return a provider-neutral normalized response.

The approved response shape for a successful upload is:

```json
{
  "imageUrl": "https://res.cloudinary.com/<cloud-name>/image/upload/<version>/<public-id>",
  "imagePublicId": "osms/inventory/<asset-identifier>"
}
```

The response must not include:

- Cloudinary API secrets or credentials,
- raw Cloudinary administration response payloads,
- internal server stack traces,
- filesystem paths,
- temporary upload paths,
- provider request details.

---

## 23. Error Handling Contract

### Decision

The backend must return safe, structured error responses for upload failures.

Permitted error information to return to the client:

| Condition | Permitted Response |
|---|---|
| Unsupported MIME type | Error message: unsupported image type |
| File exceeds size limit | Error message: image too large |
| Upload provider unavailable | Error message: upload unavailable |
| Unauthorized role | HTTP 401 / 403: unauthorized |

Error responses must not expose:

- Cloudinary API secrets,
- backend stack traces,
- server filesystem paths,
- temporary server upload paths,
- internal Cloudinary provider request details.

---

## 24. Provider Credential Management

### Decision

Cloudinary configuration belongs to backend environment configuration only.

Required Cloudinary configuration values:

- `CLOUDINARY_CLOUD_NAME`
- `CLOUDINARY_API_KEY`
- `CLOUDINARY_API_SECRET`

Credential security requirements:

- Credentials must not be committed to Git (must be listed in `.gitignore` via `.env`).
- Credentials must not be bundled into frontend build artifacts.
- Credentials must not appear in browser environment variables.
- Credentials must not be logged by the backend.
- Credentials must not be returned through any API response.
- The backend must load credentials from environment variables or a secure configuration mechanism.

---

## 25. Frontend Boundary

### Decision

The frontend image-upload implementation (DDP-053) must:

- Replace the manual raw `imageUrl` text input with an image upload experience.
- Provide file selection (file input or drag-and-drop).
- Provide image preview before submission.
- Perform basic client-side UX validation (file type check, file size check) to provide immediate user feedback.
- Show an upload loading state while the backend processes the upload.
- Display safe upload error messages (unsupported type, too large, upload unavailable, unauthorized).
- Show replacement and removal controls where supported by the backend.

The frontend must not:

- independently integrate Cloudinary administration credentials,
- call Cloudinary APIs directly,
- use Cloudinary Upload Widget unsigned presets,
- expose or log Cloudinary API keys or secrets,
- use client-supplied `imagePublicId` to directly trigger Cloudinary deletion operations.

---

## 26. Backend Boundary

### Decision

The backend upload implementation (DDP-078) must:

- Expose a protected HTTP upload endpoint accepting multipart form data.
- Authenticate the requesting user before processing.
- Authorize the requesting user using the `BRANCH_MANAGER` role check before upload.
- Validate MIME type against the approved allow-list (`image/jpeg`, `image/png`, `image/webp`).
- Validate file size against the configured maximum (5 MB).
- Upload validated images to Cloudinary under `osms/inventory/` using server-side credentials.
- Return normalized `imageUrl` and `imagePublicId` in the response.
- Implement failure-safe replacement sequencing (persist first, delete old asset second).
- Implement orphan-upload immediate compensation on Inventory persistence failure.
- Manage Cloudinary credentials entirely through backend environment configuration.
- Never return Cloudinary secrets, stack traces, or internal provider details in any response.

---

## 27. SDS and AWS S3 Alignment Impact

### Decision

The SDS must be updated through the normal change-management process to reflect:

1. Cloudinary is the approved hosted provider for Inventory images in OSMS.
2. The Inventory image architecture follows this ADR (ADR-008), not the current AWS S3 object-storage reference for this specific use case.
3. AWS S3 references in the SDS for unrelated OSMS storage responsibilities are not automatically replaced by this ADR. Only Inventory image storage is affected by this decision.
4. The `imagePublicId` field is added to the Inventory entity as an approved metadata field alongside the existing `imageUrl`.

This impact must be noted in the SDS update work item and resolved through the approved documentation change-management process before final SDS release.

---

## 28. Scope Boundaries

This ADR does not introduce:

- a general media-management platform,
- multiple-image galleries per Inventory item,
- image editing or cropping,
- AI image processing,
- Customer image uploads,
- Prescription-document storage changes,
- a complete S3-to-Cloudinary migration,
- Cloudinary administration UI,
- automated image transformations,
- supplier or purchase-order image attachments.

---

## 29. Consequences

### Positive Consequences

- One authoritative upload architecture ensures consistent, secure implementation across frontend, backend, and Cloudinary integration.
- Backend-mediated upload keeps all provider credentials server-side.
- Explicit MIME allow-list and backend size enforcement prevent malicious or oversized uploads.
- `imagePublicId` enables safe, reliable asset replacement and deletion.
- Failure-safe replacement sequencing prevents broken Inventory image references.
- Orphan-upload compensation prevents indefinite accumulation of unreferenced Cloudinary assets.
- Provider-neutral response contract decouples frontend from Cloudinary internals.
- SDS impact is explicitly documented rather than introduced as an undocumented implementation detail.

### Negative Consequences

- Each Inventory image upload traverses the OSMS backend (higher latency than direct client-to-Cloudinary upload).
- The backend must handle multipart form data processing and Cloudinary SDK integration.
- Cloudinary environment configuration must be managed carefully to prevent credential exposure.
- An orphan-cleanup strategy must be implemented and maintained.
- The SDS must be updated to reflect the new provider and `imagePublicId` field.
- One new external provider dependency (`cloudinary` Node.js SDK) is introduced to the backend.

---

## 30. SRS / SDS / ADR References

### SRS

- FR-003 – Digital Product Catalogs and Filters
- FR-005 – Real-Time Stock Synchronization
- Relevant Inventory-management requirements
- Relevant authentication and authorization requirements

### SDS

- Inventory entity (extended with `imagePublicId`)
- React Presentation Layer – Inventory management UI
- Backend Application Layer – upload endpoint and Cloudinary SDK integration
- Cloud / object-storage architecture – Cloudinary replaces AWS S3 for Inventory images
- Authorization Levels / Access Control Matrix – `BRANCH_MANAGER` upload authorization
- Input Validation and Sanitization Strategy – MIME type and file-size validation
- Data Protection / Security Design – credential management, no secrets in responses

### ADR

- ADR-002 – Authoritative RBAC Role Model (role authorization model followed)
- ADR-006 – Inventory Reorder-Alert Rules and Persistence Model (stock business rules not redefined)

---

## 31. Implementation Boundary

This ADR defines architecture and decision rules only.

This ADR does not implement:

- Inventory model changes,
- backend upload endpoint code,
- Cloudinary SDK integration code,
- MIME-type or file-size middleware,
- Cloudinary orphan-cleanup job,
- frontend image upload component,
- frontend drag-and-drop or preview UI,
- updated SDS documentation.

Those changes belong to their dedicated implementation issues:

- DDP-078 – Secure backend Inventory image upload and Cloudinary integration
- DDP-053 – Implement Inventory Management and Low-Stock Alert Interfaces (frontend image upload)

---

## 32. Verification Checklist

- [x] Cloudinary provider selected
- [x] backend-mediated upload architecture selected
- [x] role authorization documented
- [x] no frontend secret exposure
- [x] supported formats defined (JPEG, PNG, WebP)
- [x] 5 MB size limit confirmed
- [x] controlled Cloudinary folder defined (`osms/inventory/`)
- [x] `imageUrl` field defined
- [x] `imagePublicId` field defined
- [x] replacement strategy defined (failure-safe sequencing)
- [x] removal strategy defined
- [x] orphan-upload handling defined
- [x] safe response contract defined
- [x] safe error contract defined
- [x] frontend boundary defined
- [x] backend boundary defined
- [x] SDS / AWS S3 impact documented

---

## 33. Review Record

| Date | Reviewer / Group | Result | Notes |
|---|---|---|---|
| 2026-10-13 | Development Team | Approved | All decisions reviewed and agreed: Cloudinary provider, backend-mediated upload, BRANCH_MANAGER authorization, JPEG/PNG/WebP allow-list, 5 MB limit, osms/inventory/ folder, imageUrl + imagePublicId fields, fail-safe replacement sequencing, and orphan-upload strategy. No blocking issues. |
| 2026-10-13 | Development Team | Approved | Full team discussion completed. All architecture decisions accepted unanimously. |
| 2026-10-13 | Supervisor | Approved | Supervisor review completed. No blocking issues remain. ADR approved for Accepted status. |
