# SAMA-SAIB Integration Flow Design

This document details the request and acknowledgment (ACK) flows for the integration between SAMA and SAIB.

## 1. Network & Architecture Flow

This diagram illustrates where each component resides within the SAIB network infrastructure, specifically highlighting the position of **APIC Internal** in the Trusted Zone.

```mermaid
graph TD
    subgraph External_Network [External / SAMA Network]
        SAMA[SAMA SOAP Client]
    end

    subgraph SAIB_DMZ [SAIB DMZ - Security Zone]
        SJN[APIC SJN - Secure Junction]
    end

    subgraph SAIB_Internal [SAIB Internal - Trusted Zone]
        INT[APIC Internal]
        ESB[IBM ACE - ESB]
        MS[Microservice]
    end

    SAMA -- "SOAP (HTTPS)" --> SJN
    SJN -- "Forward (mTLS)" --> INT
    INT -- "Route" --> ESB
    ESB -- "JSON (REST)" --> MS
    
    %% Response path
    MS -.-> ESB
    ESB -.-> INT
    INT -.-> SJN
    SJN -.-> SAMA

    style External_Network fill:#fff,stroke:#333
    style SAIB_DMZ fill:#fff4dd,stroke:#d4a017,stroke-width:2px
    style SAIB_Internal fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style INT fill:#bbdefb,stroke:#01579b
```

## 2. Sequence Diagram (Request Flow)

```mermaid
sequenceDiagram
    participant SAMA as SAMA (SOAP)
    box "SAIB API Gateway" #f9f9f9
        participant APICSJN as APIC SJN
        participant APICInt as APIC Internal
    end
    participant ESB as IBM ACE (ESB)
    participant MS as Microservice (JSON)

    Note over SAMA, APICSJN: Protocol: SOAP over HTTP
    SAMA->>APICSJN: SOAP Request
    APICSJN->>APICInt: Route to Internal Gateway
    
    Note over APICInt, ESB: Internal API Exchange
    APICInt->>ESB: SOAP Request
    
    Note over ESB: Transformation: SOAP to JSON<br/>Security Validation<br/>Routing Logic
    
    ESB->>MS: JSON REST Request
    
    Note over MS: Business Logic Execution
    
    MS-->>ESB: JSON Response
    
    Note over ESB: Transformation: JSON to SOAP
    
    ESB-->>APICInt: SOAP Response
    APICInt-->>APICSJN: Forward Response
    APICSJN-->>SAMA: SOAP Response
```

## 3. Acknowledgment (ACK) Flow: Transaction / Doc Inquiry

The ACK flow specifically handles the asynchronous or synchronous acknowledgment requirements for services like Transaction Search and Document Inquiry.

```mermaid
sequenceDiagram
    participant SAMA as SAMA (SOAP)
    participant APIC as SAIB APIC (SJN/Int)
    participant ESB as IBM ACE (ESB)
    participant MS as Microservice (JSON)
    participant DB as Backend Systems / DB

    Note over SAMA, MS: Initial Request Path (as seen above)
    
    MS->>DB: Query / Process Data
    DB-->>MS: Data Results
    
    Note right of MS: Check ACK Strategy
    
    alt Synchronous Response (Standard)
        MS-->>ESB: JSON Payload (Success/No Data)
        ESB-->>APIC: SOAP Response
        APIC-->>SAMA: 200 OK / SOAP Body
    else Async Acknowledgement (ACK)
        MS-->>ESB: JSON 202 Accepted (Correlation ID)
        ESB-->>APIC: SOAP Accepted
        APIC-->>SAMA: Transaction Receipt / ACK
        
        Note over SAMA, DB: Background Processing continues...
        
        DB-->>MS: Detailed Data Ready
        MS->>ESB: Trigger Callback / Update Status
        ESB->>APIC: Push Result (if applicable)
        APIC->>SAMA: Notification / Callback (Optional)
    end
```

## 4. Component Responsibilities & Locations

| Component | Location | Responsibility |
| :--- | :--- | :--- |
| **SAMA** | External | Initiates SOAP requests. |
| **APIC SJN** | **DMZ** | Secure Junction; handles perimeter security and authentication. |
| **APIC Internal** | **Internal Zone** | Central traffic management and routing within the trusted network. |
| **IBM ACE (ESB)** | **Internal Zone** | Protocol conversion (SOAP ↔ JSON) and specialized routing. |
| **Microservice** | **Internal Zone** | Core business logic and data processing. |
