# Data Model — Personal Finance Platform

## 1. Core architectural decision

The backend follows a two-layer design:

- **Transactional layer** (DuckDB raw schema) — double-entry bookkeeping, append-only,
  correctness-focused. This is the source of truth.
- **Analytical layer** (dbt models) — read-only projections of the transactional truth,
  denormalised into shapes that Power BI and Streamlit can query efficiently.

These two layers coexist in DuckDB. The boundary between them is exactly where dbt lives.

---

## 2. Double-entry bookkeeping

Every financial transaction follows double-entry accounting: for every debit there is a
corresponding credit. The sum of all debits must equal the sum of all credits. This is
what makes the books balance.

### Example — grocery purchase by credit card

A €50 Mercadona purchase creates two journal lines:
journal_entry  id=1001  date=2026-04-10  source='bank_import'  status='classified'
journal_line  account='Expenses:Food:Groceries'   dr_cr='DR'  amount=50.00
journal_line  account='Liabilities:CreditCard'    dr_cr='CR'  amount=50.00

When the credit card bill is paid from the bank account:
journal_entry  id=1002  date=2026-04-28  source='bank_import'
journal_line  account='Liabilities:CreditCard'    dr_cr='DR'  amount=50.00
journal_line  account='Assets:BankAccount'        dr_cr='CR'  amount=50.00

The first posting captures the economic event (goods received, liability created).
The second posting settles the liability (cash leaves the bank).

### Debit and credit rules by account type

| Account type | Increases with | Decreases with |
|---|---|---|
| Assets | DR | CR |
| Liabilities | CR | DR |
| Income | CR | DR |
| Expenses | DR | CR |

---

## 3. The journal_entry / journal_line pattern

`journal_entry` is the **header** — it represents one financial event.
`journal_line` contains the **individual account postings** for that event.

### Key rules
- Every `journal_entry` must have at least two `journal_line` rows
- The sum of DR lines must equal the sum of CR lines (the entry must balance)
- Both tables are **append-only** — rows are never updated or deleted
- Corrections are posted as **reversing entries** — a new journal_entry that mirrors
  the original with DR and CR swapped, followed by a new corrected entry

### Why append-only matters
Append-only preserves a complete, tamper-evident audit trail. You can always reconstruct
the exact state of any account at any point in time by replaying the journal from the
beginning. Updating or deleting rows would destroy this property.

---

## 4. Account hierarchy

Accounts are stored in a single self-referencing table using the adjacency list pattern.
`account.parent_account_id` references `account.account_id`.

This supports up to 5 levels of hierarchy:
Assets                          (level 1)
Bank Accounts                 (level 2)
Santander Current           (level 3)
Savings                       (level 2)
Emergency Fund              (level 3)
Expenses                        (level 1)
Food                          (level 2)
Groceries                   (level 3)
Restaurants                 (level 3)
Housing                       (level 2)
Rent                        (level 3)
Utilities                   (level 3)
Electricity               (level 4)
Water                     (level 4)

### Recursive CTE for hierarchy traversal

DuckDB supports recursive CTEs natively. The following pattern is used in
`int_account_balances` to flatten the hierarchy and produce level_1 through level_5
columns for Power BI drill-down:

```sql
WITH RECURSIVE account_tree AS (
  -- Anchor: root accounts (no parent)
  SELECT
    account_id,
    account_name,
    parent_account_id,
    account_type,
    account_code,
    account_name AS level_1,
    NULL         AS level_2,
    NULL         AS level_3,
    NULL         AS level_4,
    NULL         AS level_5,
    1            AS depth
  FROM stg_accounts
  WHERE parent_account_id IS NULL

  UNION ALL

  -- Recursive: child accounts
  SELECT
    a.account_id,
    a.account_name,
    a.parent_account_id,
    a.account_type,
    a.account_code,
    CASE h.depth WHEN 0 THEN a.account_name ELSE h.level_1 END,
    CASE h.depth WHEN 1 THEN a.account_name ELSE h.level_2 END,
    CASE h.depth WHEN 2 THEN a.account_name ELSE h.level_3 END,
    CASE h.depth WHEN 3 THEN a.account_name ELSE h.level_4 END,
    CASE h.depth WHEN 4 THEN a.account_name ELSE h.level_5 END,
    h.depth + 1
  FROM stg_accounts a
  JOIN account_tree h ON a.parent_account_id = h.account_id
)
SELECT
  h.*,
  SUM(
    CASE WHEN jl.dr_cr = 'DR' THEN jl.amount ELSE -jl.amount END
  ) AS balance
FROM account_tree h
LEFT JOIN stg_journal_lines jl ON jl.account_id = h.account_id
GROUP BY ALL;
```

