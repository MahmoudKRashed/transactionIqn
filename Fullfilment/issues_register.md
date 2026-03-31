# Tanfeeth Fulfillment — Consolidated Issues Register

**Date:** 2026-03-30  
**Source:** Code reviews of `TanfeethFulfillmentReceivedAmountApp` + `TanfeethExecutionProcessService`  
**Total Issues:** 25

---

## Summary Dashboard

| Category | 🔴 Critical | 🟠 High | 🟡 Medium | 🔵 Low | Total |
|----------|:-----------:|:-------:|:---------:|:------:|:-----:|
| **Logic Bugs** | 1 | 2 | 0 | 0 | **3** |
| **Data Integrity** | 0 | 2 | 1 | 0 | **3** |
| **Error Handling** | 1 | 1 | 1 | 0 | **3** |
| **Security & Compliance** | 0 | 0 | 2 | 0 | **2** |
| **Architecture & Design** | 0 | 1 | 4 | 0 | **5** |
| **Code Quality** | 0 | 0 | 1 | 5 | **6** |
| **Spec & Documentation** | 1 | 0 | 3 | 1 | **5** |
| **Total** | **3** | **6** | **12** | **6** | **27** |

---

## 🔴 CRITICAL — Must Fix Before Production

These issues cause **incorrect behavior** or **regulatory risk**.

---

### C-02: EventId Always NULL for Historical Credits

| Attribute | Detail |
|-----------|--------|
| **Category** | Logic Bug |
| **App** | ReceivedAmountApp |
| **File** | [ProcessingData.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethFulfillmentReceivedAmountApp/gen/TanfeethFulfillmentReceivedAmount_ProcessingData.esql), Line 165 + [Utils.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethFulfillmentReceivedAmountApp/gen/Utils.esql), Line 32 |
| **Requirement** | FR-22 |
| **Impact** | SAMA receives credits without EventIds — K2 cannot map approvals back to deposits |

**Problem:** Two bugs compound:

1. `getTotalDepositAMT` doesn't SELECT `EVENTID`
2. Credit loop references `eventIdROW.records[1].EVENT_ID` (wrong field name, wrong record)

**Fix:**

```diff
-- Utils.esql — add EVENTID to SELECT:
- SELECT DEPOSIT_AMOUNT, DEPOSIT_CURRENCY, HOLD_EXCHANGE_RATE, TRANSACTIONID
+ SELECT EVENTID, DEPOSIT_AMOUNT, DEPOSIT_CURRENCY, HOLD_EXCHANGE_RATE, TRANSACTIONID

-- ProcessingData.esql Line 165 — reference loop record:
- SET creditRefLoop.ns:EventId = eventIdROW.records[1].EVENT_ID;
+ SET creditRefLoop.ns:EventId = CAST(record.EVENTID AS CHARACTER);
```

**Status:** ✅ Fixed in Phase 1

---

### C-03: No SQL Error Handlers in ExecSvc Fulfillment

| Attribute | Detail |
|-----------|--------|
| **Category** | Error Handling |
| **App** | ExecutionProcessService |
| **File** | [FulfillmentRecievedAmount_processRequest.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethExecutionProcessService/sa/com/saib/tanfeeth/fulfillment/FulfillmentRecievedAmount_processRequest.esql) |
| **Requirement** | NFR-04 |
| **Impact** | 8 unprotected DB calls — failures crash silently with no logging or error codes |

**Problem:** The entire `FulfillmentRecievedAmount_processRequest` module has zero `EXIT HANDLER FOR SQLSTATE` blocks. Any DB error (hold lookup, status update, declaration insert/update) produces an unhandled exception.

**Fix:** Wrap DB operations in `BEGIN...END` blocks with SQL error handlers, matching the pattern used in `ProcessingData.esql`.

**Status:** ✅ Fixed in Phase 1

---

### C-04: Undocumented Status Values '3' and '4'

| Attribute | Detail |
|-----------|--------|
| **Category** | Spec & Documentation |
| **App** | ExecutionProcessService |
| **File** | [FulfillmentRecievedAmount_processRequest.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethExecutionProcessService/sa/com/saib/tanfeeth/fulfillment/FulfillmentRecievedAmount_processRequest.esql), Lines 42 & 52 |
| **Requirement** | BR-06 |
| **Impact** | Spec only documents `'0'` and `'1'` — status `3`/`4` are unquoted integers to a VARCHAR column, semantics unknown to operations/support teams |

