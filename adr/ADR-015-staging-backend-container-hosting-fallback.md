# ADR-015: Staging Backend Container Hosting Operational Fallback (Render.com)

## 1. Title

Architectural Decision Record (ADR-015): Authoritative Operational Fallback for Staging Backend Container Hosting on Render.com Preserving Docker Containerization Parity and SDS v1.0 Alignment

## 2. Status

**Accepted** — Formally approved through architectural review and team consensus on 2026-10-22 to unblock Sprint 3 End-to-End System Integration Testing, UAT, and Academic Evaluation.

---

## 3. Context & Architectural Baseline

The Optical Shop Management System (OSMS) approved Software Design Specification (SDS v1.0 Section 7: *Deployment Design*) establishes the deployment architecture across separated environments:
- **Frontend Staging Target:** Dedicated preview/staging deployment on **Vercel** (osms-frontend-staging.vercel.app).
- **Backend Staging Target:** Dedicated containerized Node.js/Express service on **AWS App Runner** (osms-backend-staging), backed by **Amazon ECR** private container registry (osms-backend:staging-<commit-sha>).
- **Staging Database:** Dedicated **MongoDB Atlas** replica set cluster (osms-staging), strictly isolated from local development (osms-dev) and production (osms-prod).
- **Automation:** GitHub Actions continuous integration and continuous deployment pipelines (infra and ackend repositories).

Under **DDP-018** (Docker Local Development Environment), the backend container image is defined by infra/docker/backend/Dockerfile. Under **DDP-082** (Infra Issue #9), the production-grade entrypoint was aligned to CMD [npm, start] (
ode src/server.js) while preserving local hot-reloading via Docker Compose (command: [npm, run, dev]).

### The Cloud Quota & Provider Restriction Constraint

