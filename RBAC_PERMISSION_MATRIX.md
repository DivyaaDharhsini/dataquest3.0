# RBAC & Permission Matrix

## Roles Defined

Based on the problem statement, the system supports the following distinct roles:

1. **Customer / Client**: Creates reactive requests, views status of own machines, approves completed jobs.
2. **Technician**: Receives dispatched jobs, executes field work, consumes parts, uploads evidence.
3. **Operations / Service Manager**: Approves job plans, monitors command center, resolves exceptions, reassigns technicians.
4. **Inventory / Warehouse Manager**: Manages parts, tool inventory, reservations, shortages, and stock transfers.
5. **Administrator**: Manages users, roles, master data (skills, models), system configuration, and audits.

---

## Site Scope Definitions

Permissions are not just boolean; they are scoped territorially:

- **Own Records**: Data directly created by or explicitly assigned to the authenticated user.
- **Site-level**: Any record associated with a specific Site (e.g., Site A, Site B) the user is authorized for.
- **Customer-level**: Any record associated with a specific Customer organization (covering multiple sites).
- **Global**: Full system-wide access across all customers and sites.

---

## Permission Matrix

| Action | Customer / Client | Technician | Operations Manager | Inventory Manager | Administrator |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **View Requests** | Yes (Own Site/Customer) | Yes (Assigned only) | Yes (Site-level) | Yes (Site-level) | Yes (Global) |
| **Create Requests** | Yes (Own Site) | No | Yes (Site-level) | No | Yes (Global) |
| **Update Requests** | Yes (Add notes/attachments) | Yes (Assigned only) | Yes (Site-level) | No | Yes (Global) |
| **Delete Requests** | No | No | No | No | Yes (Soft delete only) |
| **Approve** | Yes (Customer Verification) | No | Yes (Plans, Costs, Reassignments) | No | Yes (Global) |
| **Assign** | No | No | Yes (Site-level) | No | Yes (Global) |
| **Reassign** | No | No | Yes (Site-level) | No | Yes (Global) |
| **Reserve Resources** | No | No | Yes (Automated via planning or Manual) | Yes (Inventory only) | Yes (Global) |
| **Execute Work** | No | Yes (Assigned only) | No | No | No |
| **Verify** | No | No | Yes (Site-level) | No | Yes (Global) |
| **Close** | Yes (Sign-off) | No | Yes (Override) | No | Yes (Global) |
| **Manage Inventory** | No | No | Read-only | Yes (Site-level) | Yes (Global) |
| **View Analytics** | Yes (Customer-level) | Yes (Own performance) | Yes (Site-level) | Yes (Inventory KPIs) | Yes (Global) |
| **View Audit History** | Yes (Own Site) | Yes (Own Jobs) | Yes (Site-level) | Yes (Site-level) | Yes (Global) |

---

## Implementation Freeze Summary

- **API Version**: v1 (JSON REST)
- **Roles**: Customer, Technician, Operations Manager, Inventory Manager, Administrator.
- **Main Request States**: DRAFT, SUBMITTED, CLASSIFYING, VALIDATING, NEEDS_REVIEW, PLANNING, RESOURCE_CHECK, BLOCKED_BY_RESOURCE, RESOLVING, RESOURCES_READY, PENDING_APPROVAL, APPROVED, REJECTED, DISPATCHED, ACCEPTED_BY_TECH, IN_PROGRESS, BLOCKED, EXCEPTION_OPEN, REASSIGNMENT, COMPLETED_PENDING_VERIFICATION, VERIFICATION, CUSTOMER_APPROVAL, CLOSED.
- **Authentication Approach**: JWT Session-based Auth with Object-level Site/Customer scoping enforced on every request.
- **Standard Response/Error Format**: `{ "success": boolean, "data": {...}, "error": {...} }`

### Decisions Requiring Team Confirmation
- **[DECISION REQUIRED]**: Confirm the exact SLA penalty/risk score formula and time thresholds before starting the Risk Engine backend.
- **[DECISION REQUIRED]**: Confirm whether Technicians are allowed to auto-reassign themselves in the field via the Copilot, or if Operations Manager approval is strictly required for all reassignments.
- **[TECHNICAL ASSUMPTION]**: We assume WebSockets will be used for Real-time Command Center updates rather than HTTP polling.
