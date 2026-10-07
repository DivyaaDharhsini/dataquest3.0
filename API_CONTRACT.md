# API Contract

## Common API Conventions

- **API Version Convention**: All endpoints are prefixed with `/api/v1/`.
- **Authentication Mechanism**: JWT (JSON Web Tokens) via `Authorization: Bearer <token>` header.
- **Authorization Approach**: Role-Based Access Control (RBAC) combined with Object/Site-level scoping (users can only access data for sites/customers they are authorized for).
- **JSON Format**: `application/json` for all request/response bodies.
- **ISO Timestamp Format**: ISO 8601 format (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- **Pagination Format**:
  - Request: `?page=1&limit=20`
  - Response: `{"data": [...], "meta": {"total": 100, "page": 1, "limit": 20}}`
- **Standard Success Response**:
  ```json
  {
    "success": true,
    "data": { ... },
    "meta": { ... }
  }
  ```
- **Standard Error Response**:
  ```json
  {
    "success": false,
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "Human readable message",
      "details": [...]
    }
  }
  ```
- **Error Codes**: `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION_ERROR`, `CONFLICT`, `RESOURCE_UNAVAILABLE`, `SLA_RISK`.
- **Naming Conventions**: `camelCase` for JSON properties, `kebab-case` for endpoint URLs.

---

## Service Request State Machine

| CURRENT STATE | NEXT STATE | WHO CAN TRIGGER IT | IMPORTANT PRECONDITION |
| :--- | :--- | :--- | :--- |
| **DRAFT** | **SUBMITTED** | Customer / System | Required basic payload (machine, problem) is provided. |
| **SUBMITTED** | **CLASSIFYING** | System | Request normalized and stored. |
| **CLASSIFYING** | **VALIDATING** | AI Classification Agent | AI structured extraction completes or fallback rules apply. |
| **VALIDATING** | **VALIDATED** | Deterministic Backend | Backend rules pass (machine active, SLA computed). |
| **VALIDATING** | **NEEDS_REVIEW** | Deterministic Backend | Anomaly detected (e.g. invalid contract). |
| **NEEDS_REVIEW** | **VALIDATED** | Manager | Manager manually resolves the block. |
| **VALIDATED** | **PLANNING** | System | Priority and SLA target locked. |
| **PLANNING** | **RESOURCE_CHECK** | System | Resource prediction and candidate matching generated. |
| **RESOURCE_CHECK** | **RESOURCES_READY** | Deterministic Backend | Parts and technician time successfully reserved. |
| **RESOURCE_CHECK** | **BLOCKED_BY_RESOURCE** | Deterministic Backend | Parts/tech unavailable (reservation transaction fails). |
| **BLOCKED_BY_RESOURCE** | **RESOLVING** | Exception Engine / Manager | Alternative warehouse/tech identified. |
| **RESOLVING** | **RESOURCE_CHECK** | Manager | Manager approves alternative resource plan. |
| **RESOURCES_READY** | **PENDING_APPROVAL** | System | Reservations held. |
| **PENDING_APPROVAL** | **APPROVED** | Manager / Auto-policy | Approved based on priority/cost rules. |
| **PENDING_APPROVAL** | **REJECTED** | Manager | Manager rejects plan (releases reservations). |
| **APPROVED** | **DISPATCHED** | System | Converts to executable job, SLA response timer starts. |
| **DISPATCHED** | **ACCEPTED_BY_TECH** | Technician | Tech acknowledges notification. |
| **ACCEPTED_BY_TECH** | **IN_PROGRESS** | Technician | Tech physically checks in at site. |
| **IN_PROGRESS** | **BLOCKED** | Technician / System | Job cannot proceed (e.g. missing part). |
| **BLOCKED** | **EXCEPTION_OPEN** | Exception Engine | Generates exception/intervention recommendation. |
| **EXCEPTION_OPEN** | **REASSIGNMENT** | Manager | Re-reserves resources and triggers workflow loop. |
| **IN_PROGRESS** | **COMPLETED_PENDING_VERIFICATION** | Technician | Work checklist submitted. |
| **COMPLETED_PENDING_VERIFICATION** | **VERIFICATION** | System / Manager | System OCR/Vision checks evidence and consumed parts. |
| **VERIFICATION** | **CUSTOMER_APPROVAL** | System | Verification passed. |
| **VERIFICATION** | **IN_PROGRESS** | Manager | Verification failed (rework required). |
| **CUSTOMER_APPROVAL** | **CLOSED** | Customer / Manager | Customer accepts service; updates machine passport. |

---

## AI Boundary

**AI RECOMMENDATION (Advisory / Generative)**
*Must be strictly validated; never directly mutate operational state.*
- **Issue Classification**: Extracting failure mode, severity, required skills from raw text/images.
- **Resource Prediction**: Estimating probable parts, labor duration, and cost.
- **Technician Matching**: Generating a transparent match score based on distance, skills, and SLA.
- **SLA Risk**: Calculating breach probability (Safe, At Risk).
- **Exceptions**: Suggesting root cause and recommended intervention.
- **Copilot**: Natural language summaries, Q&A, and troubleshooting guidance.
- **Predictive Maintenance**: Calculating health score from IoT telemetry.

**DETERMINISTIC BACKEND ACTION (Authoritative / Transactional)**
*Must enforce business rules and database constraints.*
- **Permissions**: RBAC and site scoping checks.
- **Validation**: Verifying machine exists, active contracts, mandatory fields.
- **Inventory/Reservations**: Database transactions to prevent negative stock and double-booking.
- **Approvals**: Enforcing approval tier policies and auditing decisions.
- **State Transitions**: Enforcing the state machine paths.
- **Closure**: Finalizing parts consumption, updating machine passport, and generating immutable audit logs.

---

## API Surface

### 1. Authentication & Users
- **Method**: POST
- **Endpoint**: `/api/v1/auth/login`
- **Purpose**: Authenticate user and issue JWT.
- **Auth**: None
- **Allowed Roles**: All
- **Request Body**: `{"email": "...", "password": "..."}`
- **Response**: `{"token": "...", "user": {"id": "...", "role": "..."}}`
- **Rules**: Validates credentials.

### 2. Equipment & Customers
- **Method**: GET
- **Endpoint**: `/api/v1/machines/{id}`
- **Purpose**: Retrieve machine profile and digital service passport.
- **Auth**: Required
- **Allowed Roles**: All (Scoped to accessible sites)
- **Rules**: Verifies user has access to the machine's site.

### 3. Service Requests & Workflow
- **Method**: POST
- **Endpoint**: `/api/v1/service-requests`
- **Purpose**: Create a reactive service request.
- **Auth**: Required
- **Allowed Roles**: Customer, Manager, Admin
- **Request Body**:
  ```json
  {
    "machineId": "m-104",
    "description": "Abnormal vibration",
    "priority": "HIGH",
    "attachments": ["url1"]
  }
  ```
- **Response**: `{"id": "sr-999", "status": "SUBMITTED"}`
- **Rules**: Machine must exist. Checks for duplicate open requests.
- **Side effects**: Creates request, writes audit log.

- **Method**: POST
- **Endpoint**: `/api/v1/service-requests/{id}/classify`
- **Purpose**: Run AI extraction on the request.
- **Auth**: Required
- **Allowed Roles**: Manager, System
- **Response**: `{"issueType": "MECHANICAL", "severity": "HIGH", "requiredSkills": ["vibration_analysis"], "confidence": 0.92}`
- **Rules**: Schema validation of AI output.

- **Method**: GET
- **Endpoint**: `/api/v1/service-requests/{id}/candidates`
- **Purpose**: Rank technician candidates.
- **Auth**: Required
- **Allowed Roles**: Manager, System
- **Response**: `{"candidates": [{"technicianId": "tech-1", "score": 85, "scoreJson": {...}}]}`

- **Method**: POST
- **Endpoint**: `/api/v1/service-requests/{id}/reserve`
- **Purpose**: Deterministically reserve tech and parts.
- **Auth**: Required
- **Allowed Roles**: Manager, System
- **Request Body**: `{"technicianId": "tech-1", "parts": [{"partId": "p-2", "warehouseId": "w-1", "qty": 1}]}`
- **Rules**: Database transaction. Fails if overlapping tech reservation or insufficient part stock.
- **Side effects**: Creates `resource_reservations`.

- **Method**: POST
- **Endpoint**: `/api/v1/service-requests/{id}/approve`
- **Purpose**: Manager approves the job plan.
- **Auth**: Required
- **Allowed Roles**: Manager
- **Request Body**: `{"decision": "APPROVED"}`
- **Side effects**: State changes to APPROVED.

- **Method**: POST
- **Endpoint**: `/api/v1/service-requests/{id}/dispatch`
- **Purpose**: Dispatch job to technician.
- **Auth**: Required
- **Allowed Roles**: Manager, System
- **Side effects**: State changes to DISPATCHED. SLA response timer starts. Push notification to technician.

- **Method**: POST
- **Endpoint**: `/api/v1/service-requests/{id}/check-in`
- **Purpose**: Technician checks in at site.
- **Auth**: Required
- **Allowed Roles**: Technician
- **Rules**: Tech must be assigned to this exact request.
- **Side effects**: State changes to IN_PROGRESS.

- **Method**: POST
- **Endpoint**: `/api/v1/service-requests/{id}/complete`
- **Purpose**: Submit job completion evidence.
- **Auth**: Required
- **Allowed Roles**: Technician
- **Request Body**: `{"checklistItems": [...], "consumedParts": [...]}`
- **Rules**: Validates mandatory checklist items are present.
- **Side effects**: State changes to COMPLETED_PENDING_VERIFICATION.

### 4. Inventory
- **Method**: GET
- **Endpoint**: `/api/v1/inventory/availability`
- **Purpose**: Check parts across warehouses.
- **Auth**: Required
- **Allowed Roles**: Manager, Inventory Manager
- **Query Params**: `?partId=p-2&lat=...&lng=...`

### 5. Exceptions
- **Method**: GET
- **Endpoint**: `/api/v1/exceptions`
- **Purpose**: List active exceptions (e.g. tech dropout, SLA at risk).
- **Auth**: Required
- **Allowed Roles**: Manager

- **Method**: POST
- **Endpoint**: `/api/v1/exceptions/{id}/resolve`
- **Purpose**: Manager resolves exception (e.g. triggers reassignment).
- **Auth**: Required
- **Allowed Roles**: Manager

### 6. AI & Copilot
- **Method**: POST
- **Endpoint**: `/api/v1/ai/copilot`
- **Purpose**: Authorized natural-language query.
- **Auth**: Required
- **Allowed Roles**: Manager, Technician
- **Request Body**: `{"query": "Why is M-104 delayed?", "contextId": "sr-999"}`
- **Response**: `{"answer": "Delay due to missing bearing X. Nearby warehouse W-2 has 4 units.", "recommendedAction": "TRANSFER_PART"}`
- **Rules**: AI must only use explicitly provided tool data; no raw SQL execution.

---

## Team Implementation Dependencies

| Module | Backend Dependency | Database Dependency | Frontend Dependency | AI/ML Dependency |
| :--- | :--- | :--- | :--- | :--- |
| **Auth & Core CRUD** | Auth, Customers, Sites, Machines | `users`, `roles`, `customers`, `sites`, `machines` | Login, Dashboards | None |
| **Service Request Core** | Request CRUD, State machine transitions | `service_requests`, `audit_logs` | Request form, Job view | Classification Engine |
| **Resource Engine** | Availability checks, transactional reservations | `technician_availability`, `resource_reservations` | Candidate UI | Parts Prediction |
| **Execution** | Checklist APIs, Evidence upload | `work_checklist`, `attachments`, `service_reports` | Technician Mobile UI | OCR (Optional) |
| **Exceptions** | Event stream, Rules engine | `exception_events` | Command Center UI | Matching + Workflow |

**Independent APIs for Parallel Implementation:**
1. Authentication & RBAC routes
2. Equipment & Customer CRUD operations
3. Inventory (Stock checking & Transfers)
4. Static Dashboards (Mocked data layer)
