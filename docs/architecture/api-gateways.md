---
title: API Architecture & Integration Guide
status: reference
verified_against: e3e24f8ed
verified_on: 2026-09-11
---

# API Architecture & Integration Guide

This document provides a comprehensive guide to the REST APIs and integration services within the Allu platform, focusing on the **Internal UI REST API** (`allu-ui-service`) and the **External System API** (`external-service`), as well as the **Field Supervision API** (`supervision-api`).

---

## 1. System Topology & API Overview

Allu adopts a distributed service-oriented architecture. Public and client-facing traffic is divided across three dedicated Spring Boot services, which orchestrate business logic via a shared library (`service-core`) and delegate persistence, search, and document rendering to internal backends.

```mermaid
graph TB
    subgraph Clients["Client Applications"]
        UI["Angular Web UI<br/>(City Officials & Handlers)"]
        EXT["External B2B Systems<br/>(Utilities, Contractors, E-Services)"]
        VALKO["Field Inspection Apps<br/>(Supervisors / Valko)"]
    end

    subgraph Gateways["Client-Facing Services"]
        UIService["allu-ui-service<br/>(Port 9000)<br/>Internal UI REST API"]
        ExtService["external-service<br/>(Port 9040)<br/>External System API (v1 / v2)"]
        SupervisionService["supervision-api<br/>(Port 9050)<br/>Field Supervision API"]
    end

    subgraph Core["Shared Business & Orchestration Layer"]
        ServiceCore["service-core / service-core-domain<br/>(Validation, Event Dispatcher, Mappers)"]
    end

    subgraph Engines["Internal Backend Engines"]
        ModelService["model-service<br/>(Port 9010)<br/>Spring Data / QueryDSL"]
        SearchService["search-service<br/>(Port 9020)<br/>Elasticsearch Client"]
        PdfService["pdf-service<br/>(Port 9030)<br/>wkhtmltopdf Engine"]
    end

    subgraph Data["Persistence & Search"]
        Postgres[(PostgreSQL<br/>allu schema)]
        ES[(Elasticsearch<br/>allu-node)]
    end

    UI -->|HTTP /api| UIService
    EXT -->|HTTP /v1, /v2| ExtService
    VALKO -->|HTTP /v1| SupervisionService

    UIService --> ServiceCore
    ExtService --> ServiceCore
    SupervisionService --> ServiceCore

    ServiceCore -->|REST / JSON| ModelService
    ServiceCore -->|REST / JSON| SearchService
    ServiceCore -->|REST / JSON| PdfService

    ModelService --> Postgres
    SearchService --> ES
```

### Microservice Port Map

| Service | Port | Audience | Documentation / Spec |
| --- | --- | --- | --- |
| `allu-ui-service` | `9000` | Angular frontend client | Internal code contracts (`*Json`) |
| `external-service` | `9040` | Third-party systems, utilities, e-services | OpenAPI 3 / Swagger (`/swagger-ui.html`) |
| `supervision-api` | `9050` | Field inspectors, mobile clients | OpenAPI 3 / Swagger (`/swagger-ui.html`) |
| `model-service` | `9010` | Internal only (CRUD engine) | Internal service REST |
| `search-service` | `9020` | Internal only (Elasticsearch queries) | Internal service REST |
| `pdf-service` | `9030` | Internal only (PDF generator) | Internal service REST |

Ports appear here for orientation alongside each API's audience and spec.
[environments.md](environments.md) §4 is canonical for the deployment port map —
change it there first.

---

## 2. Internal UI REST API (`allu-ui-service`)

The UI service powers the main back-office application used by City of Helsinki handlers, decision-makers, and administrators.

### 2.1 Authentication & Roles

- **Production:** Microsoft Azure Active Directory / ADFS OAuth2 authorization code flow.
- **Local Dev / Test:** `POST /auth/login` returning an HMAC-signed JWT Bearer token valid for 12 hours.
- **Role Permissions (`RoleType`):**
  - `ROLE_VIEW`: Search and read-only access to applications, history, and attachments.
  - `ROLE_CREATE_APPLICATION`: Internal application creation.
  - `ROLE_PROCESS_APPLICATION`: Application intake, handling, geometry modification, customer assignment.
  - `ROLE_DECISION`: Issuing binding permit decisions (proposals, approval, rejection).
  - `ROLE_SUPERVISE`: Scheduling and marking supervision tasks.
  - `ROLE_INVOICING`: Reviewing and releasing invoice rows to SAP.
  - `ROLE_ADMIN`: Global system configuration, pricing tariffs, and user role management.

