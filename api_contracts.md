# DQBH (DataQuest 3.0) — Master API & System Contracts Specification

> **Purpose**: This document provides strict, binding API contracts, JSON schemas, WebSocket event formats, database entity alignments, and AI agent execution boundaries for the **DQBH Industrial Equipment Service Orchestration Platform**. It allows **Frontend**, **Backend**, **AI/Agent**, and **Database** teams to build and test components concurrently without blocking dependencies.

---

## 1. Global Specifications & Standards

### 1.1 Base URLs & Protocol
- **REST API Base URL**: `https://api.dqbh.local/api/v1` (or `http://localhost:8000/api/v1`)
- **WebSocket Gateway**: `wss://api.dqbh.local/ws/v1/events` (or `ws://localhost:8000/ws/v1/events`)
- **Content Type**: `application/json` (except multi-part file uploads)

### 1.2 Common HTTP Headers
```http
Authorization: Bearer <JWT_TOKEN>
X-Correlation-ID: <UUIDv4>
X-User-Role: MANAGER | TECHNICIAN | CUSTOMER | WAREHOUSE_MANAGER | ADMIN
Content-Type: application/json
```

### 1.3 Standard Response Wrapper
All REST responses MUST wrap payload responses in the standard wrapper:

#### Success Response Envelope (`200 OK`, `201 Created`)
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2026-10-07T13:22:00Z",
    "correlation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

#### Error Response Envelope (`400`, `401`, `403`, `404`, `409`, `422`, `500`)
```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_BLOCKED",
    "message": "Required spare part SKU-8823 is out of stock across nearby warehouses.",
    "details": [
      {
        "field": "part_id",
        "issue": "Insufficient inventory balance"
      }
    ]
  },
  "meta": {
    "timestamp": "2026-10-07T13:22:00Z",
    "correlation_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d"
  }
}
```

---

## 2. Common Data Types & Enums

### 2.1 Enums
- **`SourceType`**: `"PERIODIC" | "REACTIVE" | "PREDICTIVE"`
- **`Priority`**: `"LOW" | "MEDIUM" | "HIGH" | "CRITICAL"`
- **`Severity`**: `"MINOR" | "MODERATE" | "SEVERE" | "CRITICAL"`
- **`SLARiskCategory`**: `"GREEN" | "AMBER" | "RED"`
- **`UserRole`**: `"CUSTOMER" | "TECHNICIAN" | "MANAGER" | "WAREHOUSE_MANAGER" | "ADMIN"`
- **`WorkOrderStatus`**:
  - `DRAFT`: Initial draft saved by client/operator
  - `SUBMITTED`: Formally submitted for processing
  - `CLASSIFYING`: AI extraction in progress
  - `VALIDATING`: System checking business rules
  - `NEEDS_REVIEW`: Manual review required by manager
  - `VALIDATED`: Request validated against contract & rules
  - `PLANNING`: Resource planning engine estimating costs/parts/labor
  - `RESOURCE_CHECK`: Verification of inventory & technician slots
  - `BLOCKED_BY_RESOURCE`: Missing part or tool identified
  - `RESOURCES_READY`: Parts reserved, technician candidate set ready
  - `PENDING_APPROVAL`: Waiting manager sign-off
  - `REJECTED`: Rejected during approval phase
  - `APPROVED`: Manager approved execution plan
  - `DISPATCHED`: Job assigned and pushed to technician device
  - `ACCEPTED_BY_TECH`: Field technician accepted job
  - `IN_PROGRESS`: Technician checked in at site and executing
  - `BLOCKED`: Field work halted due to exception
  - `COMPLETED_PENDING_VERIFICATION`: Checklist & photos submitted by tech
  - `VERIFICATION_FAILED`: Completion audit rejected, rework required
  - `VERIFIED`: Completion evidence verified by manager/system
  - `CUSTOMER_APPROVAL`: Awaiting customer final sign-off
  - `CLOSED`: Job complete, service history digital passport updated
- **`ExceptionType`**: `"TECH_DROPOUT" | "PART_UNAVAILABLE" | "TECH_DELAYED" | "JOB_OVERRUN" | "SLA_BREACH_RISK" | "MACHINE_CRITICAL" | "CUSTOMER_ESCALATION"`
- **`ExceptionStatus`**: `"OPEN" | "IN_REVIEW" | "RESOLVED" | "AUTO_RESOLVED"`

