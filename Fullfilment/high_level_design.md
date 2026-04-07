# Tanfeeth Fulfillment — Received Amount Service

## High-Level Design (HLD)

**Version:** 1.0  
**Date:** 2026-03-30  
**Status:** Extracted from production IIB codebase  
**System:** ESB Integration Layer (IBM IIB)  
**Regulatory Body:** SAMA (Saudi Arabian Monetary Authority)jdbc

---

## 1. Service Overview

The **Received Amount Service** is a regulatory compliance integration that monitors credit transactions posted to customer accounts subject to SAMA-imposed Tanfeeth holds. When cumulative deposits meet defined thresholds, the system notifies SAMA and processes the resulting approval/rejection cycle.

### 1.1 Business Objective

```
When SAMA issues a Tanfeeth enforcement order:
  1. Bank places a HOLD on the customer's account
  2. Customer can still RECEIVE credits (salary, transfers, etc.)
  3. Bank must TRACK every credit to the held account
  4. Bank must NOTIFY SAMA when thresholds are met
  5. K2 portal REVIEWS and approves/rejects each credit
  6. Bank REPORTS final approved amounts back to SAMA
```

### 1.2 Service Boundaries

```mermaid
graph TB
    subgraph External Systems
        CBS["Core Banking System<br/>(Credit Transactions)"]
        K2["K2 Portal<br/>(SAMA Adapter)"]
        SAMA["SAMA Backend<br/>(Regulatory Authority)"]
    end

    subgraph "ESB — Received Amount Service"
        App1["TanfeethFulfillment<br/>ReceivedAmountApp"]
        ExecSvc["TanfeethExecution<br/>ProcessService"]
    end

    subgraph Persistence
        DB[("Tanfeeth DB<br/>TANFEETHEXEC")]
    end

    CBS -->|MQ/DFDL| App1
    App1 -->|SOAP| K2
    K2 -->|SOAP| ExecSvc
    ExecSvc -->|MQ/XMLNSC| App1
    App1 -->|SOAP| SAMA
    App1 & ExecSvc <-->|ODBC| DB
```

---

## 2. System Context

### 2.1 Actors & Interfaces

| Actor | Interface | Protocol | Direction | Purpose |
|-------|-----------|----------|-----------|---------|
| **Core Banking System** | MQ Queue `MIKE.IIB.TanfeethFulfillRecievedAmount` | MQ → DFDL | Inbound | Credit transaction notifications |
| **K2 / SAMA Adapter** | `K2IntService.svc` | SOAP/HTTP | Outbound | Send received amount notification |
| **K2 / SAMA Adapter** | `TanfeethExecutionProcessService` | SOAP/HTTP | Inbound | Receive approval/rejection |
| **SAMA Backend** | `FIFFResrvdAmtCallBack` | SOAP/HTTP | Outbound | Report final approved amounts |
| **Tanfeeth Database** | `SAIBAPP` ODBC | SQL/ODBC | Bidirectional | Persistence & lookups |

### 2.2 Application Components

| Component | IIB Type | Description |
|-----------|----------|-------------|
| `TanfeethFulfillmentReceivedAmountApp` | Application | Main app container — 2 message flows |
| `TanfeethFulfillmentReceivedAmountLib` | Shared Library | DFDL model (`TransactionNotification`) |
| `TanfeethExecutionProcessService` | Service | SOAP service — 5 operations (shared with other Tanfeeth flows) |
| `TanfeethExecutionProcessLib` | Shared Library | WSDL/XSD for the execution process service |
| `TanfeethFulfillmentRecievedAmountCallbackLib` | Shared Library | WSDL/XSD for SAMA callback |
| `TanfeethICSLib` | Shared Library | ICS copybooks, XSD models, currency data |

---

## 3. End-to-End Flow Architecture

### 3.1 Phase Diagram

```mermaid
graph LR
    P1["Phase 1<br/>Deposit<br/>Ingestion"]
    P2["Phase 2<br/>Threshold<br/>Evaluation"]
    P3["Phase 3<br/>K2/SAMA<br/>Notification"]
    P4["Phase 4<br/>K2 Approval<br/>Processing"]
    P5["Phase 5<br/>SAMA<br/>Callback"]

    P1 --> P2 --> P3 --> P4 --> P5

    style P1 fill:#2196F3,color:#fff
    style P2 fill:#FF9800,color:#fff
    style P3 fill:#4CAF50,color:#fff
    style P4 fill:#9C27B0,color:#fff
    style P5 fill:#F44336,color:#fff
```