This single model gives Power BI a dimension table where every account carries its
full hierarchy path — enabling drill-down from level_1 to level_5 in DAX with no
extra work.

---

## 5. Envelope design

### What envelopes are
Virtual savings buckets — money mentally allocated for a future cost or savings goal.
Examples: holiday fund, car insurance (annual), emergency fund top-up.

### Design decision — envelopes as analytical overlay
Envelopes are NOT a separate ledger. They are an analytical overlay on top of real
journal lines.

- The `envelope` table defines each envelope and its savings target
- Each `journal_line` has an optional `envelope_id` foreign key
- An envelope's `current_balance` is always **derived** by summing the journal lines
  tagged to it — it is never stored directly
- This keeps the double-entry books clean — no phantom balances, no parallel ledger

### Why this matters
If envelopes were a separate ledger you would have two sources of truth that could
drift out of sync. By tagging journal lines with envelope_id, the envelope balance
is always mathematically consistent with the underlying transactions.

### Virtual loans between envelopes
When money is borrowed from one envelope to cover a current need, this is recorded
as a pair of journal entries:
- Entry 1: debit the destination account, credit the source envelope account
- Entry 2 (repayment): debit the source envelope account, credit the destination account

This keeps the loan trackable and reversible within the same double-entry framework.

---

## 6. Budget versioning

Each CSV budget load creates a new `budget_version` row with:
- A unique `budget_version_id`
- A `loaded_at` timestamp
- An `is_current` boolean flag

`budget_line` always carries `budget_version_id` so every version is permanently
preserved. The dbt staging model `stg_budget_lines` filters to `is_current = true`
by default, but historical versions remain queryable for variance analysis.

---

## 7. Bank reference mapping

`bank_reference_map` is a lookup table that defines how bank transaction references
map to accounts:

- If a bank reference matches exactly one account → auto-classify the transaction
- If a bank reference is ambiguous or unknown → flag as unclassified

Unclassified transactions trigger the manual assignment flow:
int_classified_txns flags row as unclassified
→ Prefect detects on daily run
→ Gmail API sends assignment request
→ Streamlit UI presents transaction for manual split / classification
→ User input written back to journal_line in DuckDB


---

## 8. Key database tables summary

| Table | Layer | Purpose |
|---|---|---|
| account | Raw | Chart of accounts with self-referencing hierarchy |
| journal_entry | Raw | Transaction header — one row per financial event |
| journal_line | Raw | Transaction lines — debits and credits |
| budget_version | Raw | Versioned budget load metadata |
| budget_line | Raw | Monthly budget amounts by account and version |
| bank_reference_map | Raw | Reference pattern to account mapping matrix |
| envelope | Raw | Virtual savings bucket definitions |
| loan | Raw | Bank loans, mortgages, personal loans tracking |

---

## 9. dbt model layers

| Layer | Prefix | Materialisation | Purpose |
|---|---|---|---|
| Staging | stg_* | Views | Type casting, renaming, light cleaning |
| Intermediate | int_* | Ephemeral or views | Business logic, classification, rollups |
| Marts | mart_* | Tables | Final analytical models for BI and Streamlit |

### Mart models
- `mart_actuals` — full classified transaction ledger
- `mart_budget_vs_actuals` — monthly variance by account
- `mart_account_hierarchy` — flattened dimension table for BI drill-down
- `mart_envelopes` — envelope balances and savings targets
- `mart_loans` — amortisation schedules