---

## 3. Core Backend REST API Endpoints

### 3.1 Authentication & RBAC

#### `POST /auth/login`
- **Description**: Authenticate user and issue JWT token.
- **Request Body**:
```json
{
  "email": "manager@factory.com",
  "password": "SecurePassword123!"
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsIn...",
    "expires_at": "2026-10-08T13:22:00Z",
    "user": {
      "id": "usr_101",
      "name": "Alex Mercer",
      "email": "manager@factory.com",
      "role": "MANAGER",
      "home_site_id": "site_99"
    }
  }
}
```

---

### 3.2 Equipment & Site Management

#### `GET /machines`
- **Query Params**: `site_id` (string, optional), `status` (string, optional), `criticality` (string, optional)
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": [
    {
      "id": "mch_104",
      "asset_code": "M-104",
      "site_id": "site_01",
      "site_name": "Plant A - Chennai",
      "model_id": "mdl_vibe_9",
      "model_name": "Heavy Industrial Compressor 900",
      "serial_no": "SN-99482-B",
      "status": "OPERATIONAL",
      "criticality": "HIGH",
      "warranty_end": "2027-12-31"
    }
  ]
}
```

#### `GET /machines/{id}/passport`
- **Description**: Retrieve digital twin service passport and lifetime history of a machine.
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "machine": {
      "id": "mch_104",
      "asset_code": "M-104",
      "health_score": 78.5,
      "risk_status": "MODERATE_RISK",
      "total_service_count": 14,
      "last_serviced_at": "2026-08-15T09:00:00Z"
    },
    "recent_work_orders": [
      {
        "request_no": "REQ-2026-0891",
        "source_type": "REACTIVE",
        "description": "Bearing vibration anomaly",
        "completed_at": "2026-08-15T14:30:00Z",
        "technician_name": "David Miller",
        "parts_replaced": [
          { "part_sku": "BRG-6205-2RS", "qty": 1 }
        ]
      }
    ],
    "health_snapshots": [
      {
        "timestamp": "2026-10-07T08:00:00Z",
        "health_score": 78.5,
        "risk_score": 0.22,
        "explanation": "Vibration elevated by 12% over 48 hours."
      }
    ]
  }
}
```

---

### 3.3 Service Request Workflow Lifecycle

#### `POST /service-requests`
- **Description**: Create new service request intake (Level 1 Intake).
- **Request Body**:
```json
{
  "source_type": "REACTIVE",
  "machine_id": "mch_104",
  "description": "Machine M-104 has stopped due to abnormal high frequency vibration and loud noise.",
  "user_priority": "HIGH",
  "requested_window_start": "2026-10-07T14:00:00Z",
  "requested_window_end": "2026-10-07T18:00:00Z",
  "attachment_urls": [
    "https://storage.dqbh.com/evidence/m104_vibe_photo.jpg"
  ]
}
```
- **Response (`201 Created`)**:
```json
{
  "success": true,
  "data": {
    "id": "req_5501",
    "request_no": "REQ-2026-1042",
    "status": "SUBMITTED",
    "machine_id": "mch_104",
    "created_at": "2026-10-07T13:22:00Z"
  }
}
```

#### `POST /service-requests/{id}/classify`
- **Description**: Execute AI Classification Agent (Level 2) to convert raw text/media into structured diagnosis.
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "CLASSIFYING",
    "ai_analysis": {
      "issue_type": "MECHANICAL_BEARING_FAILURE",
      "failure_mode": "BEARING_WEAR_OVERHEATING",
      "calculated_severity": "HIGH",
      "required_skills": ["Mechanical Vibration", "Compressor Overhaul"],
      "probable_tools": ["Vibration Analyzer", "Hydraulic Puller"],
      "probable_parts": [
        { "sku": "BRG-6205-2RS", "name": "Heavy Duty Ball Bearing", "probability": 0.92, "qty": 1 },
        { "sku": "FUSE-20A-IND", "name": "20A Industrial Fuse", "probability": 0.65, "qty": 1 }
      ],
      "confidence_score": 0.89,
      "reasoning": "High-frequency vibration with noise strongly indicates inner race bearing damage."
    }
  }
}
```

#### `POST /service-requests/{id}/plan`
- **Description**: Compute SLA, Resource Prediction, and candidate search (Levels 4, 5, 6).
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "sla": {
      "priority": "HIGH",
      "response_deadline": "2026-10-07T15:22:00Z",
      "resolution_deadline": "2026-10-07T19:22:00Z",
      "risk_category": "AMBER",
      "risk_explanation": "Travel time from Warehouse B adds 45 min delay."
    },
    "resource_plan": {
      "estimated_labor_hours": 2.5,
      "estimated_travel_hours": 0.75,
      "estimated_cost": 340.00,
      "parts_status": "AVAILABLE_WITH_TRANSFER",
      "missing_parts": []
    }
  }
}
```

