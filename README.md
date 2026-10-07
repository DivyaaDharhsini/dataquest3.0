# DataQuest 3.0: Next-Generation Activity Management Platform

## Overview
An intelligent, centralized, extensible full-stack activity management platform for industrial equipment servicing and maintenance across multi-site operations. 

This platform acts as a service orchestration engine that receives maintenance requests, validates them, predicts required resources, assigns the best technician, coordinates parts/tools, tracks execution, detects exceptions, dynamically reassigns work, verifies completion, and builds a permanent digital service history.

## Core Capabilities
- **Lifecycle Management**: End-to-end tracking of service requests (reactive, periodic, predictive).
- **AI-Powered Orchestration**: Intelligent classification, technician matching, and spare-part prediction.
- **Resource Management**: Real-time tracking of technician availability, workload, and inventory stock.
- **SLA & Exception Engine**: Automated risk detection, dynamic reassignment, and strict SLA enforcement.
- **Service Passport**: Immutable digital service history for every machine.

## Recommended Tech Stack (Hackathon Blueprint)
- **Frontend**: React + Vite + Tailwind CSS
- **Backend**: FastAPI + Python (Modular Monolith)
- **Database**: PostgreSQL (Relational schema, JSONB, pgvector)
- **Cache/Locks**: Redis
- **Background Jobs**: Celery / RQ
- **Object Storage**: S3-compatible / Supabase Storage

## Documentation
The following documents define the core behavior and architecture for this implementation:

- [API Contract & Workflow](API_CONTRACT.md): Details the REST APIs, API conventions, core workflow state machine, AI boundaries, and implementation dependencies.
- [RBAC & Permissions](RBAC_PERMISSION_MATRIX.md): Defines user roles, access scoping (site vs. global), and operation authorization matrices.

## Getting Started

*(Further instructions on local setup, database seeding, and environment configuration will be added here as the implementation progresses.)*

## Key Architectural Principles
1. **AI as Advisor, Backend as Enforcer**: AI models interpret and recommend (classification, scoring, prediction), but deterministic backend services validate rules, reserve inventory, and mutate operational state.
2. **Modular Monolith**: Build within distinct domain modules (Auth, Inventory, Execution, Exception) to allow future microservice extraction if needed.
3. **Data Integrity**: Enforce constraints strictly at the database layer (no negative inventory, no conflicting reservations) and log all significant transitions into an immutable audit trail.
