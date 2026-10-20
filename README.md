# SAP S/4HANA FI: End-to-End Financial Accounting Implementation — Nova Retail GmbH

This project documents SAP S/4HANA Financial Accounting (FI) configuration and business-process testing for the fictional company Nova Retail GmbH. The work covers foundational FI settings, Procure-to-Pay (P2P) and Order-to-Cash (O2C).

> All company, supplier, customer and account data is fictional. No client or employer data is included.
---

## 1. Business scenario

Nova Retail GmbH is a fictional mid-sized German retailer expanding into a new legal entity. It needs a working finance foundation in SAP S/4HANA so the company can post and report its books in EUR.

### Requirements

| # | Requirement | Why it matters |
|---|---|---|
| 1 | A separate legal entity with its own books | Statutory reporting per company code |
| 2 | A chart of accounts with G/L accounts | The structure behind every posting |
| 3 | Controlled posting periods | Prevents postings into closed periods |
| 4 | Field and number controls | Data quality and a clean audit trail |
| 5 | Posting limits for users | Control over who can post what |
| 6 | Proof that the setup works | Positive and negative test cases |

---

## 2. Enterprise structure

![Organizational structure](org.jpeg)

| Object | Value |
|---|---|
| Company code | NR01, Nova Retail GmbH |
| Country / currency | DE / EUR |
| Chart of accounts | NRCA (6-digit, manual numbering) |
| Fiscal year variant | K4 (calendar year, 12 periods + 4 special periods) |
| Field status variant | NR01 |
| Posting period variant | NR01 |
| Ledger | 0L (leading ledger) |

---

## 3. Configuration steps

| # | Step | Transaction | What I did and why |
|---|---|---|---|
| 1 | Company code | Copy / OBY6 | Created NR01. Captured the settings before and after to show the change |
| 2 | Chart of accounts | OB13, OB62 | Created NRCA and assigned it to NR01. One shared list of accounts |
| 3 | Fiscal year variant | OB29, OB37 | Checked that standard K4 exists and reused it instead of creating a custom variant |
| 4 | Field status variant | OBC4, OBC5 | Created variant NR01 and assigned it to the company code |
| 5 | Field status groups | OBC4 | Created N001 (expense, cost center required), N002 (bank), N003 (reconciliation) |
| 6 | Posting period variant | OBBO, OBBP | Created NR01 and assigned it to the company code |
| 7 | Open posting periods | OB52 | Opened only period 10/2026 for +, A, D, K, M and S |
| 8 | Account groups | OBD4 | BANK, RECV, PAY, REV, EXP with separate number ranges |
| 9 | Document number ranges | FBN1 | Defined interval 01 for G/L documents (SA) |
| 10 | G/L accounts | FS00 | Created six accounts and extended them to NR01 |
| 11 | Employee tolerance group | OBA4 | Blank (default) group with a posting limit |
| 12 | Ledger settings | FINSC_LEDGER | S/4HANA universal journal: NR01 assigned to ledger 0L |
| 13 | Document splitting | Classify G/L accounts | Assigned item categories to the NRCA accounts by range |

![OBY6 before: global data blank](oby6_global_data_blank.png)
![OBY6 after: company code completed](company_code_oby6_completed.png)

### Before and after: company code global data

| Field | Before | After |
|---|---|---|
| Chart of accounts | empty | NRCA |
| Fiscal year variant | empty | K4 |
| Field status variant | empty | NR01 |
| Posting period variant | empty | NR01 |

### G/L accounts

| Account | Description | Group | Type | Field status group |
|---|---|---|---|---|
| 100000 | Bank EUR | BANK | Balance sheet | N002 |
| 140000 | Customer reconciliation | RECV | Balance sheet | N003 |
| 160000 | Vendor reconciliation | PAY | Balance sheet | N003 |
| 400000 | Revenue | REV | Primary costs or revenue | N003 |
| 500000 | Cost of goods sold | EXP | Primary costs or revenue | N002 |
| 600000 | Office expense | EXP | Primary costs or revenue | N001 |

Reconciliation accounts 140000 and 160000 carry the customer and vendor reconciliation account types, so they can only be posted to through the sub-ledgers.

![G/L accounts creation](Gl_accounts_creation.png)

### Configuration evidence

**Company code (OX02, OBY6)**

![Company code in OX02](company_code_ox02.png)
![OBY6 overview](OBY6_overview.png)

**Fiscal year variant (OB29)**

![Fiscal year variants](ob29_fiscal_year_variants.png)

**Field status variant assignment (OBC5)**

![Assign field status variant](assign_field_status_variant.png)

**Posting periods (OB52)**

![Posting periods OB52](posting_periods_ob52.png)
![Posting periods open](ob52_posting_periods_open.png)