### 2.2 Core Controller Responsibilities

```mermaid
graph LR
    subgraph UI_Controllers["allu-ui-service Controllers"]
        APP["ApplicationController<br/>& ApplicationDraftController"]
        APP_STAT["ApplicationStatusController<br/>(State Transitions)"]
        APPR["ApprovalController<br/>(Decisions & PDF Distribution)"]
        CHRG["ChargeBasisController<br/>& InvoicingPeriodController"]
        INFO["InformationRequestController<br/>(Applicant Queries)"]
        SUP["SupervisionTaskController<br/>(Inspections)"]
        WFS["WFSController<br/>(GeoServer Proxy)"]
    end

    APP --> APP_STAT
    APP_STAT --> APPR
    APP --> CHRG
    APP --> INFO
    APP --> SUP
    APP --> WFS
```

- **`ApplicationController`**: Complete lifecycle CRUD, search filters (`POST /applications/search`), history log, and ownership changes.
- **`ApprovalController`**: Compiles decision proposals, generates preview PDFs, and sends official decisions via email to distribution lists (`POST /applications/{id}/decision/send`).
- **`ChargeBasisController` & `InvoicingPeriodController`**: Manages fee rows, manual overrides, discounts, and multi-period billing splits.
- **`WFSController`**: Proxies requests to Helsinki's open and restricted GeoServer layers (`kartta.hel.fi`), providing address geocoding, district lookups, and street payment classes.

---

## 3. External System API (`external-service`)

`external-service` is the public B2B gateway allowing third parties—such as infrastructure utility companies (Helen, Elisa, Telia, DNA, HSY), engineering firms, and the city's citizen portal—to programmatically submit and track applications.

### 3.1 Authentication & Identity Flow

External systems authenticate using dedicated API credentials to obtain a scoped JWT Bearer token:

```mermaid
sequenceDiagram
    autonumber
    actor Client as External System (Client)
    participant Auth as external-service (/v1/login)
    participant Core as service-core
    participant DB as model-service

    Client->>Auth: POST /v1/login { username, password }
    Auth->>Core: authenticate(username, password)
    Core->>DB: findByUsername(username)
    DB-->>Core: ExternalUser + HashedPassword + Roles
    Core-->>Auth: Authentication Success (ExternalUser)
    Auth-->>Client: 200 OK "Bearer <JWT_TOKEN>"

    Note over Client, Auth: Subsequent requests pass Authorization: Bearer <JWT_TOKEN>
    Client->>Auth: GET /v1/applications/{id}
    Auth->>Auth: Validate Token + Verify Ownership (externalOwnerId)
    Auth-->>Client: 200 OK ApplicationExt JSON
```

### 3.2 Roles & Multi-Tenant Isolation

- **`ROLE_SERVICE`**: Automated machine-to-machine integrations.
- **`ROLE_TRUSTED_PARTNER`**: Certified utility operators and major contractors with extended capabilities.
- **`ROLE_INTERNAL`**: External users integrated within the municipal network.
- **Tenant Isolation:**
  Every application created via the External API stores an `externalOwnerId`. All operations (`GET`, `PUT`, uploads) execute `validateOwnedByExternalUser(applicationId)`. If an external user attempts to access an application owned by another organization, the request is rejected with `403 Forbidden` (`application.ext.notowner`).

---

## 4. External Application Lifecycle & Modification Rules

To prevent data corruption while city officials review permits, Allu strictly enforces a state-dependent mutation model.

