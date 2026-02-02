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
| `TransType` | Yes | String | Length 2 (Refer to BRD Types 01-14) |
| `AccId` | No | Complex | Target Account (Optional) |
| ├─ `Num` | Yes | String | Account Number |
| ├─ `IBAN`| Yes | String | IBAN format |
| └─ `BIC` | Yes | String | Bank Identifier Code |
| `FinRelType`| No | String | Length 2 (Pattern [0-9]{2}) |
| `TransAmt` | No | Complex | Filter by Amount |
| ├─ `Val` | Yes | Decimal | Amount value |
| └─ `Cur` | Yes | String | Currency (e.g., SAR) |