**Account groups (OBD4)**

![G/L account groups overview](gl_account_groups_overview.png)

**Document splitting customizing (SPRO)**

![SPRO](SPRO.png)

**Ledger consistency check (FINS_CUST_CONS_CHK_P)**

Posting was blocked with message FINS_ACDOC_CUST201. In S/4HANA, a company code must have consistent ledger settings for the universal journal before it can post. I ran the consistency check for company code NR01 and ledger 0L. The error log showed [exact message from your screenshot]. I corrected [the setting you changed] in FINSC_LEDGER and ran the check again, which passed. Posting then worked.


![Ledger check: error log](fins_cust_chk_ledger_error_log.png)
![Ledger check: passed](fins_cust_chk_ledger_check_passed.png)

---

## 4. Test cases (FI foundation)

All tests were run in FB50 (G/L account document), company code NR01, document type SA, currency EUR. For each test I used Simulate first, then Post, and kept a screenshot with the status bar message visible.

| # | Test | Entry | Expected result |
|---|---|---|---|
| 1 | Post in an open period | Dr 500000 / Cr 100000, 1.000,00 EUR, posting date 14.10.2026 | Document posted |
| 2 | Post in a closed period | Same entry, posting date 15.09.2026 | Error F5201: posting period not open |
| 3 | Unbalanced entry | Dr 500000 1.000,00 / Cr 100000 900,00 | Document cannot be posted, difference is not zero |
| 4 | Account not extended to the company code | Dr 690000 / Cr 100000, 1.000,00 EUR | Error: account not created in company code NR01 |
| 5 | Reverse a posted document | FB08 on the test 1 document | Reversal document created, balances return to zero |
| 6 | Post directly to a reconciliation account | Dr 140000 / Cr 100000, 1.000,00 EUR | Error: reconciliation account cannot be posted to directly |
| 7 | Cost center required | Dr 600000 without cost center / Cr 100000 | Error for missing cost center (or not testable without CO setup) |

### Test results

#### Test 1: Post in an open period
**Result:** 
![Test 1: entry in open period](Post_to%20_open%20period.png)
![Test 1: document created](Document_created.png)
![Test 1: FB03 display](fb03.png)

#### Test 2: Post in a closed period
**Result:** 

![Test 2: posting period error](posting_period_error_test.png)

#### Test 3: Unbalanced entry
**Result:** 


#### Test 4: Account not in company code
**Result:** 

![Test 4: account not in company code](Account_not_in%20_company_code_testing.png)

#### Test 5: Reversal
**Result:** 

![Test 5: FB08 reversal](FB08.png)
![Test 5: reversal document](fb08_.png)

#### Test 6: Reconciliation account
**Result:** `[FILL IN: exact status bar message]`

![Test 6: reconciliation account error](reconciliation%20account_ERROR.png)

#### Test 7: Cost center required
**Result:** `[FILL IN: exact status bar message, or "not testable: no controlling area for NR01"]`

![Test 7: missing cost center error](without_cost_cent_Error.png)



## 5. Project 2: Procure-to-Pay (P2P)

**Objective:** Process a supplier invoice, post an outgoing payment, and verify that the supplier open item is cleared.

**Process flow:** Supplier Master → Supplier Invoice → Outgoing Payment → Clearing Verification

| Step | SAP transaction | Result |
|---|---|---|
| Create/extend supplier | BP | Supplier 1000592 created and extended to company code NR01 |
| Post supplier invoice | FB60 | Document 1900000000 |
| Post outgoing payment | F-53 | Payment document 1500000000 |
| Verify supplier items | FBL1N | Invoice confirmed cleared |

**Accounting entries**

| Step | Debit | Credit |
|---|---|---|
| Supplier invoice | Expense account | Supplier (reconciliation account 160000) |
| Outgoing payment | Supplier (160000) | Bank (100000) |

**Key learning:** Supplier master data, accounts payable postings, outgoing payments, and vendor open-item clearing.

**Screenshots**


![Supplier business partner](Vendor.png)
![Simulation check before posting](simulate_check.png)
![Supplier invoice 1900000000](invoice%201900000000.png)
![Supplier invoice 1900000000 in FB03](invoice%201900000000%20in%20FB03.png)
![Outgoing payment 1500000000](inv_1500000000.png)
![Outgoing payment 1500000000 in FB03](payment_1500000000%20in%20FB03.png)
![FBL1N cleared supplier item](FBL1N_showing%20the%20cleared%20invoice.png)

---

## 6. Project 3: Order-to-Cash (O2C)

**Objective:** Create a customer, post a customer invoice, record an incoming payment, and verify clearing.

**Process flow:** Customer Master → Customer Invoice → Incoming Payment → Clearing Verification

