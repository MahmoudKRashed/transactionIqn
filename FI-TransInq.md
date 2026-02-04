# Transaction Inquiry: Request Field Mapping (FITransInqRq)

This document details the structure and fields of the `FITransInqRq` message, which is sent by the requester to inquire about specific transactions.

---

## 1. Request Header (MsgHdrRq)
Refer to common SAMA Header standards. Key fields include:
| Field | Mandatory | Type | Description |
|-------|-----------|------|-------------|
| `PID` | Yes | String | Organization PID (e.g., SAMA) |
| `Sys` | No  | String | Source System Name |
| `MsgDtTm`| Yes | DateTime | Message timestamp |
| `SRN` | No  | String | Service Request Number |

---

## 2. Requester Details (`Rqstr`)
Identifies the individual/officer making the inquiry.

| Hierarchy / Field | Mandatory | Type | Constraints / Description |
|-------------------|-----------|------|---------------------------|
| `PID` | Yes | String | Org ID (Length 5) |
| `RID` | Yes | String | Requester ID (Digits only, 2-20) |
| `Name`| Yes | String | Full Name (Min 1, Max 50) |
| `Pstn`| Yes | String | Position/Title |
| `RRN` | Yes | String | Reference Number (internal) |
| `GeoLoc`| Yes | String | Location Coordinates (Length 18) |
| `GeoLocDsc`| No | String | Location Description |

---

## 3. Involved Party (`InvPrty`)
The target of the inquiry (e.g., a citizen or a company). This is a **Choice**.

### 3.1 Individual (`Indv`)
| Hierarchy / Field | Mandatory | Type | Constraints / Description |
|-------------------|-----------|------|---------------------------|
| `Id` | Yes | String | Identification Number |
| `IdType`| Yes | String | ID Type code (e.g., 10 for NID) |
| `frstName`| Yes | String | First Name |
| `scndName`| No  | String | Second Name |
| `thrdName`| No  | String | Third Name |
| `lstName` | Yes | String | Last Name |
| `Ntnlty` | No  | String | 3-char Country Code (ISO) |

### 3.2 Corporate / Other Types
- `Corp`: `Id`, `IdType`, `Name`
- `Gov`: `Id` (Opt), `IdType` (Opt), `Name`
- `Chrty`: Same as Corp
- `Chmbr`: Same as Corp
- `Adj`: Same as Corp

---

## 4. Inquiry Outline (`Outline`)
Specific search criteria for the transaction inquiry.

| Hierarchy / Field | Mandatory | Type | Constraints / Description |
|-------------------|-----------|------|---------------------------|
| `InqType` | Yes | Choice | **Select ONE of below:** |
| ├─ `TransNum` | 1 | String | Specific Transaction Reference |
| └─ `DtTmDuration` | 1 | Complex | Date/Time Range |
| &nbsp;&nbsp;&nbsp;├─ `StrtDtTm` | Yes | DateTime | Start of period |
| &nbsp;&nbsp;&nbsp;└─ `EndDtTm` | Yes | DateTime | End of period |
| `TransType` | Yes | String | Length 2 (Refer to BRD Types 01-15) |
| `AccId` | No | Complex | Target Account (Optional) |
| ├─ `Num` | Yes | String | Account Number |
| ├─ `IBAN`| Yes | String | IBAN format |
| └─ `BIC` | Yes | String | Bank Identifier Code |
| `FinRelType`| No | String | Length 2 (Pattern [0-9]{2}) |
| `TransAmt` | No | Complex | Filter by Amount |
| ├─ `Val` | Yes | Decimal | Amount value |
| └─ `Cur` | Yes | String | Currency (e.g., SAR) |

---

## 5. Transaction Type & Data Relationship (BRD Alignment)
The mandatory/optional status of data blocks depends on the inquiry method.

### 5.1 Scenario A: Unique Transaction Number Search
When a specific `TransNum` is provided, follow the BRD baseline for direct record retrieval. `MoreDtls` is generally forbidden except for Transfer (03), SAMA Order (14), FX (10), and Other (15).

### 5.2 Scenario B: Duration Search (Granular Matrix)
When searching by date range, use the matrix below (Ref: BRD Page 13):

| Code | Type (Arabic) | Type (English) | Secondary Criteria | AccInfo | CardInfo | MoreDtls |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **01** | إيداع نقدي | Cash Deposit | ATM | Mand. | Mand. | No |
| | | | Branch | Mand. | No | No |
| **02** | سحب نقدي | Withdrawal | Mada Card | Mand. | Mand. | No |
| | | | Credit Card | Opt. | Mand. | No |
| | | | Branch | Mand. | No | No |
| **05** | سداد | SADAD | Mada Card | Mand. | Mand. | No |
| | | | Credit Card | Opt. | Mand. | No |
| | | | Account Relation | Mand. | Opt. | No |
| **06** | مشتريات | POS/Internet | Mada Card | Mand. | Mand. | No |
| | | | Credit Card | Opt. | Mand. | No |
| | | | Account Relation | Mand. | Opt. | No |
| **07** | خصم مستحقات | Dues Deduction| None | Mand. | Opt. | No |
| **03** | تحويل | Transfer | Account Relation | Mand. | Opt. | No |
| | | | Remit Membership | Opt. | Opt. | Mand. |
| | | | No relation in req | Opt* | Opt* | Opt* (At least 1) |
| **14** | أوامر البنك | SAMA Order | Account Relation | Mand. | Opt. | No |
| | | | Remit Membership | Opt. | Opt. | Mand. |
| | | | No relation in req | Opt* | Opt* | Opt* (At least 1) |
| **04** | شيكات | Cheques | None | Mand. | Opt. | No |
| **08** | طلب أمين صندق | Teller Box | None | Mand. | Opt. | No |
| **09** | كمبيالة | Promissory Note| None | Mand. | Opt. | No |
| **10** | شراء أو بيع عملة| FX Exchange | Account Relation | Mand. | Opt. | No |
| | | | Remit Membership | Opt. | No | Mand. |
| | | | No relation in req | Opt* | Opt* | Opt* (At least 1) |
| **11** | تسوية بطاقات | CC Settlement | None | Opt. | Mand. | No |
| **12** | الأسهم | Shares | None | Mand. | Opt. | No |
| **13** | التمويل بالتورق | Tawarruq | None | Mand. | Opt. | No |
| **15** | عمليات أخرى | Other | Any | Opt* | Opt* | Opt* |

*\* Refer to the master callback mapping for technical XML tags ($MoreDtls).*