#### `GET /service-requests/{id}/candidates`
- **Description**: Get ranked list of feasible technicians for matching (Level 6).
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "candidates": [
      {
        "technician_id": "tech_88",
        "name": "Sarah Connor",
        "match_score": 92.5,
        "score_breakdown": {
          "skill_match": 100.0,
          "availability": 90.0,
          "sla_feasibility": 95.0,
          "distance_score": 85.0,
          "machine_familiarity": 90.0,
          "workload_score": 90.0,
          "performance_score": 95.0
        },
        "distance_km": 12.4,
        "eta_minutes": 25,
        "is_recommended": true
      }
    ]
  }
}
```

#### `POST /service-requests/{id}/reserve`
- **Description**: Execute atomic database transaction to lock technician time and inventory stock (Level 7).
- **Request Body**:
```json
{
  "technician_id": "tech_88",
  "reserved_parts": [
    { "part_sku": "BRG-6205-2RS", "warehouse_id": "wh_01", "qty": 1 }
  ]
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "RESOURCES_READY",
    "reservation_ids": ["res_9912", "res_9913"],
    "expires_at": "2026-10-07T14:22:00Z"
  }
}
```

#### `POST /service-requests/{id}/approve`
- **Description**: Manager sign-off on the service order plan (Level 8).
- **Request Body**:
```json
{
  "decision": "APPROVED",
  "reason": "Resource plan and SLA within bounds. Approved for immediate dispatch.",
  "override_technician_id": null
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "APPROVED",
    "approved_at": "2026-10-07T13:30:00Z",
    "approver_id": "usr_101"
  }
}
```

#### `POST /service-requests/{id}/dispatch`
- **Description**: Transition request to `DISPATCHED` and push notification to assigned technician (Level 9).
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "DISPATCHED",
    "assigned_technician_id": "tech_88",
    "dispatched_at": "2026-10-07T13:31:00Z"
  }
}
```

#### `POST /service-requests/{id}/check-in`
- **Description**: Field technician site check-in timestamp and geo-verify (Level 10).
- **Request Body**:
```json
{
  "technician_latitude": 13.0827,
  "technician_longitude": 80.2707
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "IN_PROGRESS",
    "checked_in_at": "2026-10-07T13:55:00Z"
  }
}
```

#### `PUT /service-requests/{id}/checklist`
- **Description**: Update checklist execution status by field technician.
- **Request Body**:
```json
{
  "checklist_items": [
    { "item_id": "chk_01", "completed": true, "note": "Power isolated and locked out." },
    { "item_id": "chk_02", "completed": true, "note": "Old bearing unmounted." }
  ]
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "completed_items_count": 2,
    "total_items_count": 4
  }
}
```

#### `POST /service-requests/{id}/complete`
- **Description**: Technician completes job, uploads completion photos and report (Level 10/13).
- **Request Body**:
```json
{
  "diagnosis_findings": "Inner race spalling on primary drive shaft bearing.",
  "work_performed": "Replaced ball bearing BRG-6205-2RS, re-aligned drive shaft, ran 15 min load test.",
  "consumed_parts": [
    { "part_sku": "BRG-6205-2RS", "qty": 1 }
  ],
  "completion_photo_urls": [
    "https://storage.dqbh.com/evidence/new_bearing_installed.jpg"
  ]
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "COMPLETED_PENDING_VERIFICATION",
    "submitted_at": "2026-10-07T15:45:00Z"
  }
}
```