### 3.2 Complete Sequence

```mermaid
sequenceDiagram
    autonumber
    participant CBS as Core Banking
    participant MQ1 as MQ Input Queue
    participant F1 as Flow 1: ProcessingData
    participant DB as Tanfeeth DB
    participant K2 as K2 SOAP Service
    participant F1R as Flow 1: LoggingResponse
    participant ExecSvc as ExecutionProcessService
    participant MQ2 as MQ Callback Queue
    participant F2 as Flow 2: Callback
    participant SAMA as SAMA Backend
    rect rgb(33, 150, 243, 0.1)
        Note over CBS,DB: Phase 1 — Deposit Ingestion
        CBS->>MQ1: Credit Transaction (DFDL)
        MQ1->>F1: Consume message
        F1->>DB: getAccountHoldByAccountNumber()
        DB-->>F1: Hold records (or empty)
        alt No holds found
            F1->>F1: Log "No hold found", RETURN FALSE
        else Hold exists
            F1->>DB: getCurrencyByCurrencyCode()
            DB-->>F1: Currency exponent
            F1->>F1: Calculate depositAmount = rawAmt / 10^exponent
            F1->>DB: INSERT FULFILLMENT_DEPOSIT_AMOUNTS (STATUS='0')
            F1->>DB: COMMIT
        end
    end
    rect rgb(255, 152, 0, 0.1)
        Note over F1,DB: Phase 2 — Threshold Evaluation
        F1->>F1: Condition 1: deposit + balance ≥ holdAmount?
        alt Condition 1 met
            F1->>F1: K2=true, depositRecords=false
        else Condition 1 not met
            F1->>DB: getSUMDepositAMT()
            DB-->>F1: Sum of pending deposits
            F1->>F1: Condition 2: deposit + sum ≥ holdAmount?
            alt Condition 2 met
                F1->>F1: K2=true, depositRecords=true
            else Condition 2 not met
                F1->>F1: Condition 3: deposit + sum ≥ SamaReportingThreshold?
                alt Condition 3 met
                    F1->>F1: K2=true, depositRecords=true
                else No condition met
                    F1->>DB: UpdateStatus(eventId, '1')
                    F1->>F1: RETURN FALSE (no output)
                end
            end
        end
    end
    rect rgb(76, 175, 80, 0.1)
        Note over F1,K2: Phase 3 — K2/SAMA Notification
        F1->>DB: getCustomerByAccountNumber()
        DB-->>F1: IBAN, IDVALUE, DOCUMENT_TYPE, CUSTOMER_NAME
        F1->>DB: UPDATE deposit with SRN, IBAN, hold details
        F1->>F1: Build SOAP CreateTanFulRecAmoRequest
        opt depositRecords = true
            F1->>DB: getTotalDepositAMT()
            DB-->>F1: All pending deposit records
            F1->>F1: Add historical credits to CreditList
        end
        F1->>K2: SOAP Request (via K2Call subflow)
        K2-->>F1R: SOAP Response (K2RefNum)
        F1R->>DB: UpdateStatus(eventId, '2')
        F1R->>DB: INSERT K2_EXECUTION_INFO
    end
    rect rgb(156, 39, 176, 0.1)
        Note over K2,DB: Phase 4 — K2 Approval Processing
        K2->>ExecSvc: FulfillmentRecievedAmountApproval (SOAP)
        ExecSvc->>ExecSvc: Check Submission Window (BR-02, BR-03)
        alt Out of window (Window: 21:00 - 06:00)
            ExecSvc-->>K2: SOAP Fault (User Exception)
        else Valid window
            ExecSvc->>DB: getAccountHold (validate hold)
            loop For each Credit in CreditList
                alt Credit.Approved = true
                    ExecSvc->>DB: UPDATE STATUS='3' (approved)
                else Credit.Approved = false
                    ExecSvc->>DB: UPDATE STATUS='4' (rejected)
                end
            end
            ExecSvc->>DB: Update declaration tables
            ExecSvc-->>K2: SOAP Success Response
            ExecSvc->>MQ2: Route to FULFIL.IN queue
        end
    end
    rect rgb(244, 67, 54, 0.1)
        Note over MQ2,SAMA: Phase 5 — SAMA Callback
        MQ2->>F2: Consume callback message
        F2->>DB: Get account & hold details
        F2->>F2: Build FIFFResrvdAmtCallBack SOAP
        F2->>DB: INSERT FULFILLMENT_RECIEVED_AMOUNT_CB
        F2->>SAMA: SOAP Request
        alt Success
            SAMA-->>F2: Success Response
            F2->>DB: UPDATE CB status = 'SUCCESS'
        else Failure
            SAMA-->>F2: Fault/Error
            F2->>DB: UPDATE CB status = 'FAILED'
        end
    end
```

