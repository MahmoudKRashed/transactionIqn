# Transaction Inquiry: Full Callback Body Field Mapping

This document provides a comprehensive breakdown of all fields within the `FITransInqCallBackRq` body.

---

## 1. Top-Level Body Fields
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `IdAcctRel` | No | String | Length 2, Pattern: [0-9]{2} |
| `TransNumInvPrtyAcctRel` | No | String | Length 2, Pattern: [0-9]{2} |

## 2. Customer Information (`CustInfo`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `CustName` | Yes | String | Min 2, Max 100 |
| `Id` | Yes | String | Min 1, Max 20 |
| `IdType` | Yes | String | Min 2, Max 10 (e.g., 10 for NID) |
| `Ntnlty` | No | String | Length 3 (ISO Country Code) |

## 3. Account / Card Selection (`TransAccInfo`)
Each entry in the list can contain ONE of the following product types along with its transactions.

### 3.1 Account Information (`AccInfo`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `IBAN` | Yes | String | SA[0-9]{4}[A-Za-z0-9]{18} |
| `Num` | Yes | String | 4 to 24 characters |
| `IsJntAcc` | Yes | String | y / n |
| `AccCur` | Yes | String | 3 chars (e.g., SAR) |
| `PrdUsrsList` | Yes | List | List of associated users |
| └─ `UsrInfo` | Yes | Complex | User identity container |
| &nbsp;&nbsp;&nbsp;├─ `Id` | Yes | String | Min 1, Max 20 |
| &nbsp;&nbsp;&nbsp;├─ `IdType` | Yes | String | Min 2, Max 10 |
| &nbsp;&nbsp;&nbsp;├─ `Name` | Yes | String | Min 2, Max 100 |
| &nbsp;&nbsp;&nbsp;└─ `UsrType` | Yes | String | Role of the user |

### 3.2 Card Information (`CardInfo`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `Num` | Yes | String | 4 to 24 characters |
| `Type` | No | String | Length 2 |
| `Company` | Yes | String | Length 2 (e.g., 01 for Visa) |
| `Name` | No | String | Min 4, Max 100 |
| `IsJntCard` | Yes | String | y / n |
| `PrdUsrsList` | Yes | List | Same as Account User List above |

### 3.3 Additional Relationship (`AdditionalRel`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `Num` | Yes | String | Min 4, Max 50 |
| `Type` | Yes | String | Length 2 |

---

## 4. Transaction Details (`TransDtls`)
Common fields for every transaction record in `TransAccInfo`.

| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `TransNum` | Yes | String | Min 3, Max 65 |
| `TransType` | Yes | String | Length 2 |
| `TransDesc` | No | String | Min 4, Max 100 |
| `TransDate` | Yes | DateTime | ISO format (19 chars) |
| `TransClsf` | Yes | String | Length 2 |
| `FXRate` | Yes | Decimal | Standard Amount |
| `Fees` | Yes | Decimal | Standard Amount |
| `TransAmt` | Yes | Complex | Currency + Value |
| ├─ `Val` | Yes | Decimal | Amount value |
| └─ `Cur` | Yes | String | Currency code |
| `TotalTransAmt` | Yes | Decimal | Positive Amount |
| `TransChannelInfo` | Yes | Complex | Parent container |
| ├─ `ChannelCode` | Yes | String | Length 2 |
| ├─ `PayMethod` | No | String | Length 2 |
| └─ `AddressInfo` | Yes | Complex | Location details |
| &nbsp;&nbsp;&nbsp;├─ `Country` | No | String | Length 3 |
| &nbsp;&nbsp;&nbsp;├─ `City` | No | String | Min 3, Max 100 |
| &nbsp;&nbsp;&nbsp;├─ `District` | No | String | Min 3, Max 100 |
| &nbsp;&nbsp;&nbsp;├─ `Street` | No | String | Min 3, Max 50 |
| &nbsp;&nbsp;&nbsp;├─ `PostalCode`| No | String | Length 5 |
| &nbsp;&nbsp;&nbsp;└─ `Desc` | No | String | Min 3, Max 300 |
| `CreditDueDate` | No | Date | YYYY-MM-DD |
| `DebitDueDate` | No | Date | YYYY-MM-DD |
| `ShopName` | No | String | Min 4, Max 100 |
| `TransExecutor` | No | Complex | Execution Party |
| ├─ `Id` | Yes | String | Executor ID |
| ├─ `IdType` | Yes | String | ID Type |
| └─ `Name` | Yes | String | Executor Name |
| `MoreDtls` | No | Choice | **One of following types:** |