```mermaid
stateDiagram-v2
    [*] --> PENDING_CLIENT: POST /v1/{type}<br/>pendingOnClient = true
    [*] --> PENDING: POST /v1/{type}<br/>pendingOnClient = false

    state "Client Editable Phase" as EditPhase {
        PENDING_CLIENT --> PENDING_CLIENT: Full PUT allowed
        PENDING_CLIENT --> PENDING: Client finalizes draft
        PENDING --> PENDING: Full PUT allowed
    }

    PENDING --> HANDLING: Official begins review

    state "City Processing Phase" as LockedPhase {
        HANDLING --> WAITING_INFORMATION: City issues Info Request
        WAITING_INFORMATION --> INFORMATION_RECEIVED: Client responds via Info Request API
        INFORMATION_RECEIVED --> HANDLING: City reviews response
        HANDLING --> DECISIONMAKING: Prepared for signoff
        DECISIONMAKING --> DECISION: Approved
    }

    note right of LockedPhase
        Full PUT /v1/{type}/{id} is BLOCKED!
        Modifications must use:
        1. Information Request Response
        2. Report Application Change (Tag: OTHER_CHANGES)
    end note
```

### 4.1 Creation (`POST /v1/{type}`)

- **Payload:** Type-specific DTO (`ExcavationAnnouncementExt`, `CableReportExt`, `ShortTermRentalExt`, etc.).
- **Drafting (`pendingOnClient`):**
  - When `pendingOnClient: true`, status is set to `PENDING_CLIENT`. City staff will not process it, allowing the client system to freely update the payload.
  - When `pendingOnClient: false`, status moves to `PENDING`, queuing it in city work queues.

### 4.2 Full Updates (`PUT /v1/{type}/{id}`)

- Allowed **only** while the application is in `PENDING_CLIENT` or `PENDING`.
- Once city handling begins (`HANDLING` or later), a full `PUT` request is rejected with `application.ext.notpending`.

### 4.3 Handling Customer Changes via Information Requests

When city handlers require modifications, or when applicants must update an active permit, communication occurs through structured Information Requests:

```mermaid
sequenceDiagram
    autonumber
    actor Handler as City Official (allu-ui-service)
    actor Client as External Client (external-service)
    participant Core as service-core / DB

    Handler->>Core: Create InformationRequest (specifying required fields/attachments)
    Core->>Core: Change status to WAITING_INFORMATION
    Client->>Core: GET /v1/applications/{id}/informationrequests
    Core-->>Client: List of open InformationRequests & required fields

    Note over Client: Client modifies requested data or attaches documents
    Client->>Core: POST /v1/applications/{id}/informationrequests/{reqId}/response<br/>{ responseData, updatedFields }
    Core->>Core: Change status to INFORMATION_RECEIVED
    Core->>Handler: Notify Handler of received update
    Handler->>Core: Review & merge changes into active application
```

---

## 5. Post-Decision Life-Cycle & Milestone Reporting

For excavation announcements (`Kaivuilmoitus`), contractors report execution milestones directly through the External API:

```mermaid
sequenceDiagram
    autonumber
    actor Contractor as Digging Contractor (External API)
    participant Ext as external-service
    participant Core as ExcavationAnnouncementStatusChangeHandler
    actor Supervisor as City Inspector (supervision-api)

    Note over Contractor, Supervisor: Phase 1: Operational Condition (Base asphalt for winter)
    Contractor->>Ext: PUT /v1/excavationannouncements/{id}/operationalcondition<br/>{ operationalConditionDate }
    Ext->>Core: reportCustomerOperationalCondition()
    Core->>Core: Schedule OPERATIONAL_CONDITION Supervision Task

    Supervisor->>Ext: Approve Operational Condition (supervision-api)
    Core->>Core: Status -> OPERATIONAL_CONDITION
    Core->>Core: Close & Lock Invoicing Period 1 (Area fees billed to winter start)

    Note over Contractor, Supervisor: Phase 2: Final Work Completion (Permanent restoration)
    Contractor->>Ext: PUT /v1/excavationannouncements/{id}/workfinished<br/>{ workFinishedDate }
    Ext->>Core: reportCustomerWorkFinished()
    Core->>Core: Schedule FINAL_SUPERVISION Task

    Supervisor->>Ext: Approve Final Supervision (supervision-api)
    Core->>Core: Status -> FINISHED
    Core->>Core: Lock Final Invoices & Start 2-Year Warranty Clock (WARRANTY Task)
```

### Available External Milestone Endpoints