---

## 4. Component Architecture

### 4.1 Flow 1 — Received Amount Processing

```mermaid
graph LR
    subgraph "TanfeethFulfillmentReceivedAmount.msgflow"
        MQIn["MQ Input<br/>DFDL Domain<br/>TransactionNotification"] --> TC["Try/Catch"]
        TC -->|try| PD["Compute:<br/>ProcessingData<br/>───────────<br/>• Parse DFDL<br/>• Hold lookup<br/>• Deposit calc<br/>• DB insert<br/>• Threshold eval<br/>• SOAP build"]
        PD -->|out| K2["K2Call Subflow<br/>───────────<br/>• Log URL<br/>• SOAP Request<br/>• CreateTanFul<br/>RecAmoRequest"]
        K2 -->|out| LR["Compute:<br/>LoggingResponse<br/>───────────<br/>• Log K2 response<br/>• Status → '2'<br/>• Insert K2 info"]
        PD -->|out1| Audit["AuditingImpl<br/>Subflow"]
        TC -->|catch| UDS["Compute:<br/>UpdateDBStatus<br/>───────────<br/>• Status → '1'"]
        UDS --> EW["ExceptionWrapper<br/>Subflow"]
        MQIn -->|failure| UDS
        MQIn -->|catch| UDS
    end
```

**Key Design Decisions:**

- DFDL parsing for ICS mainframe-format messages
- Explicit `COMMIT` after deposit insert (before threshold evaluation)
- Three-condition threshold evaluation in priority order
- K2Call as a reusable subflow with configurable endpoint URL (UDP)

### 4.2 Flow 2 — SAMA Callback

```mermaid
graph LR
    subgraph "TanfeethFulfillmentRecievedAmountCallback.msgflow"
        MQIn2["MQ Input<br/>XMLNSC Domain<br/>K2 Approval Result"] --> PP["TaskExecution<br/>PreProcessing"]
        PP -->|out| CR["Compute:<br/>CreateRequest<br/>───────────<br/>• Build SAMA<br/>SOAP header<br/>• Account & hold<br/>lookup<br/>• Cap at hold amt<br/>• Insert CB record"]
        CR -->|out| SOAP["SOAP Request<br/>───────────<br/>• FIFFResrvdAmt<br/>CallBack<br/>• To SAMA endpoint"]
        CR -->|out1| Audit2["AuditingImpl"]
        SOAP -->|success| UCS["Compute:<br/>UpdateCallback<br/>Status<br/>───────────<br/>• Status=SUCCESS"]
        SOAP -->|fault| UFS["Compute:<br/>UpdateFailure<br/>Status<br/>───────────<br/>• Status=FAILED"]
        SOAP -->|failure| UFS
        MQIn2 -->|failure| EW2["ExceptionWrapper"]
        MQIn2 -->|catch| EW2
    end
```

### 4.3 ExecutionProcessService — Fulfillment Operation

```mermaid
graph LR
    subgraph "TanfeethExecutionProcessService.msgflow (Fulfillment path)"
        SI["SOAP Input<br/>5 Operations"] --> SPP["SOAP<br/>PreProcessing"]
        SPP --> RTL["Route To Label"]
        RTL -->|FulfillmentRecieved<br/>AmountApproval| FS["Fulfillment<br/>Subflow<br/>───────────<br/>• Validate hold<br/>• Loop credits<br/>• Status 3/4<br/>• Update decl"]
        FS --> SPost["SOAP<br/>PostProcessing"]
        SPost --> FO["Flow Order"]
        FO -->|1st| SR["SOAP Reply"]
        FO -->|2nd| Route{"Route<br/>RequestType"}
        Route -->|Fulfillment| PF["PrepareRequest<br/>ForFulfillment<br/>───────────<br/>• Build MQ msg<br/>• Set queue name"]
        PF --> MQH["MQ Header"] --> MQOut["MQ Output"]
    end
```