**Problem:** The status lifecycle is undocumented in the spec (`0→1`, `0→2→3/4`), which leads to confusion. Status `3` and `4` need to be explicitly documented.

**Fix:** Document the full lifecycle (`0→1`, `0→2→3/4`), quote status values (`'3'`, `'4'`) in the code.

**Status:** ✅ Fixed in Phase 1

---

## 🟠 HIGH — Should Fix Before Next Release

These issues cause **data quality problems** or **operational risk**.

---

### H-01: FLOAT Used for All Monetary Calculations

| Attribute | Detail |
|-----------|--------|
| **Category** | Data Integrity |
| **App** | Both |
| **Files** | ProcessingData.esql (Lines 75, 98, 100), FulfillmentRecievedAmount_processRequest.esql (Lines 31, 33, 34) |
| **Requirement** | Constraint C4, Improvement I3 |
| **Impact** | IEEE 754 floating-point cannot precisely represent many decimal values — rounding errors accumulate in financial calculations |

**Fix:** Replace all `FLOAT` declarations with `DECIMAL` for monetary variables: `depositAmt`, `totalValue`, `holdAccValue`, `TotalApprovedAccCur`, `TotalAmtReqCurr`, `TotalAmtSAR`, `TotalAmt`, `AppliedRate`.

**Status:** 🟡 Partially Fixed (ProcessingData.esql updated in Phase 2, ExecSvc pending)

---

### H-02: Unreachable THROW After RESIGNAL

| Attribute | Detail |
|-----------|--------|
| **Category** | Error Handling |
| **App** | ReceivedAmountApp |
| **Files** | ProcessingData.esql (Lines 87-88, 137-138), Callback_CreateRequest.esql (Lines 107-108), Callback_Updating_Callback_Status.esql (Line 28) |
| **Requirement** | NFR-04 |
| **Impact** | Dead code — custom BIP 2951 exception with descriptive message is never sent. Only the raw SQL exception propagates. |

**Fix:** Pick one strategy per handler — either `RESIGNAL` (re-throw original) or `THROW USER EXCEPTION` (custom message), not both.

**Status:** ❌ Discarded (Kept THROW as fallback per operational requirement)

---

### H-03: TotalApprovedAccCur Adds Declaration Total

| Attribute | Detail |
|-----------|--------|
| **Category** | Logic Bug |
| **App** | ExecutionProcessService |
| **File** | [FulfillmentRecievedAmount_processRequest.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethExecutionProcessService/sa/com/saib/tanfeeth/fulfillment/FulfillmentRecievedAmount_processRequest.esql), Line 61 |
| **Impact** | Potential double-counting — the value sent to SAMA callback includes both new and old declaration amounts |

**Problem:** `TotalApprovedAccCur = newlyApproved + existingDeclaration.TOTAL_AMOUNT`. While the downstream callback caps this at `HOLD_AMOUNT`, the semantic is misleading and could cause reporting errors.

**Status:** 🟡 Pending Comment-Out for Investigation (Phase 2)

---

### H-04: Only Last Approved EventID Stored

| Attribute | Detail |
|-----------|--------|
| **Category** | Logic Bug |
| **App** | ExecutionProcessService |
| **File** | [FulfillmentRecievedAmount_processRequest.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethExecutionProcessService/sa/com/saib/tanfeeth/fulfillment/FulfillmentRecievedAmount_processRequest.esql), Line 43 |
| **Impact** | If multiple credits approved, only last EventId is passed downstream — loss of traceability |

**Fix:**

```diff
- SET EnvVarRef.ApprovedList.EventID = creditRecord.EventId;
+ CREATE LASTCHILD OF EnvVarRef.ApprovedList NAME 'EventID' VALUE creditRecord.EventId;
```

---

### H-05: Partial Commit / Transaction Integrity

| Attribute | Detail |
|-----------|--------|
| **Category** | Data Integrity |
| **App** | ExecutionProcessService |
| **File** | [FulfillmentRecievedAmount_processRequest.esql](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethExecutionProcessService/sa/com/saib/tanfeeth/fulfillment/FulfillmentRecievedAmount_processRequest.esql), Lines 42-68 |
| **Impact** | STATUS updates committed at line 66, but `UpdateDeclration` / `UpdateDeclrationDtls` (lines 67-68) may fail after commit — leaving DB inconsistent |