- `PUT /v1/excavationannouncements/{id}/operationalcondition` — Report winter operational readiness.
- `PUT /v1/excavationannouncements/{id}/workfinished` — Report final surface completion.
- `PUT /v1/excavationannouncements/{id}/validityperiod` — Request an extension to the permit dates.
- `GET /v1/excavationannouncements/{id}/approval/operationalcondition` — Download operational condition PDF certificate.
- `GET /v1/excavationannouncements/{id}/approval/workfinished` — Download final completion certificate.
- `GET /v1/documents/applications/{id}/decision` — Download official decision PDF.

---

## 6. Field Supervision API (`supervision-api`, Port 9050)

`supervision-api` is a specialized interface optimized for city inspectors working on tablets and smartphones on construction sites.

```mermaid
graph LR
    subgraph Supervision_Capabilities["supervision-api Capabilities"]
        TASKS["SupervisionTaskController<br/>- Find tasks by district/date<br/>- Task details & history"]
        APPROVE["SupervisionTaskApprovalService<br/>- Approve / Reject tasks<br/>- Record inspection notes"]
        PHOTOS["AttachmentController<br/>- Upload on-site photos<br/>- Attach inspection reports"]
        TAGS["ApplicationTagController<br/>- Set tags: SUPERVISION_DONE,<br/>PRELIMINARY_SUPERVISION_DONE"]
    end
```

- **Port:** `9050`
- **Swagger / OpenAPI:** Available at `http://localhost:9050/swagger-ui.html`.
- **Target Workflows:**
  - Finding pending site visits by municipal district (`/v1/supervisiontasks/search`).
  - Executing preliminary checks (*Aloitusvalvonta*), operational condition checks (*Toiminnallisen kunnon valvonta*), final checks (*Loppuvalvonta*), and 2-year warranty checks (*Takuuvalvonta*).
  - Approving or rejecting tasks with reason codes and corrective deadlines.
  - Snapping on-site photos and attaching them directly to the application file.

---

## 7. Comparative Feature Matrix

| Feature / Dimension | UI REST API (`allu-ui-service`) | External System API (`external-service`) | Field Supervision API (`supervision-api`) |
| --- | --- | --- | --- |
| **Port** | `9000` | `9040` | `9050` |
| **Primary Consumer** | Angular Web App | B2B Integrators / E-Services | Mobile / Field Inspector Web Apps |
| **Authentication** | Azure AD / OAuth2 + Local JWT | Username/Password Exchange to JWT | Azure AD / OAuth2 + Local JWT |
| **OpenAPI / Swagger** | Not exposed | Fully documented (`/swagger-ui.html`) | Fully documented (`/swagger-ui.html`) |
| **Access Boundary** | Global across all city applications | Restricted to caller's own applications | Scoped to assigned tasks / districts |
| **Full Entity Updates** | Supported across entire lifecycle | Only while in `PENDING` / `PENDING_CLIENT` | Not supported (Inspection notes only) |
| **Document Generation** | Generates, modifies, and emails PDFs | Read-only download of finalized PDFs | Views decisions and attaches photos |
| **Invoicing Operations** | Releasing, locking, updating SAP rows | No invoicing access | Triggers invoice locking via approvals |

---

## 8. Code Reference Map

| Component | Repository Path |
| --- | --- |
| **UI REST Controllers** | `backend/allu-ui-service/src/main/java/fi/hel/allu/ui/controller/` |
| **UI Security Config** | `backend/allu-ui-service/src/main/java/fi/hel/allu/ui/config/SecurityConfig.java` |
| **External REST Controllers** | `backend/external-service/src/main/java/fi/hel/allu/external/controller/api/` |
| **External Security & Token** | `backend/external-service/src/main/java/fi/hel/allu/external/config/SecurityConfig.java` |
| **External Swagger Config** | `backend/external-service/src/main/java/fi/hel/allu/external/config/SwaggerConfig.java` |
| **External DTO Models** | `backend/external-domain/src/main/java/fi/hel/allu/external/domain/` |
| **Supervision Controllers** | `backend/supervision-api/src/main/java/fi/hel/allu/supervision/api/controller/` |
| **Supervision Swagger Config** | `backend/supervision-api/src/main/java/fi/hel/allu/supervision/api/config/SwaggerConfig.java` |
| **Shared Orchestration** | `backend/service-core/src/main/java/fi/hel/allu/servicecore/service/` |