---

## 5. Data Architecture

### 5.1 Database Tables

```mermaid
erDiagram
    FULFILLMENT_DEPOSIT_AMOUNTS {
        NUMBER EVENTID PK "Sequence: FULFILLMENT_DEPOSIT_SEQ"
        VARCHAR TRANSACTIONID "MQ MsgId"
        VARCHAR ACCOUNT_NUMBER
        VARCHAR CUSTOMER_NUMBER
        DECIMAL DEPOSIT_AMOUNT
        DECIMAL REMAINING_AMOUNT
        VARCHAR DEPOSIT_CURRENCY
        VARCHAR ACCOUNT_CURRENCY
        VARCHAR STATUS "0=Pending, 1=Processed, 2=SentToK2, 3=Approved, 4=Rejected"
        VARCHAR TRANSACTION_DIRECTION
        VARCHAR SRN "SAMA Reference Number"
        VARCHAR IBAN
        DECIMAL HOLD_AMOUNT
        VARCHAR HOLD_CURRENCY
        DECIMAL HOLD_EXCHANGE_RATE
        VARCHAR CRITERIA_ID
        TIMESTAMP CREATE_DATETIME
        VARCHAR CREATE_USER
        TIMESTAMP UPDATE_DATETIME
        VARCHAR UPDATE_USER
    }

    HOLD_FULFILLMENT_STATUS {
        NUMBER ID PK "Sequence: HOLD_FULFILLMENT_STATUS_SEQ"
        VARCHAR ACCOUNT_NUMBER
        VARCHAR CRITERIA_ID FK
        DECIMAL HOLD_AMOUNT
        DECIMAL ACCUMULATED_AMOUNT
        VARCHAR STATUS "1=Partially Fulfilled, 2=Fully Fulfilled"
    }

    ACCOUNT_HOLD {
        VARCHAR ACCOUNT_NUMBER
        DECIMAL HOLD_AMOUNT
        VARCHAR HOLD_CURRENCY
        DECIMAL AMOUNT_IN_ACCOUNT_CURRENCY
        DECIMAL APPLIED_RATE
        VARCHAR CRITERIA_ID FK
    }

    CRITERIA_DETAILS {
        VARCHAR CRITERIA_ID PK
        VARCHAR SRN "SAMA Reference Number"
    }

    K2_EXECUTION_INFO {
        NUMBER ID PK
        VARCHAR CRITERIA_ID FK
        VARCHAR K2_REQUEST_TYPE
        VARCHAR K2_REFERENCE_NUM
        VARCHAR RESPONSE_TYPE
        VARCHAR RESPONSE_DESC
        VARCHAR MESSAGE_ID
        VARCHAR CORREL_ID
        VARCHAR IIB_STATUS
        VARCHAR TRANSACTION_ID
        TIMESTAMP CREATE_DATETIME
    }

    EXE_ACCOUNTS {
        VARCHAR NUMBER
        VARCHAR IBAN
        VARCHAR CRITERIA_ID FK
    }

    EXE_CUSTOMER {
        VARCHAR CRITERIA_ID FK
        VARCHAR CUSTOMER_NAME
        VARCHAR IDVALUE
        VARCHAR DOCUMENT_TYPE
    }

    FULFILLMENT_RECIEVED_AMOUNT_CB {
        NUMBER FULFILLMENT_CB_ID PK "Sequence"
        VARCHAR TRANSACTION_ID
        VARCHAR STATUS "SENT / SUCCESS / FAILED"
        VARCHAR STATUS_DESC
        VARCHAR CALLBACK_STATUS
        TIMESTAMP CALLBACKREQUESTTIME
        TIMESTAMP CALLBACKRESPONSETIME
    }

    DECLARATION {
        NUMBER ID PK
        DECIMAL TOTAL_AMOUNT_IN_REQUEST_CURRENCY
        DECIMAL TOTAL_AMOUNT_IN_SAR
        DECIMAL TOTAL_AMOUNT
        VARCHAR CRITERIA_ID FK
    }

    ACCOUNT_HOLD ||--o{ CRITERIA_DETAILS : "joined via CRITERIA_ID"
    EXE_ACCOUNTS ||--o{ EXE_CUSTOMER : "joined via CRITERIA_ID"
    FULFILLMENT_DEPOSIT_AMOUNTS }o--|| ACCOUNT_HOLD : "by ACCOUNT_NUMBER"
    HOLD_FULFILLMENT_STATUS }o--|| ACCOUNT_HOLD : "tracks fulfillment"
```