| Step | SAP transaction | Result |
|---|---|---|
| Create/extend customer | BP | Customer 1000593 extended to company code NR01 |
| Post customer invoice | FB70 | Document 1800000000 |
| Post incoming payment | F-28 | Payment document 1400000000 |
| Verify customer items | FBL5N | Invoice confirmed cleared |

**Accounting entries**

| Step | Debit | Credit |
|---|---|---|
| Customer invoice | Customer (reconciliation account 140000) | Revenue (400000) |
| Incoming payment | Bank (100000) | Customer (140000) |

**Key learning:** Accounts receivable, customer invoicing, incoming payments, and customer open-item clearing.

**Screenshots**

![Customer business partner](Customer_Master_Created.png)
![Customer invoice 1800000000 posted](1800000000_%20posted%20.png)
![FBL5N open items](Open_items.png)
![Incoming payment F-28](F-28.png)
![Incoming payment 1400000000 posted](Document%201400000000%20was%20posted%20i.png)
![FBL5N cleared customer items](cleared_items.png)

---

## 7. Additional process: Automatic payments (F110)

SAP also supports automatic outgoing supplier payments through transaction F110. This can be explored as an extension of the P2P process.
SAP transaction F110 (Automatic Payment Program) automates supplier payments by selecting eligible open invoices based on due dates, payment methods, and configured payment rules. It is useful for businesses processing multiple supplier payments in scheduled payment cycles, reducing manual effort and improving payment control.

F110 vs. F-53: F-53 is used for manual outgoing payments, while F110 supports automated payment selection, proposal review, payment execution, and payment-medium generation.


**Typical process flow:** Open Supplier Invoice → Payment Parameters → Proposal Run → Review Exceptions → Payment Run → Payment Medium

**Key configuration and testing areas**

- Payment methods and paying company codes (FBZP)
- House bank and bank determination
- Supplier payment details
- Payment proposal review and exception handling
- Payment execution and payment-medium verification

**Status:**  The manual outgoing payment (F-53) in Project 2 was used for the completed cycle.

![F110 payment program](F110.png)
![F110 payment run](F110_1.png)

---


## 8. Key learnings

1. A new S/4HANA company code needs much more than the obvious configuration. Ledger settings, document splitting, tolerance groups and number ranges each block the first posting if missing.
2. Each error message points to exactly one missing setting. Reading it carefully is faster than guessing.
3. Verify at table level (T004, FBN1, SE16N) rather than trusting a dropdown.
4. Posting period control checks the + row first, then the specific account type.



## 9. Month-End Closing : October 2026

### Business Scenario

Nova Retail GmbH requires a structured month-end closing process to review General Ledger balances, monitor outstanding customer and supplier items, record accrued expenses, and review financial statements for October 2026.

**Company Code:** NR01  
**Ledger:** 0L  
**Currency:** EUR  
**Fiscal Year:** 2026  
**Closing Period:** October 2026

### Closing Activities

| Step | Activity | SAP Transaction / Tool | Objective |
|---|---|---|---|
| 1 | G/L balance review | FS10N / FAGLB03 | Review account balances and posting activity |
| 2 | Supplier open-item review | FBL1N | Identify outstanding supplier invoices and payments |
| 3 | Customer open-item review | FBL5N | Identify outstanding customer invoices and receipts |
| 4 | Accrued expense posting | FB50 | Record expenses incurred but not yet invoiced |
| 5 | Financial statement review | Applicable G/L reporting tools | Review Profit & Loss and Balance Sheet balances |
| 6 | Closing checklist | Manual documentation | Record exceptions and outstanding activities |

### Accrued Expense : Example

**Business scenario:** Office services worth €500 were consumed during October 2026, but the supplier invoice had not been received by month-end.

Illustrative accounting entry:

| G/L Account | Debit | Credit |
|---|---:|---:|
| Office Expense — 600000 | €500 | — |
| Accrued Expenses Liability — suitable configured account | — | €500 |
| **Total** | **€500** | **€500** |

**Proposed posting date:** 31 October 2026  
**Proposed transaction:** FB50

The liability account must be created or verified in the SAP system before posting. The entry should be simulated and checked before it is posted. If required and configured, a reversal can be scheduled for November 2026.

### Expected Controls

- Review G/L balances and investigate unexpected balances.
- Identify outstanding supplier and customer items.
- Confirm that accrual entries are supported and balanced.
- Review financial statements for unusual movements.
- Maintain a closing checklist and document identified exceptions.

### Project Status

The month-end closing workflow and illustrative accounting entries are documented as an extension of the Nova Retail GmbH FI case study. 

---


## Disclaimer

Built in an SAP training system. All company, supplier, customer and account data is fictional. No client or employer data is included.
