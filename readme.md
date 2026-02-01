# Transaction Inquiry: MoreDtls Field Mapping

This document provides a detailed breakdown of the fields available under `MoreDtls` within the Transaction Inquiry callback. All nested child fields are included in their respective tables.

---

## 1. Deposit Details (`Deposit`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `Purpose` | Yes | String | Min Length: 4, Max Length: 100 |

## 2. SADAD Details (`SADAD`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `BillerCode` | Yes | String | Fixed Length: 2 |
| `BillerDesc` | Yes | String | Min Length: 2, Max Length: 100 |
| `PaymentNum` | Yes | String | Min Length: 4, Max Length: 30 |
| `PaymentDesc` | No | String | Min Length: 4, Max Length: 100 |

## 3. Dues Deduction Details (`DuesDeduction`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `DeductionType` | Yes | String | Fixed Length: 2 |
| `DeductionDesc` | No | String | Min Length: 4, Max Length: 200 |

## 4. Transfer Details (`Transfer`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `XferTransInfo` | Yes | Complex | Parent Container |
| -- `Type` | Yes | String | Fixed Length: 2 |
| -- `Purpose` | Yes | String | Fixed Length: 2 |
| -- `PurposeDesc` | No | String | Min Length: 4, Max Length: 100 |
| -- `Commission` | Yes | Decimal | Max 15 digits |
| -- **Sender** | **Yes** | **Choice** | **One of the following:** |
| -- -- `AccId` | -- | Complex | Local Account |
| -- -- -- `Num` | Yes | String | 4 to 24 chars |
| -- -- -- `IBAN` | Yes | String | SA[0-9]{4}[A-Za-z0-9]{18} |
| -- -- -- `BIC` | Yes | String | 5 digits |
| -- -- `IntAccId` | -- | Complex | International Account |
| -- -- -- `Num` | Yes | String | 3 to 32 chars |
| -- -- -- `IBAN` | No | String | Optional |
| -- -- -- `BankName` | Yes | String | 4 to 50 chars |
| -- -- -- `Name` | Yes | String | 2 to 100 chars |
| -- -- -- `Address` | No | String | Optional |
| -- -- -- `Country` | Yes | String | 3 chars (ISO) |
| -- **Receiver** | **Yes** | **Choice** | **One of the following:** |
| -- -- `Benf` | -- | Complex | Local Beneficiary |
| -- -- -- `IBAN` | Yes | String | SA... IBAN format |
| -- -- -- `BIC` | Yes | String | 5 digits |
| -- -- -- `Name` | Yes | String | 2 to 100 chars |
| -- -- `IntBenf` | -- | Complex | International Beneficiary (same as IntAccId above) |
| -- -- `SARIEEInfo` | -- | Complex | Local Instant Payment |
| -- -- -- `BenfName` | Yes | String | Beneficiary Name |
| -- -- -- `SARIEEType` | Yes | Choice | NID, RID, AcctNum, Mobile, or Email |

## 5. Cheque Details (`Cheque`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `Num` | Yes | String | Min 2, Max 100 |
| `IssuerId` | No | String | Max 20 chars |
| `IssuerIdType` | No | String | e.g. 10 for NID |
| `IssuerName` | Yes | String | Min 2, Max 100 |
| `IssuerBank` | Yes | String | Min 2, Max 100 |
| `IssuerBankCountry` | Yes | String | 3 chars (ISO) |
| `ExecType` | Yes | String | Fixed Length: 2 |
| `EndorsStatus` | Yes | String | Fixed Length: 2 |

## 6. Teller Details (`Teller`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `RequestNum` | Yes | String | Min 3, Max 50 |

## 7. Promissory Note Details (`PromissoryNote`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `Num` | Yes | String | Min 3, Max 50 |

## 8. Exchange Details (`Exchange`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `Costs` | Yes | Decimal | Max 15 digits |
| `Purpose` | No | String | Min 4, Max 100 |
| `FundsSrc` | Yes | String | Min 4, Max 200 |

## 9. Shares Details (`Shares`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `RequestNum` | Yes | String | Min 3, Max 50 |
| `SharesCount` | No | String | Max 15 digits |
| `EntityName` | Yes | String | Min 4, Max 100 |
| `AccNum` | Yes | String | 4 to 24 chars |
| `BankName` | No | String | Min 2, Max 200 |

## 10. SAMA Order Details (`SAMAOrder`)
| Hierarchy / Field | Mandatory | Type | Constraints |
|-------------------|-----------|------|-------------|
| `XferTransInfo` | Yes | Complex | Parent Container |
| -- `Type` | Yes | String | Fixed Length: 2 |
| -- `Purpose` | Yes | String | Fixed Length: 2 |
| -- `PurposeDesc` | No | String | Min 4, Max 100 |
| -- `Commission` | Yes | Decimal | Max 15 digits |
| -- **Sender** | **Yes** | **Choice** | **One of the following:** |
| -- -- `AccId` | -- | Complex | Local Account (see fields in Transfer section) |
| -- -- -- `Num` | Yes | String | 4 to 24 chars |
| -- -- -- `IBAN` | Yes | String | SA... format |
| -- -- -- `BIC` | Yes | String | 5 digits |
| -- -- `IntAccId` | -- | Complex | International Account |
| -- -- -- `BankName` | Yes | String | 4 to 50 chars |
| -- -- -- `Name` | Yes | String | 2 to 100 chars |
| -- **Receiver** | **Yes** | **Choice** | **One of the following:** |
| -- -- `Benf` | -- | Complex | Local Beneficiary |
| -- -- -- `IBAN` | Yes | String | SA... format |
| -- -- -- `BIC` | Yes | String | 5 digits |
| -- -- -- `Name` | Yes | String | 2 to 100 chars |
| -- -- `IntBenf` | -- | Complex | International Beneficiary |
| -- -- `SARIEEInfo` | -- | Complex | Local Instant Payment |
| `SAMARefNum` | Yes | String | Min 3, Max 50 |