### 5.2 Deposit Status Lifecycle

```mermaid
stateDiagram-v2
    [*] --> 0
    
    0 --> 1 : Processed (Threshold not met)
    0 --> 2 : Threshold met, Sent to K2

    1 --> 3 : Sent to K2 -> Approved
    1 --> 4 : Sent to K2 -> Rejected
    2 --> 3 : Approved by K2
    2 --> 4 : Rejected by K2

    1 --> [*]
    3 --> [*]
    4 --> [*]

    state "STATUS = '0' (Just Inserted)" as 0
    state "STATUS = '1' (Pending in Quota)" as 1
    state "STATUS = '2' (Sent to K2)" as 2
    state "STATUS = '3' (Approved)" as 3
    state "STATUS = '4' (Rejected)" as 4
    
    note right of 1 : Awaits cumulative threshold.<br/>Summations and fetch queries<br/>target this status.
```

### 5.3 Key Database Queries

| Query | Purpose | Called By |
|-------|---------|----------|
| `getAccountHoldByAccountNumber` | Get active holds for an account (joined with CRITERIA_DETAILS for SRN) | Flow 1: ProcessingData |
| `getCurrencyByCurrencyCode` | Get currency exponent for amount calculation | Flow 1: ProcessingData |
| `getSUMDepositAMT` | Sum pending deposits for cumulative threshold check | Flow 1: ProcessingData |
| `getTotalDepositAMT` | Get all pending deposit records for CreditList | Flow 1: ProcessingData |
| `getCustomerByAccountNumber` | Get customer IBAN, ID, name for SAMA notification | Flow 1: ProcessingData |
| `UpdateStatus` | Update deposit record status | Flow 1: ProcessingData, LoggingResponse |
| `getAccountHoldByAccountNumberCriteriaID` | Validate hold for specific criteria | ExecSvc: Fulfillment, Callback |
| `SelectDecleration` / `SelectDeclerationDetails` | Get existing declaration for update | ExecSvc: Fulfillment |

---

## 6. Integration Contracts

### 6.1 Inbound — Credit Transaction (MQ/DFDL)

| Property | Value |
|----------|-------|
| **Queue** | `MIKE.IIB.TanfeethFulfillRecievedAmount` |
| **Domain** | DFDL |
| **Message Type** | `TransactionNotification` |
| **Model** | `TanfeethFulfillmentReceivedAmountLib` |

**Key Fields:**

| Field | Description |
|-------|-------------|
| `MsgReqDat.CusAcc` | Customer account number |
| `MsgReqDat.CusNum` | Customer CIF number |
| `MsgReqDat.CusAccCcy` | Account currency (ISO 4217) |
| `MsgReqDat.TrxAmt` | Raw transaction amount (needs exponent conversion) |
| `MsgReqDat.Trxdir` | Transaction direction ("C" for credit) |

### 6.2 Outbound to K2 — Received Amount Notification (SOAP)

| Property | Value |
|----------|-------|
| **Operation** | `CreateTanFulRecAmoRequest` |
| **WSDL** | `K2IntService.wsdl` |
| **Namespace** | `http://tempuri.org/` |
| **Endpoint** | Configurable via UDP `endPointURL` |

