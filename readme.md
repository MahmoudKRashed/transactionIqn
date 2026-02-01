# Transaction Inquiry: MoreDtls Field Mapping

This document provides a detailed breakdown of the fields available under `MoreDtls` within the Transaction Inquiry callback. This information is intended for the business team to confirm the expected data and their types.

---

## 1. Deposit Details (`Deposit`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `Purpose` | Yes | String | Min Length: 4, Max Length: 100 |

## 2. SADAD Details (`SADAD`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `BillerCode` | Yes | String | Fixed Length: 2 |
| `BillerDesc` | Yes | String | Min Length: 2, Max Length: 100 |
| `PaymentNum` | Yes | String | Min Length: 4, Max Length: 30 |
| `PaymentDesc` | No | String | Min Length: 4, Max Length: 100 |

## 3. Dues Deduction Details (`DuesDeduction`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `DeductionType` | Yes | String | Fixed Length: 2 |
| `DeductionDesc` | No | String | Min Length: 4, Max Length: 200 |

## 4. Transfer Details (`Transfer`)
Sub-structure: `XferTransInfo`

| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `Type` | Yes | String | Fixed Length: 2 |
| `Purpose` | Yes | String | Fixed Length: 2 |
| `PurposeDesc` | No | String | Min Length: 4, Max Length: 100 |
| `Commission` | Yes | Decimal | Up to 15 digits |
| `Sender` | Yes | Choice | See [Sender/Receiver Details](#sender-receiver-details) |
| `Receiver` | Yes | Choice | See [Sender/Receiver Details](#sender-receiver-details) |

## 5. Cheque Details (`Cheque`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `Num` | Yes | String | Min Length: 2, Max Length: 100 |
| `IssuerId` | No | String | Min Length: 1, Max Length: 20 |
| `IssuerIdType` | No | String | e.g., NID, RID |
| `IssuerName` | Yes | String | Min Length: 2, Max Length: 100 |
| `IssuerBank` | Yes | String | Min Length: 2, Max Length: 100 |
| `IssuerBankCountry` | Yes | String | Fixed Length: 3 (ISO code) |
| `ExecType` | Yes | String | Fixed Length: 2 |
| `EndorsStatus` | Yes | String | Fixed Length: 2 |

## 6. Teller Details (`Teller`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `RequestNum` | Yes | String | Min Length: 3, Max Length: 50 |

## 7. Promissory Note Details (`PromissoryNote`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `Num` | Yes | String | Min Length: 3, Max Length: 50 |

## 8. Exchange Details (`Exchange`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `Costs` | Yes | Decimal | Up to 15 digits |
| `Purpose` | No | String | Min Length: 4, Max Length: 100 |
| `FundsSrc` | Yes | String | Min Length: 4, Max Length: 200 |

## 9. Shares Details (`Shares`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `RequestNum` | Yes | String | Min Length: 3, Max Length: 50 |
| `SharesCount` | No | String | Max Length: 15, Pattern: `[0-9]+` |
| `EntityName` | Yes | String | Min Length: 4, Max Length: 100 |
| `AccNum` | Yes | String | 4 to 24 chars |
| `BankName` | No | String | Min Length: 2, Max Length: 200 |

## 10. SAMA Order Details (`SAMAOrder`)
| Field | Mandatory | Type | Constraints |
|-------|-----------|------|-------------|
| `XferTransInfo` | Yes | Complex | See [Transfer Section](#4-transfer-details) |
| `SAMARefNum` | Yes | String | Min Length: 3, Max Length: 50 |

---

<a name="sender-receiver-details"></a>
## Sender / Receiver Details

### Local Account (`AccId`)
- `Num`: Account Number (4-24 chars).
- `IBAN`: Pattern `SA[0-9]{4}[A-Za-z0-9]{18}`.
- `BIC`: Bank Identification Code (5 digits).

### International Account (`IntAccId` / `IntBenf`)
- `Num`: Account Number (3-32 chars).
- `IBAN`: Optional.
- `BankName`: Mandatory (4-50 chars).
- `Name`: Mandatory (2-100 chars).
- `Address`: Optional.
- `Country`: Mandatory (3 chars).

### Beneficiary (`Benf`)
- `IBAN`: Mandatory.
- `BIC`: Mandatory.
- `Name`: Mandatory.

### SARIEE Info (`SARIEEInfo`)
- `BenfName`: Mandatory.
- `SARIEEType` (Choice):
    - `NID`: 10 digits.
    - `RID`: 10 digits.
    - `AcctNum`: Account number.
    - `MobileNo`: 4-24 digits.
    - `Email`: Valid email format.
