# FHIR Server Setup & Care Gaps Architecture 

## 1. Introduction

This document explains:

* How to set up a **HAPI FHIR JPA Server**
* Core **FHIR server capabilities** (search, sort, filter, pagination)
* How to **ingest data into FHIR**
* How **Care Gaps** are modeled using FHIR resources
* Why a **Proxy (Aggregation) Server** is required
* How Frontend (FE) should interact with FHIR & Proxy
* Recommended **Frontend libraries** (bonFHIR, etc.)

Repository used:
👉 [https://github.com/hapifhir/hapi-fhir-jpaserver-starter](https://github.com/hapifhir/hapi-fhir-jpaserver-starter)

---

## 2. Setting Up HAPI FHIR JPA Server

### 2.1 Prerequisites

* Java 17+
* Maven
* PostgreSQL 
* Docker 

### 2.2 Clone and Run

```bash
git clone https://github.com/hapifhir/hapi-fhir-jpaserver-starter.git
cd hapi-fhir-jpaserver-starter
mvn spring-boot:run
```

Server will be available at:

```
http://localhost:8080/fhir
```

## 3. FHIR Server Capabilities

### 3.1 Searching

FHIR supports rich search out of the box.

Examples:

```http
GET /Patient?name=John
GET /DetectedIssue?patient=Patient/123
```

### 3.2 Filtering

```http
GET /DetectedIssue?status=identified
GET /Task?status=in-progress
```

### 3.3 Sorting

```http
GET /DetectedIssue?_sort=-identified-date
```

### 3.4 Pagination

```http
GET /DetectedIssue?_count=20&page=2
```

### 3.5 Includes (Reference Traversal)

```http
GET /DetectedIssue?patient=Patient/123&_include=DetectedIssue:patient
```

### 3.6 Counts

```http
GET /DetectedIssue?status=identified&_summary=count
```

⚠️ FHIR does **not** support GROUP BY or complex joins.

---

## 4. Ingesting Data into FHIR Database

### 4.1 REST-based Ingestion

```http
POST /Patient
POST /DetectedIssue
POST /Task
```

### 4.2 Bundles (Bulk Insert)

```http
POST /Bundle
```

Used for bulk migration.

## 5. Care Gaps Modeling

### 5.1 Selected FHIR Resources

| Concept            | FHIR Resource                   |
| ------------------ | ------------------------------- |
| Patient            | Patient                         |
| Provider           | Practitioner / PractitionerRole |
| Organization       | Organization                    |
| Care Gap           | DetectedIssue                   |
| Follow-up Workflow | Task                            |

---

## 6. Care Gaps Field Mapping 

### 6.1 DetectedIssue (Care Gap)
| Field              | FHIR Mapping                       | Notes (Justification)                                                                                                                                                            |
| ------------------ | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **GapId**          | `DetectedIssue.identifier.value`   | `identifier` is intended for **business or external IDs** that are stable across systems. Using it avoids overloading the technical FHIR `id`, which is server-generated and not portable. |
| **CareGapId**      | `DetectedIssue.code.coding.code`   | `code` represents the **type or classification** of the detected issue. Care gaps are categorized (e.g., “HbA1c overdue”, “Missing screening”), which aligns directly with `CodeableConcept`. |
| **Name**           | `DetectedIssue.code.text`          | `code.text` provides a **human-readable display name** for the care gap, suitable for UI display without requiring terminology resolution.                                       |
| **Status**         | `DetectedIssue.status`             | FHIR provides lifecycle status for detected issues. Business values such as *open* and *closed* are mapped to standard FHIR statuses (e.g., `identified` = open, `resolved` = closed) to remain interoperable. |
| **Severity**       | `DetectedIssue.severity`           | FHIR natively supports severity levels (`high`, `moderate`, `low`), which aligns with care gap prioritization logic and avoids custom extensions.                                |
| **Reason**         | Extension `care-gap-reason`        | FHIR does not provide a native field for **business or workflow-specific gap reasons**. Extensions are the recommended mechanism for capturing non-clinical rationale without misusing clinical elements. |
| **ReasonDetails**  | `DetectedIssue.detail`             | `detail` is designed for **free-text explanatory information** describing the issue. It appropriately captures additional context without requiring structured coding.           |
| **DocumentedDate** | `DetectedIssue.identifiedDateTime` | This element records **when the issue was identified**, which directly corresponds to when the care gap was documented in the system.                                            |
| **GapResolution**  | Extension `care-gap-resolution`    | FHIR’s native mitigation elements focus on **clinical mitigation**, not business resolution outcomes. An extension avoids semantic misuse while preserving auditability.         |
| **ClosureType**    | Extension `care-gap-closure-type`  | FHIR does not model **how** a care gap was closed (manual, system, patient refusal, etc.). This is business-specific and correctly handled via extension.                        |
| **GapMetDate**     | Extension `care-gap-met-date`      | There is no native FHIR element to represent the **date a care gap was satisfied**. An extension cleanly captures this lifecycle milestone.                                      |
| **Notes**          | Extension `care-gap-notes`         | Notes are business/workflow-oriented rather than clinical annotations. Using an extension prevents conflation with clinician-authored notes and supports structured handling if needed. |
| **PatientEId**     | `DetectedIssue.subject`            | `subject` is the canonical FHIR reference for the **entity affected by the issue**.                            |



### 6.2 Task (Care Coordination)
| Field                     | FHIR Mapping                       | Notes (Justification)                                                                                                                                                                                  |
| ------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **NextSteps**             | `Task.description`                 | `description` is intended to capture **what action needs to be performed**. It provides a human-readable summary of the follow-up steps and is required for workflow clarity.                          |
| **AppointmentDate**       | `Task.executionPeriod.start`       | `executionPeriod` represents **when the task is expected to occur**. Using `start` aligns with scheduled appointment or follow-up initiation dates.                                                    |
| **AppointmentOption**     | Extension `appointment-option`     | FHIR does not natively represent **UI- or workflow-specific appointment choices** (e.g., in-person, virtual, patient preference). An extension avoids misusing clinical elements such as `reasonCode`. |
| **CommunicationStatus**   | Extension `communication-status`   | Task status reflects the lifecycle of the task itself, not **outreach progress**. Communication state is business-specific and therefore correctly modeled via an extension.                           |
| **CommunicationAttempts** | Extension `communication-attempts` | FHIR provides no native element to store **numeric outreach attempt counts**. Using an extension preserves computability without overloading notes or status fields.                                   |
| **AssignedTo**            | `Task.owner`                       | `owner` identifies the **individual or role responsible** for completing the task. Referencing `PractitionerRole` supports role-based assignment and organizational context.                           |
| **PatientEid**            | `Task.for`                         | `for` is the canonical FHIR element indicating **who the task is being performed for**. Referencing `Patient` enables standard search and reference traversal.                                         |
| **GapId**                 | `Task.focus` → `DetectedIssue`     | `focus` is designed to reference **the resource the task is acting upon**. Linking to `DetectedIssue` explicitly ties the task to the originating care gap, preserving traceability.                   |

---

## 7. Linking Patient & Provider Details

### 7.1 PractitionerRole (Recommended)

PractitionerRole links:

* Practitioner
* Organization
* Role

```text
Patient → generalPractitioner → PractitionerRole
Task → owner → PractitionerRole
```

This allows retrieving:

* Provider Name
* Organization Name

Using `_include` queries.

---

## 8. Why a Proxy (Aggregation) Server is Required

### 8.1 Problem

FHIR cannot return:

* Patient + Provider + Organization
* Care gaps + tasks
* Open / Closed counts in a **single query**.

### 8.2 Proxy Server Responsibilities

* Call FHIR multiple times
* Aggregate responses
* Compute counts
* Apply business rules
* Return UI-friendly JSON

---

## 10. Frontend Interaction Strategy

### Recommended

```
Frontend
 ├── Direct → FHIR (CRUD, simple search)
 └── Proxy → Aggregated APIs (workque+ overView page)
```

This avoids unnecessary latency and keeps FHIR pure.

---

## 11. Frontend Libraries (FHIR JSON Handling)

### 11.1 bonFHIR (Recommended)

* Type-safe FHIR resources
* Clean builders
* Validation support

👉 [https://bonfhir.dev](https://bonfhir.dev)

### 11.2 Alternatives

* fhirclient (SMART on FHIR)
* custom TypeScript models



----


# FHIR Server Setup & Care Gaps Architecture Guide

## 1. Introduction

> **References**
>
> * HL7 FHIR Specification: [https://hl7.org/fhir/](https://hl7.org/fhir/)
> * HAPI FHIR Documentation: [https://hapifhir.io/hapi-fhir/docs/](https://hapifhir.io/hapi-fhir/docs/)

## 1. Introduction

This document explains:

* How to set up a **HAPI FHIR JPA Server**
* Core **FHIR server capabilities** (search, sort, filter, pagination)
* How to **ingest data into FHIR**
* How **Care Gaps** are modeled using FHIR resources
* Why a **Proxy (Aggregation) Server** is required
* How Frontend (FE) should interact with FHIR & Proxy
* Recommended **Frontend libraries** (bonFHIR, etc.)

This is intended as a **handover document** for leadership, architects, and senior engineers.

Repository used:
👉 [https://github.com/hapifhir/hapi-fhir-jpaserver-starter](https://github.com/hapifhir/hapi-fhir-jpaserver-starter)

---

## 2. Setting Up HAPI FHIR JPA Server

### 2.1 Prerequisites

* Java 17+
* Maven
* PostgreSQL (recommended for production)
* Docker (optional)

### 2.2 Clone and Run

```bash
git clone https://github.com/hapifhir/hapi-fhir-jpaserver-starter.git
cd hapi-fhir-jpaserver-starter
mvn spring-boot:run
```

Server will be available at:

```
http://localhost:8080/fhir
```

### 2.3 Key Components

* **JPA Layer** – persists all FHIR resources
* **REST API** – standard FHIR endpoints
* **Search Indexes** – automatic indexing of references and search params

---

## 3. FHIR Server Capabilities

> **Official References**
>
> * FHIR REST API: [https://hl7.org/fhir/http.html](https://hl7.org/fhir/http.html)
> * FHIR Search: [https://hl7.org/fhir/search.html](https://hl7.org/fhir/search.html)

## 3. FHIR Server Capabilities

### 3.1 Searching

FHIR supports rich search out of the box.

Examples:

```http
GET /Patient?name=John
GET /DetectedIssue?patient=Patient/123
```

### 3.2 Filtering

```http
GET /DetectedIssue?status=identified
GET /Task?status=in-progress
```

### 3.3 Sorting

```http
GET /DetectedIssue?_sort=-identified-date
```

### 3.4 Pagination

```http
GET /DetectedIssue?_count=20&page=2
```

### 3.5 Includes (Reference Traversal)

```http
GET /DetectedIssue?patient=Patient/123&_include=DetectedIssue:patient
```

### 3.6 Counts

```http
GET /DetectedIssue?status=identified&_summary=count
```

⚠️ FHIR does **not** support GROUP BY or complex joins.

---

## 4. Ingesting Data into FHIR Database

### 4.1 REST-based Ingestion

```http
POST /Patient
POST /DetectedIssue
POST /Task
```

### 4.2 Bundles (Bulk Insert)

```http
POST /Bundle
```

Used for bulk migration.

### 4.3 External System Ingestion

* ETL jobs
* Microservices pushing computed data
* Manual creation via UI

---

## 5. Care Gaps Modeling (Final Design)

### 5.1 Selected FHIR Resources

| Concept            | FHIR Resource                   |
| ------------------ | ------------------------------- |
| Patient            | Patient                         |
| Provider           | Practitioner / PractitionerRole |
| Organization       | Organization                    |
| Care Gap           | DetectedIssue                   |
| Follow-up Workflow | Task                            |

---

## 6. Care Gaps Field Mapping

> **Key References**
>
> * DetectedIssue: [https://hl7.org/fhir/detectedissue.html](https://hl7.org/fhir/detectedissue.html)
> * Task: [https://hl7.org/fhir/task.html](https://hl7.org/fhir/task.html)
> * Extensions: [https://hl7.org/fhir/extensibility.html](https://hl7.org/fhir/extensibility.html)

## 6. Care Gaps Field Mapping (Complete)

### 6.1 DetectedIssue (Care Gap)

| Business Field       | FHIR Element                        |
| -------------------- | ----------------------------------- |
| CareGapId            | DetectedIssue.id                    |
| CareGapName          | DetectedIssue.code.text             |
| Description          | DetectedIssue.code.text (narrative) |
| Status (Open/Closed) | DetectedIssue.status                |
| Severity             | DetectedIssue.severity              |
| Reason               | Extension (care-gap-reason)         |
| ReasonDetails        | DetectedIssue.detail                |
| DocumentedDate       | DetectedIssue.identifiedDateTime    |
| GapResolution        | DetectedIssue.mitigation            |
| PatientId            | DetectedIssue.patient               |
| GroupId              | Extension                           |

### 6.2 Task (Care Coordination)

| Business Field        | FHIR Element               |
| --------------------- | -------------------------- |
| TaskStatus            | Task.status                |
| NextSteps             | Task.description           |
| AppointmentDate       | Task.executionPeriod       |
| AppointmentOption     | Extension                  |
| CommunicationStatus   | Task.businessStatus        |
| CommunicationAttempts | Extension                  |
| AssignedProvider      | Task.owner                 |
| Patient               | Task.for                   |
| Linked Care Gap       | Task.focus → DetectedIssue |

---

## 7. Linking Patient & Provider Details

### 7.1 PractitionerRole (Recommended)

PractitionerRole links:

* Practitioner
* Organization
* Role

```text
Patient → generalPractitioner → PractitionerRole
Task → owner → PractitionerRole
```

This allows retrieving:

* Provider Name
* Organization Name

Using `_include` queries.

---

## 8. Why a Proxy (Aggregation) Server is Required

### 8.1 Problem

FHIR cannot return:

* Patient + Provider + Organization
* Care gaps + tasks
* Open / Closed counts

in a **single query**.

### 8.2 Proxy Server Responsibilities

* Call FHIR multiple times
* Aggregate responses
* Compute counts
* Apply business rules
* Return UI-friendly JSON

---

## 9. Proxy Server vs HAPI Custom Operations

| Aspect            | Proxy Server | HAPI $operation |
| ----------------- | ------------ | --------------- |
| Custom FHIR logic | ❌            | ✅               |
| Flexibility       | High         | Medium          |
| Vendor lock-in    | Low          | High            |
| Multi-DB support  | Easy         | Hard            |
| Recommended here  | ✅            | ❌               |

---

## 10. Frontend Interaction Strategy

### Recommended

```
Frontend
 ├── Direct → FHIR (CRUD, simple search)
 └── Proxy → Aggregated APIs (dashboards)
```

This avoids unnecessary latency and keeps FHIR pure.

---

## 11. Frontend Libraries (FHIR JSON Handling)

> **References**
>
> * bonFHIR: [https://bonfhir.dev](https://bonfhir.dev)
> * SMART on FHIR JS Client: [https://github.com/smart-on-fhir/client-js](https://github.com/smart-on-fhir/client-js)

## 11. Frontend Libraries (FHIR JSON Handling)

### 11.1 bonFHIR (Recommended)

* Type-safe FHIR resources
* Clean builders
* Validation support

👉 [https://bonfhir.dev](https://bonfhir.dev)

### 11.2 Alternatives

* fhirclient (SMART on FHIR)
* custom TypeScript models

---

## 12. Summary & Architectural Verdict

* HAPI FHIR used as **clinical data platform**
* Care gaps modeled using **DetectedIssue + Task**
* Proxy server handles **business logic & aggregation**
* Frontend uses both FHIR and Proxy
* Design is **FHIR-compliant, scalable, and future-proof**

---

## 13. Future Extensions

* Introduce Measure/MeasureReport if population analytics required
* Add CDS Hooks
* Add bulk export/import

---

**End of Document**

---

## 12. Example: Care Gap Represented in FHIR

### 12.1 DetectedIssue Example

```json
{
  "resourceType": "DetectedIssue",
  "identifier": [
    { "system": "urn:caregap", "value": "58980fe3-48ef-449c-96b2-9df1637814e9" }
  ],
  "status": "identified",
  "severity": "high",
  "code": {
    "coding": [
      { "system": "urn:caregap-type", "code": "4" }
    ],
    "text": "Screening for Future Fall Risk"
  },
  "subject": { "reference": "Patient/431224" },
  "detail": null,
  "identifiedDateTime": null,
  "extension": [
    { "url": "care-gap-reason", "valueString": null },
    { "url": "care-gap-closure-type", "valueString": null },
    { "url": "care-gap-met-date", "valueDate": null },
    { "url": "care-gap-notes", "valueString": "" }
  ]
}
```

### 12.2 Task Example

```json
{
  "resourceType": "Task",
  "status": "in-progress",
  "description": "Send Initial Communication",
  "for": { "reference": "Patient/431224" },
  "owner": { "reference": "PractitionerRole/12861393" },
  "focus": { "reference": "DetectedIssue/58980fe3-48ef-449c-96b2-9df1637814e9" },
  "executionPeriod": { "start": null },
  "extension": [
    { "url": "appointment-option", "valueString": null },
    { "url": "communication-status", "valueString": null },
    { "url": "communication-attempts", "valueInteger": 0 }
  ]
}
```