```xml
<soapenv:Envelope>
  <soapenv:Body>
    <tem:CreateTanFulRecAmoRequest>
      <tem:Request>
        <saib:AccountCurrency/>
        <saib:AccountNumber/>
        <saib:CreditList>
          <ns:Credit>                       <!-- Current + historical -->
            <ns:AmountInAccountCurrency/>
            <ns:DepositeCurrency/>
            <ns:EventId/>                   <!-- Only for historical -->
            <ns:ExchnageRate/>
            <ns:PostingDate/>
            <ns:SourceAmount/>
          </ns:Credit>
        </saib:CreditList>
        <saib:Currency/>
        <saib:CustomerID/>
        <saib:CustomerName/>
        <saib:ESBReferenceNumber/>          <!-- CRITERIA_ID -->
        <saib:ExchageRate/>
        <saib:HoldAmount/>
        <saib:HoldAmountCurrency/>
        <saib:IBAN/>
        <saib:IDType/>
        <saib:ReportingDateTime/>
        <saib:SamaRefNum/>                  <!-- SRN -->
      </tem:Request>
    </tem:CreateTanFulRecAmoRequest>
  </soapenv:Body>
</soapenv:Envelope>
```

### 6.3 Inbound from K2 — Approval Confirmation (SOAP)

| Property | Value |
|----------|-------|
| **Operation** | `FulfillmentRecievedAmountApproval` |
| **WSDL** | `TanfeethExecutionProcessService.wsdl` |
| **Namespace** | `http://www.saib.com.sa/integration/TanfeethExecutionProcess/1.0` |
| **URL** | `/integration/TanfeethExecutionProcess/1.0` |

```xml
<FulfillmentRecievedAmountApproval>
    <ESBReferanceNumber>CRIT-001</ESBReferanceNumber>
    <AccountNumber>0012345678</AccountNumber>
    <CreditList>
        <Credit>
            <AmountInAccountCurrency>5000.00</AmountInAccountCurrency>
            <DepositeCurrency>SAR</DepositeCurrency>
            <EventId>12345</EventId>
            <ExchnageRate>1.0</ExchnageRate>
            <SourceAmount>5000.00</SourceAmount>
            <Approved>true</Approved>           <!-- K2 decision -->
        </Credit>
    </CreditList>
</FulfillmentRecievedAmountApproval>
```

### 6.4 Outbound to SAMA — Reserved Amount Callback (SOAP)

| Property | Value |
|----------|-------|
| **Operation** | `FIFFResrvdAmtCallBack` |
| **WSDL** | `FIFFResrvdAmtCallBack.wsdl` |
| **Namespace** | `http://www.sama.bea.sa/execution/services/FIFFResrvdAmtCallBack` |
| **Endpoint** | Configurable via UDP `SAMAEndpoint` |

### 6.5 Internal — MQ Callback Queue

| Property | Value |
|----------|-------|
| **Queue** | `IIB.TNFTH.EXECUTION.K2.PRCS.FULFIL.IN` |
| **Domain** | XMLNSC |
| **Direction** | ExecSvc → ReceivedAmountApp Callback Flow |

---

## 7. Threshold Decision Logic

### 7.1 Decision Tree

```mermaid
flowchart TD
    Start["New Credit Deposit"] --> HasHold{"Account has<br/>active holds?"}
    HasHold -->|No| End1["Log + Terminate<br/>No action"]
    HasHold -->|Yes| CalcAmt["Calculate deposit amount<br/>Insert record (STATUS '0')"]
    CalcAmt --> HoldLoop["Loop over all active holds<br/>(Exclude fully fulfilled)"]
    HoldLoop --> Consume["ConsumeRemainingAmount<br/>Update Hold & Deposit Remaining Amts"]
    Consume --> C1{"Condition 1<br/>Single deposit ≥ hold?"}
    
    C1 -->|Yes| Notify1["Notify K2/SAMA<br/>CreditList = current"]
    C1 -->|No| SumQuery["Get accumulated credit for hold"]
    SumQuery --> C2{"Condition 2<br/>Accumulated ≥ hold?"}
    
    C2 -->|Yes| Notify2["Notify K2/SAMA<br/>CreditList = current + pending"]
    C2 -->|No| C3{"Condition 3<br/>Accumulated ≥ SamaThreshold?"}
    
    C3 -->|Yes| Notify3["Notify K2/SAMA<br/>CreditList = current + pending"]
    C3 -->|No| NextHold["Next hold in loop"]
    NextHold --> End2["Update STATUS → '1'<br/>if no holds triggered notification"]

    style Notify1 fill:#4CAF50,color:#fff
    style Notify2 fill:#4CAF50,color:#fff
    style Notify3 fill:#FF9800,color:#fff
    style End2 fill:#9E9E9E,color:#fff
    style End1 fill:#9E9E9E,color:#fff
```

