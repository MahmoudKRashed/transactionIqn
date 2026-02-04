# Transaction Inquiry: Data Flow Architecture

This diagram illustrates the high-level integration between SAMA, the Bank Application, and the Data Lake for processing transaction inquiries.

---

## 1. End-to-End Data Flow (Sequence Diagram)

```mermaid
sequenceDiagram
    autonumber
    participant SAMA as "SAMA (Tanfeeth)"
    participant APP as "Tanfeeth App (Bank)"
    participant DL as "Data Lake Application"

    Note over SAMA, APP: Phase 1: Request Processing
    SAMA->>APP: FITransInqRq (Search Criteria)
    Note right of APP: Extract: Id, AccNum, TransNum, Date Range

    Note over APP: Phase 2: Internal Mapping
    APP->>APP: Translate SAMA TransTypes (01-15) to Bank Codes

    Note over APP, DL: Phase 3: Data Retrieval
    APP->>DL: Request Data based on Case (Id, AccNum, TransNum, Dates, Bank Types)
    DL-->>APP: Return Transaction Data List (Full SAMA-required attributes)

    Note over APP: Phase 4: SAMA Schema Mapping
    APP->>APP: Map Records to SAMA Callback Structure
    Note right of APP: Populate: AccInfo, CardInfo, MoreDtls

    Note over APP, SAMA: Phase 5: Callback Response
    APP->>SAMA: FITransInqCallBackRq (Bilingual Payload)
```

---

## 2. Component Responsibilities

| Component | Responsibility |
| :--- | :--- |
| **SAMA (Tanfeeth)** | Initiates inquiry via SOAP Request; Receives final Callback Response. |
| **Tanfeeth App** | Manages security, business logic, type mapping (SAMA ↔ Bank), and XML transformation. |
| **Data Lake** | Serves as the golden source for historical and current transaction records across all products. |

---

## 3. Data Point Mapping Summary

| Inquiry Parameter | App Processing Logic |
| :--- | :--- |
| **Id** | Used to filter records belonging to a specific National ID/Iqama. |
| **Account Number** | Direct filter on `AccInfo` records within the Data Lake. |
| **Transaction Number** | Used for Scenario A (Unique record retrieval). |
| **Date Range** | Applied to `TransDate` for duration-based searches (Scenario B). |
| **Transaction Types** | **Critical Steps**: Convert SAMA Codes (e.g., 03) to internal Bank Transaction Codes before querying. |