During the physical cloud provisioning phase of Infra Issue #9 (DDP-082), newly registered and student-tier AWS accounts encountered cloud provider service restrictions:
1. **AWS App Runner Regional Availability & Onboarding Restrictions:** AWS temporarily restricted the automated creation of new AWS App Runner services in target Asia Pacific regions (p-southeast-1 and adjacent regions) for unverified new accounts, requiring extended manual enterprise account verification and support ticket escalation.
2. **Delivery Milestone Dependency (Sprint 3 Blockers):** Multiple active Sprint 3 downstream deliverables strictly depend on an active, publicly accessible, HTTPS-secured staging backend API target:
   - **Sprint 3 Frontend Staging Deployment (Vercel #10):** Requires an active, live HTTPS API base URL for VITE_API_BASE_URL.
   - **Sprint 3 End-to-End & Integration Testing:** Full checkout, PayHere sandbox payments, appointment scheduling, and customer authentication validation require a stable, cloud-hosted API target independent of developer localhost.
   - **Customer & Staff Demonstration / Academic Evaluation:** Requires verifiable cloud availability over public HTTPS.

Waiting for indefinite cloud provider quota resolution risked missing critical university project deadlines.

---

## 4. Evaluated Options

The architecture team evaluated three potential deployment strategies:

| Evaluation Criteria | Option 1: AWS Serverless Lambda (Vercel Backend) | Option 2: Render.com Container Hosting (Selected Fallback) | Option 3: Wait for AWS App Runner Quota Resolution |
| :--- | :--- | :--- | :--- |
| **Docker Parity** | ❌ **Violated.** Bypasses Docker entirely; wraps Express as serverless functions. | ✅ **100% Preserved.** Executes the identical infra/docker/backend/Dockerfile container image. | ✅ Preserved. |
| **SDS Architectural Alignment** | ❌ **High Departure.** Converts monolithic Express to ephemeral serverless Lambda. | ✅ **High Alignment.** Monolithic, stateful Express container behind HTTPS reverse proxy. | ✅ Native SDS target. |
| **Timeline & Agility** | Fast setup, but requires Express refactoring for serverless cold-starts. | ✅ **Immediate.** Instant live container deployment with automated GitHub CI/CD. | ❌ High risk of milestone failure. |
| **Environment & Secrets** | Vercel dashboard environment variables. | ✅ **Secure.** Web-service runtime environment variable encryption matching staging.env.example. | App Runner / Secrets Manager. |
| **Cost & Quota** | Free tier available. | ✅ **Free tier available** (Zero infrastructure overhead). | Requires AWS billing activation. |
| **Reversibility** | Hard to reverse; codebase requires serverless wrappers. | ✅ **Seamlessly Reversible.** Same Docker image and env contract; zero code changes needed. | N/A |

---

## 5. Decision

The team decided to adopt **Option 2 (Render.com Container Hosting)** as an **authoritative, non-disruptive operational fallback for the OSMS Staging Backend** while strictly preserving the existing codebase, containerization specifications, and AWS CI/CD pipelines.

### Key Decision Rules:

1. **Preserve Containerization Parity (Zero Docker Changes):**
   Render builds and executes the backend using the exact approved Dockerfile:
   infra/docker/backend/Dockerfile with production entrypoint 
pm start (
ode src/server.js).
   No changes to application code, Docker configuration, or business logic are permitted for deployment convenience.

2. **Preserve Configuration Contract (staging.env.example Authority):**
   Render Web Service runtime configuration strictly conforms to the verified 42-variable contract established in infra/environments/staging.env.example:
   - NODE_ENV=production
   - PORT=5000
   - MONGODB_URI pointing to dedicated MongoDB Atlas osms-staging replica set.
   - FRONTEND_ORIGIN=https://osms-frontend-staging.vercel.app (enforcing strict CORS preflight).
   - PayHere sandbox merchant credentials, Cloudinary, and Gemini AI credentials injected securely via platform secret stores.
   - SMTP credentials kept disabled by default or pointed to controlled test recipients.

3. **Preserve AWS CI/CD Artifacts in infra:**
   The AWS App Runner deployment pipeline (.github/workflows/backend-staging-deploy.yml), AWS CLI automation scripts (scripts/deploy-apprunner.cjs), and health verification gates (scripts/verify-staging-health.cjs) remain fully intact in the infra repository. The automated test suite (33/33 tests passing) remains authoritative proof of AWS readiness once account restrictions are lifted.

4. **Staging Operational Target URL:**
   The live staging backend URL is provisioned as:
   https://osms-backend-staging.onrender.com
   with public HTTPS health-check endpoint:
   https://osms-backend-staging.onrender.com/api/health

---

## 6. Consequences & System Impact

### Positive Consequences
- **Unblocks Sprint 3 Deliverables:** The frontend team can immediately configure VITE_API_BASE_URL=https://osms-backend-staging.onrender.com/api on Vercel and complete staging integration.
- **Strict Docker Container Compliance:** Evaluators and examiners verify real Docker container execution running Node 20 on Linux, satisfying NFR-006 (Maintainability) and AC1 (Container Reuse).
- **Zero Cost & Zero Maintenance:** Eliminates student AWS billing risks and regional quota limitations.
- **Continuous Deployment (Auto-Sync):** Render automatically rebuilds the Docker container upon validated commits to develop, ensuring automated continuous delivery.
- **100% Reversible:** Because the application remains a standard Docker container conforming to the environment contract, switching back to AWS App Runner requires only pointing DNS or changing the frontend environment variable without any code rewrites.

### Negative & Mitigated Consequences
- **Free Tier Inactivity Sleep (Cold Start):**
  - *Risk:* Render free tier instances spin down after 15 minutes of inactivity, resulting in a ~30-50 second delay on the first incoming HTTP request.
  - *Mitigation:* Documented in operational runbooks; for university demonstration and scheduled evaluation sessions, an automated health ping or pre-demo wake-up probe ensures the container is warm and fully responsive.

---

## 7. Compliance and Verification Matrix

| Acceptance Criteria / NFR | Enforcement Mechanism | Status |
| :--- | :--- | :---: |
| **AC1: Docker Container Reuse** | Built directly from infra/docker/backend/Dockerfile | ✅ Compliant |
| **AC3: Production Node.js App** | Executes 
pm start (
ode src/server.js) on port 5000 | ✅ Compliant |
| **AC4: MongoDB Atlas Staging** | Injected SRV connection to dedicated osms-staging cluster | ✅ Compliant |
| **AC5: Zero Committed Secrets** | All keys injected via Render Encrypted Environment Variables | ✅ Compliant |
| **AC6: Strict Staging CORS** | Bound to https://osms-frontend-staging.vercel.app | ✅ Compliant |
| **AC9: HTTPS Health Verification**| GET /api/health returns 200 OK and { status: ok } | ✅ Compliant |
| **AC10: RBAC & JWT Security** | Full production JWT middleware and 5-role RBAC active | ✅ Compliant |
| **AC11: Sandbox Integrations** | PayHere sandbox, Gemini staging key active | ✅ Compliant |
| **AC12: Chatbot TTL Index Gate** | MongoDB Atlas staging database verified via scripts/chatbot-usage-index.js | ✅ Compliant |

---

## 8. References

- **SDS v1.0 Section 7:** Deployment Design and Environment Architecture
- **ADR-002:** Authoritative RBAC Role Model
- **ADR-011:** Secure Payment Gateway Integration Architecture
- **DDP-018:** Local Docker Development Environment
- **DDP-076 (Infra #4):** Secure Staging Environment Baseline Specification
- **DDP-082 (Infra #9):** Backend Staging Deployment & Automation