### 7.2 CreditList Composition Rules

| Trigger | CreditList Contents | `depositRecords` Flag |
|---------|--------------------|-----------------------|
| Condition 1 (single deposit + balance ≥ hold) | Current deposit only | `false` |
| Condition 2 (cumulative ≥ hold) | Current deposit + all pending from DB | `true` |
| Condition 3 (cumulative ≥ SamaReportingThreshold UDP) | Current deposit + all pending from DB | `true` |

---

## 8. Configuration & Deployment

### 8.1 User-Defined Properties (UDPs)

| UDP | Default | Scope | Description |
|-----|---------|-------|-------------|
| `LoggerName` | `FulfillmentRecievedAmount` | Flow 1 | Log4j logger name |
| `SamaReportingThreshold` | `5000.00` | Flow 1 | Dynamic threshold for sending deposit reports to SAMA |
| `endPointURL` | `Http://K2` | K2Call subflow | K2 SOAP endpoint URL |
| `SAMAEndpoint` | `http://samaendpoint` | Callback flow | SAMA SOAP endpoint URL |
| `ISCALLBACK_REQUIRED` | `('E1020024','E1020025')` | ExecSvc | SAMA status codes requiring callback |
| `SamaSubmissionStartTime`| `'21:00:00'` | ExecSvc | Start of SAMA allowed window (BR-02) |
| `SamaSubmissionEndTime` | `'06:00:00'` | ExecSvc | End of SAMA allowed window (BR-02) |

### 8.2 Data Sources

| Name | Used By | Purpose |
|------|---------|---------|
| `SAIBAPP` | All compute nodes | Main Tanfeeth database connection |
| `SOSDB` | Currency lookup | Currency list / exponent reference data |

### 8.3 Queue Configuration

| Queue Name | Direction | Domain | Consumer |
|-----------|-----------|--------|----------|
| `MIKE.IIB.TanfeethFulfillRecievedAmount` | Input | DFDL | Flow 1 |
| `IIB.TNFTH.EXECUTION.K2.PRCS.FULFIL.IN` | Internal | XMLNSC | Callback Flow |

### 8.4 Logging & Monitoring

| Category | Configuration |
|----------|---------------|
| **Framework** | Log4j v1 (DailyRollingFileAppender) |
| **Log File** | `/home/S803SOSA/soaApps/IIB/TanfeethFulfillmentRecievedAmountApp/logs/` |
| **Max Size** | 10 MB per file |
| **Max Backup** | 50 files |
| **BAM** | Business Activity Monitoring via AuditingImpl subflow |
| **Correlation** | MQ CorrelId + MsgId (hex-decoded) |

---

## 9. Error Handling Strategy

### 9.1 Error Flow

```mermaid
graph TD
    subgraph "Flow 1 Error Handling"
        MQIn["MQ Input"] -->|failure/catch| EH1["Update DB Status<br/>(STATUS='1')"]
        EH1 --> EW1["ExceptionWrapper<br/>Subflow"]
        
        TC["Try/Catch"] -->|catch| EH1
        
        DB_ERR["SQL Error<br/>in ProcessingData"] --> Handler["EXIT HANDLER<br/>SQLSTATE LIKE '%'"]
        Handler --> Log["Log error<br/>(TFA.ESB.C001)"]
        Log --> Resignal["RESIGNAL<br/>(re-throw)"]
    end

    subgraph "Callback Error Handling"
        SOAP_OK["SOAP Success"] --> UCS["Update CB<br/>STATUS=SUCCESS"]
        SOAP_FAIL["SOAP Fault/Failure"] --> UFS["Update CB<br/>STATUS=FAILED"]
    end
```

### 9.2 Error Codes

| Code | Scope | Description |
|------|-------|-------------|
| `TFA.ESB.C001` | All DB operations | Generic database error (INSERT, UPDATE, SELECT failures) |
| `BIP 2951` | User exception | Custom IIB exception with error message |

---

## 10. Security Considerations