**Fix:** Move all DB operations into a single transactional block, commit all at end, or use savepoints.

**Status:** ✅ Fixed in Phase 1

---

### H-06: SOAP Reply Sent Before MQ Put

| Attribute | Detail |
|-----------|--------|
| **Category** | Architecture & Design |
| **App** | ExecutionProcessService |
| **File** | [TanfeethExecutionProcessService.msgflow](file:///c:/Users/MahmoudKamalRashed/Downloads/Fullfilment_Repo/Fullfilment/IIB/TanfeethExecutionProcessService/sa/com/saib/tanfeeth/TanfeethExecutionProcessService.msgflow) |
| **Impact** | K2 gets success response, but if MQ put fails, callback never fires — SAMA is never notified |

**Problem:** FlowOrder sends SOAP Reply (first) then routes to MQ (second). If MQ put fails, K2 already has a success.

---

## 🟡 MEDIUM — Plan for Next Iteration

These are **design gaps** or **compliance risks** that don't cause immediate failure.

---

### M-01: Hardcoded 5,000 SAR Threshold

| Attribute | Detail |
|-----------|--------|
| **Category** | Architecture & Design |
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Line 118 |
| **Requirement** | Constraint C2, Improvement I2 |
| **Fix** | Externalize as UDP or DB config value |
| **Status** | ✅ Fixed (Externalized to `SamaReportingThreshold` UDP in Phase 2) |

---

### M-02: Hardcoded Exchange Rate (1.0)

| Attribute | Detail |
|-----------|--------|
| **Category** | Architecture & Design |
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Line 153 |
| **Requirement** | Constraint C3, Improvement I5 |
| **Fix** | Use actual exchange rate when deposit currency ≠ account currency |
| **Status** | ✅ Fixed in Phase 2 (Mapped to `HOLD_EXCHANGE_RATE`) |

---

### M-03: PII Logged in Full Request Body

| Attribute | Detail |
|-----------|--------|
| **Category** | Security & Compliance |
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Line 50 |
| **Requirement** | NFR-07 |
| **Fix** | Mask account numbers, customer names, national IDs before logging |
| **Status** | 🟡 Business Edge Case (PII Leakage risk accepted for now) |

---

### M-04: No Idempotency / Deduplication

| Attribute | Detail |
|-----------|--------|
| **Category** | Data Integrity |
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql |
| **Requirement** | NFR-05 |
| **Fix** | Check `TransactionId` uniqueness before INSERT, or use UNIQUE constraint on table |
| **Status** | 🟡 Business Edge Case (Duplicate MQ insertion risk accepted for now) |

---

### M-05: Missing Reversed-Criteria Check in Fulfillment Routing

| Attribute | Detail |
|-----------|--------|
| **Category** | Architecture & Design |
| **App** | ExecutionProcessService |
| **File** | PrepareRequestForFulfillment.esql |
| **Fix** | Add `REVERSE_CRITERIA` and `SAMA_STATUS` checks (same as `PrepareRequestForOrch`) |

---

### M-06: Reused Error Code TFA.ESB.C001

| Attribute | Detail |
|-----------|--------|
| **Category** | Error Handling |
| **App** | Both |
| **Files** | All error handlers |
| **Requirement** | Improvement I8 |
| **Fix** | Use distinct codes: `C001` (INSERT), `C002` (UPDATE), `C003` (SELECT), `C004` (callback) |
| **Status** | 🟡 Business Edge Case (Generic debugging accepted for now) |

---

### M-07: No Currency Exponent Null Check

| Attribute | Detail |
|-----------|--------|
| **Category** | Security & Compliance |
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Line 74-75 |
| **Fix** | Check `EXISTS(currencyExponent.records[])` before accessing `.EXPONENT` |
| **Status** | 🟡 Business Edge Case (Fatal crash on invalid currency code accepted) |

---

### M-08: Duplicate Procedure Definitions

| Attribute | Detail |
|-----------|--------|
| **Category** | Code Quality |
| **App** | Both |
| **Files** | Utils.esql (Line 8) vs FulfillmentRecievedAmount_processRequest.esql (Line 131) |
| **Fix** | Consolidate `getAccountHoldByAccountNumberCriteriaID` into shared library with explicit column list |
| **Status** | 🟡 Business Edge Case (Schema drift risk via duplicate SQL accepted) |

---

### M-09: Spec Missing Phases 4 & 5

| Attribute | Detail |
|-----------|--------|
| **Category** | Spec & Documentation |
| **Impact** | K2 confirmation handling and SAMA callback flow are entirely undocumented |
| **Fix** | Add §12 K2 Confirmation Processing and §13 SAMA Callback to spec |
| **Status** | ✅ Fixed (HLD updated with Phase 4/5 logic) |

---

### M-10: Undocumented Status '2' (Sent to K2)

| Attribute | Detail |
|-----------|--------|
| **Category** | Spec & Documentation |
| **App** | ReceivedAmountApp |
| **File** | LoggingResponseAndUpdatingStatus.esql, Line 21 |
| **Fix** | Add to BR-06 status definitions |
| **Status** | ✅ Fixed (Documented in HLD Section 5.2) |

---

### M-11: SAMA Submission Timing Compliance (New Rule)

| Attribute | Detail |
|-----------|--------|
| **Category** | Architecture & Design / Compliance |
| **App** | ExecutionProcessService |
| **File** | FulfillmentRecievedAmount_processRequest.esql |
| **Impact** | Risk of SAMA rejection if manual users approve outside of 21:00-06:00 window |
| **Fix** | Implemented UDP-based time window and calendar/holiday check in the approval module |
| **Status** | ✅ Fixed |

---

## 🔵 LOW — Address When Convenient

Cosmetic issues, dead code, minor inconsistencies.

---

### L-01: Inconsistent Log Bracket Style

| Attribute | Detail |
|-----------|--------|
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Lines 43-46 |
| **Detail** | Uses `{` opening but `]` closing: `'ICS CorrID : {'||...||']'` |

---

### L-02: Unused Variable J in Credit Loop

| Attribute | Detail |
|-----------|--------|
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Lines 156, 169 |
| **Detail** | Counter `J` is incremented but never read |

---

### L-03: Typo "CutomerInfoRef"

| Attribute | Detail |
|-----------|--------|
| **App** | ReceivedAmountApp |
| **File** | ProcessingData.esql, Line 125 |
| **Detail** | Should be `CustomerInfoRef` |

---

### L-04: Dead Code — Unused Procedures

| Attribute | Detail |
|-----------|--------|
| **App** | Both |
| **Detail** | `CopyMessageHeaders()` defined in every module but almost never called; `GetDepositByEventId()` in Utils.esql never called anywhere |

---

### L-05: SELECT * in Multiple Procedures

| Attribute | Detail |
|-----------|--------|
| **App** | ExecutionProcessService |
| **Files** | `getAccountHoldByAccountNumberCriteriaID` (Line 133), `SelectDecleration` (Line 113), `SelectDeclerationDetails` (Line 118) |
| **Detail** | Fragile — any table schema change can silently break the code |

---

### L-06: Misspellings in Procedure Names

| Attribute | Detail |
|-----------|--------|
| **App** | ExecutionProcessService |
| **Detail** | `InsertDeclearation`, `SelectDecleration`, `UpdateDeclration`, `InsertDeclearationDetalis`, `ESBReferanceNumber` — 5+ unique misspellings |

---

## Prioritized Action Plan

### Phase 1 — Immediate (Pre-Production) 🔴🟠

| # | Issue | Effort | Owner Action |
|---|-------|--------|--------------|
| 1 | **Wait on C-01** (Resolved, was functionally correct) | 0 min | None |
| 2 | **C-02** Fix EventId: add to SELECT + fix loop reference | 15 min | Edit Utils.esql + ProcessingData.esql |
| 3 | **C-03** Add SQL error handlers to ExecSvc fulfillment | 1 hour | Add BEGIN/END blocks around 8 DB calls |
| 4 | **C-04** Quote status `3`/`4` as `'3'`/`'4'` | 5 min | Edit 2 lines in processRequest.esql |
| 5 | **H-02** Remove dead THROW after RESIGNAL (or remove RESIGNAL) | 30 min | Edit 4 error handlers across 3 files |
| 6 | **H-04** Fix EventID list (CREATE LASTCHILD instead of SET) | 10 min | Edit 1 line in processRequest.esql |
| 7 | **H-05** Fix transaction boundaries (single commit) | 45 min | Restructure commit logic in processRequest.esql |

*(Phase 2-4 remains the same)*