---

## 5. Specific Transaction Types (`MoreDtls`)

### 5.1 Simple Specifics (Deposit, SADAD, Dues, Teller, Note)
| Category | Hierarchy / Field | Mandatory | Type | Constraints |
|----------|-------------------|-----------|------|-------------|
| **Deposit** | `Purpose` | Yes | String | Min 4, Max 100 |
| **SADAD** | ├─ `BillerCode` | Yes | String | Length 2 |
| | ├─ `BillerDesc` | Yes | String | Min 2, Max 100 |
| | ├─ `PaymentNum` | Yes | String | Min 4, Max 30 |
| | └─ `PaymentDesc` | No | String | Min 4, Max 100 |
| **DuesDed** | ├─ `DeductionType`| Yes | String | Length 2 |
| | └─ `DeductionDesc`| No | String | Min 4, Max 200 |
| **Teller** | `RequestNum` | Yes | String | Min 3, Max 50 |
| **Note**| `Num` | Yes | String | Min 3, Max 50 (Promissory Note) |

### 5.2 Transfer Details (`Transfer`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `XferTransInfo` | Yes | Complex | Container |
| ├─ `Type` | Yes | String | Length 2 |
| ├─ `Purpose` | Yes | String | Length 2 |
| ├─ `PurposeDesc` | No | String | Min 4, Max 100 |
| ├─ `Commission` | Yes | Decimal | Max 15 digits |
| ├─ **Sender** | **Yes** | **Choice** | **One of following:** |
| │&nbsp;&nbsp;├─ `AccId` | -- | Complex | Local: Num, IBAN, BIC |
| │&nbsp;&nbsp;└─ `IntAccId` | -- | Complex | Int'l: Num, IBAN, Bank, Name, Country, Address |
| └─ **Receiver** | **Yes** | **Choice** | **One of following:** |
| &nbsp;&nbsp;&nbsp;├─ `Benf` | -- | Complex | Local: IBAN, BIC, Name |
| &nbsp;&nbsp;&nbsp;├─ `IntBenf` | -- | Complex | Int'l: (same as IntAccId) |
| &nbsp;&nbsp;&nbsp;└─ `SARIEEInfo` | -- | Complex | Instant Payment |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;├─ `BenfName`| Yes | String | |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└─ `SARIEEType`| Yes | Choice | NID, RID, AcctNum, Mobile, Email |

### 5.3 Cheque / Exchange / Shares / SAMA Order
| Category | Hierarchy / Field | Mandatory | Type | Constraints |
|----------|-------------------|-----------|------|-------------|
| **Cheque** | ├─ `Num` | Yes | String | Min 2, Max 100 |
| | ├─ `IssuerId` | No | String | Max 20 chars |
| | ├─ `IssuerIdType` | No | String | e.g. 10 |
| | ├─ `IssuerName` | Yes | String | Min 2, Max 100 |
| | ├─ `IssuerBank` | Yes | String | Min 2, Max 100 |
| | ├─ `IssuerBankCountry`| Yes | String | 3 chars (ISO) |
| | ├─ `ExecType` | Yes | String | Length 2 |
| | └─ `EndorsStatus` | Yes | String | Length 2 |
| **Exchange**| ├─ `Costs` | Yes | Decimal | Amount |
| | ├─ `Purpose` | No | String | Min 4, Max 100 |
| | └─ `FundsSrc` | Yes | String | Min 4, Max 200 |
| **Shares** | ├─ `RequestNum` | Yes | String | Min 3, Max 50 |
| | ├─ `SharesCount` | No | String | Max 15 digits |
| | ├─ `EntityName` | Yes | String | Min 4, Max 100 |
| | ├─ `AccNum` | Yes | String | 4 to 24 chars |
| | └─ `BankName` | No | String | Min 2, Max 200 |
| **SAMAOrder**| ├─ `XferTransInfo` | Yes | Complex | (See Transfer structure above) |
| | └─ `SAMARefNum` | Yes | String | Min 3, Max 50 |