| Aspect | Current State | Recommendation |
|--------|--------------|----------------|
| **Transport** | HTTP (no TLS enforcement) | Enforce HTTPS for all SOAP endpoints |
| **PII in Logs** | Full request body logged at INFO | Mask account numbers, customer names, IBANs |
| **Authentication** | None visible in code | Add WS-Security or mutual TLS |
| **Data at Rest** | Standard DB storage | Encrypt PII columns in TANFEETHEXEC schema |

---

## 11. Non-Functional Characteristics

| Attribute | Target | Notes |
|-----------|--------|-------|
| **Throughput** | Process each message within 2 seconds | Under normal DB load |
| **DB Query Latency** | Each query < 500ms | Hold lookup, sum, customer info |
| **Availability** | Follow IIB broker HA configuration | No custom HA in flow |
| **Idempotency** | Not implemented | Risk of duplicate deposit records on retry |
| **Audit Trail** | Every deposit tracked in DB | `FULFILLMENT_DEPOSIT_AMOUNTS` with timestamps |
| **Transaction Integrity** | Manual COMMIT in code | Risk of partial commits |

---

## 12. Phase 4: K2 Confirmation Processing

This process activates when the K2 portal returns approval payloads back through `TanfeethExecutionProcessService`.

### 12.1 Execution Routing
1. The K2 portal invokes the `FulfillmentRecievedAmountApproval` process via ESB.
2. The core router maps the incoming approval to the `IIB.TNFTH.EXECUTION.K2.PRCS.FULFIL.IN` internal queue.
3. The original declaration and deposit lists are re-queried, updating execution logic.
4. Database updates trigger status shifts to `STATUS = '3'` (Approved) or `STATUS = '4'` (Rejected).

---

## 13. Phase 5: SAMA Callback Integration

This phase manages the outbound notification to the SAMA endpoint confirming that the held funds have successfully been reserved for fulfillment.

### 13.1 Callback Logic
1. Messages land on `IIB.TNFTH.EXECUTION.K2.PRCS.FULFIL.IN`.
2. The Integration App formats the payload via DFDL/XML schema parameters towards `FIFFResrvdAmtCallBack`.
3. If an HTTP connection issue or 500 error takes place with SAMA, the system logs the fault and transitions into the Fallback sequence.
4. The response status is logged comprehensively into `TANFEETHEXEC.K2_EXECUTION_INFO` to leave a robust audit trail.

---

## 14. Known Limitations & Future Improvements

| # | Limitation | Impact | Priority |
|---|------------|--------|----------|
| 1 | Only first hold record processed per account | Multiple holds ignored | [RESOLVED] Fixed in Refactored.esql via multi-hold loop / progressive consumption |
| 4 | No idempotency / deduplication | Duplicate deposits on MQ retry | High |
| 6 | PII logged in full | SAMA data protection non-compliance | Medium |
| 7 | Single error code for all DB failures | Hard to diagnose in production | Low |
| 8 | SOAP field typos (`DepositeCurrency`, `ExchnageRate`) | Can't fix without SAMA schema change | Low |
| 9 | ~~Phases 4-5 not documented in spec~~ | [RESOLVED] Phase 4-5 documented in Sections 12-13 | N/A |
| 10 | No monitoring metrics | Limited operational visibility | Medium |

---

## 15. SAMA Submission Gatekeeper (Time/Calendar Validation)

This logic enforces SAMA regulatory windows for manual approvals received from the K2 portal.

### 15.1 Business Rules
*   **BR-02 (Time Window)**: Manual approvals from K2 are only permitted when `CURRENT_TIME` is between `SamaSubmissionStartTime` and `SamaSubmissionEndTime` (default 21:00 to 06:00).
*   **BR-03 (Calendar Restriction)**: Manual approvals are blocked on:
    *   **Fridays** (IIB `DAYOFWEEK` = 6)
    *   **Saturdays** (IIB `DAYOFWEEK` = 7)
    *   **Official Holidays** (Determined via `checkIsHoliday` procedure in `Utils.esql`)

### 15.2 Implementation
*   **Entry Point**: `FulfillmentRecievedAmount_processRequest.esql :: Main()`
*   **Error Handling**: If a check fails, a `USER EXCEPTION` is thrown with a descriptive message, which triggers a SOAP Fault back to the K2 user.
*   **Configurability**: The window start and end times are configurable via **User-Defined Properties (UDPs)** without needing a code redeploy.