#### `POST /service-requests/{id}/verify`
- **Description**: Perform automated/manager completion verification (Level 13).
- **Request Body**:
```json
{
  "verification_passed": true,
  "notes": "Work evidence verified, checklist 100% complete, parts reconciled."
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "CUSTOMER_APPROVAL",
    "verified_at": "2026-10-07T15:50:00Z"
  }
}
```

#### `POST /service-requests/{id}/customer-approval`
- **Description**: Final sign-off by client/customer to close service request (Level 14).
- **Request Body**:
```json
{
  "approved": true,
  "feedback_rating": 5,
  "comment": "Fast response, vibration fully resolved."
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "CLOSED",
    "closed_at": "2026-10-07T16:00:00Z"
  }
}
```

---

### 3.4 Exception Engine & Dynamic Reassignment

#### `GET /exceptions`
- **Query Params**: `status` (`OPEN` | `RESOLVED`), `severity` (`HIGH` | `CRITICAL`)
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": [
    {
      "id": "exp_7001",
      "request_id": "req_5501",
      "exception_type": "TECH_DROPOUT",
      "severity": "CRITICAL",
      "status": "OPEN",
      "payload": {
        "original_technician_id": "tech_88",
        "reason": "Technician reported vehicle breakdown en route."
      },
      "created_at": "2026-10-07T13:40:00Z"
    }
  ]
}
```

#### `POST /service-requests/{id}/reassign`
- **Description**: Trigger dynamic reassignment for work order with open exception (Level 12).
- **Request Body**:
```json
{
  "exception_id": "exp_7001",
  "new_technician_id": "tech_99",
  "reason": "Reassigning to nearest available backup technician."
}
```
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "request_id": "req_5501",
    "status": "DISPATCHED",
    "previous_technician_id": "tech_88",
    "new_technician_id": "tech_99",
    "reassigned_at": "2026-10-07T13:42:00Z"
  }
}
```

---

### 3.5 Inventory & Warehouse APIs

#### `GET /inventory/availability`
- **Query Params**: `part_sku` (required), `site_id` (optional)
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": [
    {
      "warehouse_id": "wh_01",
      "warehouse_name": "Chennai Main Warehouse",
      "part_sku": "BRG-6205-2RS",
      "available_qty": 15,
      "reserved_qty": 3,
      "latitude": 13.0827,
      "longitude": 80.2707
    },
    {
      "warehouse_id": "wh_02",
      "warehouse_name": "Sriperumbudur Depot",
      "part_sku": "BRG-6205-2RS",
      "available_qty": 4,
      "reserved_qty": 0,
      "latitude": 12.9699,
      "longitude": 79.9495
    }
  ]
}
```

#### `POST /inventory/transfers`
- **Description**: Request emergency part transfer between warehouses.
- **Request Body**:
```json
{
  "request_id": "req_5501",
  "part_sku": "FUSE-20A-IND",
  "from_warehouse_id": "wh_02",
  "to_warehouse_id": "wh_01",
  "qty": 1
}
```
- **Response (`201 Created`)**:
```json
{
  "success": true,
  "data": {
    "transfer_id": "trf_9901",
    "status": "IN_TRANSIT",
    "eta_minutes": 35
  }
}
```

---

### 3.6 Role-Based Dashboards

#### `GET /dashboard/manager`
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "kpis": {
      "open_requests": 12,
      "active_jobs": 5,
      "sla_at_risk": 2,
      "critical_exceptions": 1,
      "unassigned_requests": 3
    },
    "sla_watchlist": [
      {
        "request_no": "REQ-2026-1042",
        "machine_code": "M-104",
        "risk_category": "AMBER",
        "time_remaining_minutes": 45
      }
    ],
    "active_exceptions": [
      { "id": "exp_7001", "type": "TECH_DROPOUT", "request_no": "REQ-2026-1042" }
    ]
  }
}
```

#### `GET /dashboard/technician`
- **Response (`200 OK`)**:
```json
{
  "success": true,
  "data": {
    "assigned_jobs": [
      {
        "request_id": "req_5501",
        "request_no": "REQ-2026-1042",
        "machine_code": "M-104",
        "site_name": "Plant A - Chennai",
        "site_address": "12 Industrial Road, Sector 4",
        "priority": "HIGH",
        "status": "DISPATCHED",
        "response_deadline": "2026-10-07T15:22:00Z",
        "required_parts": [
          { "sku": "BRG-6205-2RS", "name": "Heavy Duty Ball Bearing", "reserved": true }
        ]
      }
    ]
  }
}
```

---

## 4. AI Agent Execution Interfaces & Boundaries

> **Architectural Rule**: AI agents return structured JSON proposals. Backend deterministic engines validate and execute actions. AI agents NEVER mutate database state directly.

```
┌─────────────────┐       JSON Proposal       ┌────────────────────────┐
│  AI Agent / ML  │ ────────────────────────> │ Deterministic Backend  │
│    (Service)    │                           │ (Authorization & DB)   │
└─────────────────┘                           └───────────┬────────────┘
                                                          │ Transact
                                                          ▼
                                              ┌────────────────────────┐
                                              │   PostgreSQL Database  │
                                              └────────────────────────┘
```

---

### 4.1 Request Classification Agent (Agent 1)
- **Endpoint**: `POST /ai/classify`
- **Input Payload**:
```json
{
  "request_id": "req_5501",
  "raw_description": "Machine M-104 vibration alert. Loud metallic squealing from upper housing.",
  "machine_context": {
    "model": "Heavy Industrial Compressor 900",
    "category": "COMPRESSOR",
    "historical_failure_modes": ["BEARING_WEAR", "VALVE_LEAK"]
  },
  "attachment_urls": ["https://storage.dqbh.com/evidence/m104_photo.jpg"]
}
```
- **Output Schema**:
```json
{
  "issue_type": "MECHANICAL_BEARING_FAILURE",
  "failure_mode": "BEARING_WEAR_OVERHEATING",
  "severity": "HIGH",
  "required_skills": ["Mechanical Vibration", "Compressor Overhaul"],
  "probable_tools": ["Vibration Analyzer", "Hydraulic Puller"],
  "probable_parts": [
    { "sku": "BRG-6205-2RS", "qty": 1, "confidence": 0.92 }
  ],
  "overall_confidence": 0.89,
  "explanation": "High frequency vibration combined with metallic noise matches inner race bearing degradation."
}
```

---

### 4.2 Resource Planning Agent (Agent 2)
- **Endpoint**: `POST /ai/plan-resources`
- **Input Payload**:
```json
{
  "request_id": "req_5501",
  "classification_result": { ... },
  "machine_id": "mch_104"
}
```
- **Output Schema**:
```json
{
  "estimated_labor_duration_hours": 2.5,
  "estimated_travel_duration_hours": 0.5,
  "estimated_cost_usd": 340.00,
  "recommended_parts": [
    { "sku": "BRG-6205-2RS", "qty": 1, "is_critical": true }
  ],
  "recommended_tools": ["Vibration Analyzer", "Puller Set"],
  "feasibility_snapshot": {
    "parts_available": true,
    "tools_available": true
  }
}
```

---

### 4.3 Technician Matching Engine (Agent 3)
- **Endpoint**: `POST /ai/match-technicians`
- **Input Payload**:
```json
{
  "request_id": "req_5501",
  "required_skills": ["Mechanical Vibration", "Compressor Overhaul"],
  "site_latitude": 13.0827,
  "site_longitude": 80.2707,
  "target_window_start": "2026-10-07T14:00:00Z"
}
```
- **Output Schema**:
```json
{
  "ranked_candidates": [
    {
      "technician_id": "tech_88",
      "total_score": 92.5,
      "score_components": {
        "skill_match": 100.0,
        "availability": 90.0,
        "sla_feasibility": 95.0,
        "distance_score": 85.0,
        "machine_familiarity": 90.0,
        "workload_score": 90.0,
        "performance_score": 95.0
      },
      "explanation": "Qualified, 12km from site, completed 4 similar compressor overhaul jobs."
    }
  ]
}
```

---

### 4.4 Exception Intelligence Agent (Agent 4)
- **Endpoint**: `POST /ai/detect-exceptions`
- **Input Payload**:
```json
{
  "event_type": "TECHNICIAN_UNAVAILABLE",
  "request_id": "req_5501",
  "current_status": "DISPATCHED",
  "context": {
    "technician_id": "tech_88",
    "dropout_reason": "Vehicle breakdown",
    "time_to_sla_breach_minutes": 80
  }
}
```
- **Output Schema**:
```json
{
  "exception_detected": true,
  "type": "TECH_DROPOUT",
  "severity": "CRITICAL",
  "recommended_action": "TRIGGER_DYNAMIC_REASSIGNMENT",
  "proposed_candidates": ["tech_99", "tech_104"],
  "explanation": "Tech 88 cannot reach site. Reassigning immediately to Tech 99 preserves SLA compliance."
}
```

---

### 4.5 SLA Risk Engine (Agent 5)
- **Endpoint**: `POST /ai/sla-risk`
- **Input Payload**:
```json
{
  "request_id": "req_5501",
  "current_time": "2026-10-07T13:30:00Z",
  "sla_deadline": "2026-10-07T15:22:00Z",
  "assigned_technician_eta": "2026-10-07T14:15:00Z",
  "estimated_job_duration_hours": 2.5
}
```
- **Output Schema**:
```json
{
  "risk_category": "AMBER",
  "breach_probability": 0.42,
  "estimated_finish_time": "2026-10-07T16:45:00Z",
  "projected_delay_minutes": 83,
  "mitigation_suggestion": "Authorize parallel assistant technician or expedite parts delivery."
}
```

---

### 4.6 Operations Copilot Agent (Agent 8)
- **Endpoint**: `POST /ai/copilot`
- **Input Payload**:
```json
{
  "user_id": "usr_101",
  "query": "Why is job REQ-2026-1042 on machine M-104 at risk of SLA breach?",
  "active_context_request_id": "req_5501"
}
```
- **Output Schema**:
```json
{
  "answer": "Job REQ-2026-1042 is at AMBER risk because original technician Sarah Connor experienced a vehicle breakdown at 13:40. Replacement technician Tech 99 is en route with an ETA of 14:15.",
  "recommended_actions": [
    {
      "label": "Approve Reassignment to Tech 99",
      "action_endpoint": "/api/v1/service-requests/req_5501/reassign",
      "payload": { "new_technician_id": "tech_99" }
    }
  ],
  "referenced_entities": [
    { "type": "MACHINE", "id": "mch_104", "code": "M-104" },
    { "type": "REQUEST", "id": "req_5501", "code": "REQ-2026-1042" }
  ]
}
```

---

### 4.7 RAG Diagnostic Knowledge Agent (Agent 7)
- **Endpoint**: `POST /ai/rag/query`
- **Input Payload**:
```json
{
  "machine_model": "Heavy Industrial Compressor 900",
  "error_code": "ERR-VIBE-E17",
  "query": "How to calibrate shaft bearing tension after replacement?"
}
```
- **Output Schema**:
```json
{
  "steps": [
    "Step 1: Ensure locknut torque is set to 45 Nm using torque wrench TW-200.",
    "Step 2: Spin shaft manually to ensure zero axial play.",
    "Step 3: Measure radial clearance using feeler gauge (spec: 0.03mm - 0.05mm)."
  ],
  "source_documents": [
    { "doc_name": "Compressor_900_Service_Manual.pdf", "page": 42 }
  ],
  "safety_warnings": [
    "DANGER: Lockout main power supply before checking clearance."
  ]
}
```

---

## 5. Real-Time WebSocket Event Specs

### 5.1 Connection URL
`wss://api.dqbh.local/ws/v1/events?token=<JWT_TOKEN>`

### 5.2 Event Wrapper Schema
```json
{
  "event_id": "evt_990182",
  "event_type": "WORK_ORDER_STATUS_CHANGED",
  "timestamp": "2026-10-07T13:42:00Z",
  "payload": { ... }
}
```

### 5.3 Event Types & Payloads

#### Event 1: `WORK_ORDER_STATUS_CHANGED`
```json
{
  "event_type": "WORK_ORDER_STATUS_CHANGED",
  "payload": {
    "request_id": "req_5501",
    "request_no": "REQ-2026-1042",
    "previous_status": "IN_PROGRESS",
    "new_status": "COMPLETED_PENDING_VERIFICATION",
    "updated_by": "tech_88"
  }
}
```

#### Event 2: `EXCEPTION_RAISED`
```json
{
  "event_type": "EXCEPTION_RAISED",
  "payload": {
    "exception_id": "exp_7001",
    "request_id": "req_5501",
    "request_no": "REQ-2026-1042",
    "exception_type": "TECH_DROPOUT",
    "severity": "CRITICAL",
    "message": "Technician vehicle breakdown reported."
  }
}
```

#### Event 3: `TECHNICIAN_LOCATION_UPDATED`
```json
{
  "event_type": "TECHNICIAN_LOCATION_UPDATED",
  "payload": {
    "technician_id": "tech_99",
    "latitude": 13.0840,
    "longitude": 80.2720,
    "eta_seconds": 900
  }
}
```

#### Event 4: `SLA_BREACH_ALERT`
```json
{
  "event_type": "SLA_BREACH_ALERT",
  "payload": {
    "request_id": "req_5501",
    "risk_category": "RED",
    "projected_delay_minutes": 45
  }
}
```

---

## 6. Database Entity Alignment Table

| Database Table | Primary API Endpoints | Responsible System / Layer | Key Constraints |
| :--- | :--- | :--- | :--- |
| `users`, `roles` | `POST /auth/login` | Core Auth | Foreign key roles; JWT claim matching |
| `machines`, `sites` | `GET /machines`, `GET /machines/{id}/passport` | Master Data / Core | Unique `asset_code` per site |
| `service_requests` | `POST /service-requests`, `PATCH /service-requests/{id}/status` | State Machine / Backend | Strict transition checks in state machine |
| `request_ai_analysis` | `POST /service-requests/{id}/classify` | AI Agent 1 (Classification) | Foreign key `request_id`; JSONB AI payload |
| `technicians`, `skills` | `GET /service-requests/{id}/candidates` | Matching Engine (Agent 3) | Spatial location indexing (`lat/lng`) |
| `inventory_balances` | `GET /inventory/availability` | Resource Engine | `CHECK (available_qty >= 0)` constraint |
| `resource_reservations` | `POST /service-requests/{id}/reserve` | Transactional Reservation | Exclusive slot locking via DB transactions |
| `assignments` | `POST /service-requests/{id}/dispatch` | Core Workflow | Unique active assignment per request |
| `work_checklist` | `PUT /service-requests/{id}/checklist` | Field Technician Execution | Mandatory items verified before completion |
| `exception_events` | `GET /exceptions`, `POST /service-requests/{id}/reassign` | Exception Engine (Agent 4) | Audit logged on status change |
| `audit_logs` | All status transitions & approvals | Security & Compliance | Immutable, insert-only table |

---

## 7. Parallel Team Workflow Responsibilities

```
                                    ┌────────────────────────┐
                                    │    API CONTRACTS MD    │
                                    └───────────┬────────────┘
                                                │
       ┌──────────────────┬─────────────────────┼─────────────────────┬──────────────────┐
       ▼                  ▼                     ▼                     ▼                  ▼
┌──────────────┐   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐   ┌──────────────┐
│  FRONTEND    │   │   BACKEND    │      │  AI AGENTS   │      │   DATABASE   │   │    DEVOPS    │
│  - 4 Views   │   │ - FastAPI    │      │ - 12 Agents  │      │ - PostgreSQL │   │ - Docker     │
│  - WS Listen │   │ - State Mach │      │ - JSON Schemas│     │ - Migration  │   │ - Mock Server│
│  - Forms     │   │ - REST API   │      │ - RAG / LLM  │      │ - Seed Data  │   │ - CI/CD      │
└──────────────┘   └──────────────┘      └──────────────┘      └──────────────┘   └──────────────┘
```

1. **Frontend Team**: Build UI views against mock endpoints conforming to the schemas in Section 3, 4 & 5.
2. **Backend Team**: Implement FastAPI routes, PostgreSQL state machine, and deterministic rule validation in Section 3 & 6.
3. **AI/Agent Team**: Implement Python agent modules adhering strictly to JSON request/response contracts in Section 4.
4. **Database Team**: Provision PostgreSQL schema, constraints, and seed data matching tables in Section 6.

---
*End of API Contracts Document*
