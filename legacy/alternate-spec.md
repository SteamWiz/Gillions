# Bookkeeper: Double-Entry Bookkeeping System with AI Agent Interface

## Overview

Bookkeeper is a double-entry bookkeeping system for a small, single-user business. It stores financial data in SQLite, exposes all operations through a Python core library, and provides two interfaces: a CLI for direct scripting and an MCP server for interaction through an LLM agent (Claude).

The system must handle multiple currencies. The owner operates primarily in CAD but occasionally transacts in USD and potentially other currencies.

---

## Architecture

```
Claude (LLM Agent) → MCP Server → Core Library → SQLite
Owner (terminal)   → CLI        → Core Library → SQLite
```

### Layers

**SQLite Database** — Single-file database. No external database server. All schema migrations managed by the core library on initialization.

**Core Library** (`bookkeeper/core.py` or similar) — All business logic lives here. Every operation that reads or writes data is a function in this module. Both the MCP server and CLI are thin wrappers that call into this library. The library enforces all validation rules (balanced entries, closed period restrictions, etc.). Returns plain Python dataclasses or typed dictionaries. Never prints output or formats for display — callers handle presentation.

**CLI** (`bookkeeper/cli.py`) — A command-line interface wrapping each core library function. Used for scripting, bulk operations, quick fixes, and working without the agent. Should use a library like `click` or `argparse`. Formats output as human-readable tables or text.

**MCP Server** (`bookkeeper/mcp_server.py`) — Exposes core library functions as MCP tools. Each tool handler parses input, calls the corresponding core function, and returns structured results. No business logic in this layer. Use the `mcp` Python SDK (FastMCP).

---

## Data Model

### `accounts` table

| Column      | Type    | Notes                                                                 |
|-------------|---------|-----------------------------------------------------------------------|
| id          | INTEGER | Primary key, auto-increment                                          |
| name        | TEXT    | Display name, unique, required                                       |
| type        | TEXT    | One of: `asset`, `liability`, `equity`, `revenue`, `expense`         |
| currency    | TEXT    | ISO 4217 code (e.g., `CAD`, `USD`). Required for asset/liability accounts. NULL for revenue/expense accounts. |
| parent_id   | INTEGER | FK to `accounts.id`, nullable. For sub-accounts (e.g., "Travel" under "Expenses"). |
| status      | TEXT    | One of: `active`, `hidden`, `dormant`. Default `active`.             |
| created_at  | TEXT    | ISO 8601 timestamp                                                   |
| updated_at  | TEXT    | ISO 8601 timestamp                                                   |

**Constraints:**
- Asset and liability accounts MUST have a currency.
- Revenue and expense accounts MUST NOT have a currency (currency is recorded per transaction line).
- Account names must be unique.
- An account with status `dormant` cannot appear in new transactions.

### `transactions` table

| Column     | Type    | Notes                                              |
|------------|---------|-----------------------------------------------------|
| id         | INTEGER | Primary key, auto-increment                         |
| date       | TEXT    | ISO 8601 date (YYYY-MM-DD)                          |
| memo       | TEXT    | Description of the transaction                      |
| currency   | TEXT    | ISO 4217 code. The currency of this transaction, determined by the asset/liability/equity accounts involved. |
| voided     | INTEGER | Boolean (0 or 1). Default 0.                        |
| void_of    | INTEGER | FK to `transactions.id`, nullable. If this transaction is a voiding reversal, points to the original. |
| created_at | TEXT    | ISO 8601 timestamp                                  |

**Constraints:**
- A voided transaction is never deleted. It stays in the ledger with `voided = 1`.
- The voiding reversal entry links back via `void_of`.

### `transaction_lines` table

| Column         | Type    | Notes                                                               |
|----------------|---------|----------------------------------------------------------------------|
| id             | INTEGER | Primary key, auto-increment                                         |
| transaction_id | INTEGER | FK to `transactions.id`, required                                   |
| account_id     | INTEGER | FK to `accounts.id`, required                                       |
| amount         | TEXT    | Stored as TEXT to avoid floating-point issues. Use Python `Decimal`. |
| direction      | TEXT    | `debit` or `credit`                                                 |
**Constraints:**
- Every transaction must have at least two lines.
- Every transaction operates in exactly one currency. This currency is determined by the asset/liability/equity accounts in the transaction — all such accounts in a single transaction must share the same currency. That shared currency is the transaction's currency.
- The sum of debits must exactly equal the sum of credits. Every transaction balances, no exceptions.
- Amount is always positive. The `direction` field determines whether it's a debit or credit.
- Revenue/expense accounts are currency-agnostic. They accept transactions in any currency. The transaction's currency is what gets recorded against them.
- Asset/liability/equity accounts are currency-specific. A transaction involving a CAD bank account is a CAD transaction.

**Multi-currency model:**

Currency is a property of the transaction, not the line. A transaction that debits an expense from a CAD bank account is a CAD transaction — the expense is recorded in CAD.

Example — buying office supplies with the CAD credit card:

```
debit   Office Supplies     $45.00 CAD
credit  Visa CAD            $45.00 CAD
```

Example — buying software with the USD credit card:

```
debit   Software Subscriptions   $29.00 USD
credit  Visa USD                 $29.00 USD
```

Both transactions hit expense accounts, but in different currencies. The expense accounts themselves have no currency — they accumulate amounts in whatever currency the transactions bring. At report time, if you want a consolidated view, the system converts using exchange rates.

**Currency transfers** (moving money between accounts in different currencies) are recorded as two separate transactions:

```
Transaction 1 (CAD): debit Currency Exchange $1,370.00 / credit RBC Chequing $1,370.00
Transaction 2 (USD): debit USD Savings $1,000.00 / credit Currency Exchange $1,000.00
```

The Currency Exchange account (an equity or asset account, owner's choice) acts as a clearing account. Any residual balance in it reflects unrealized exchange gains/losses. This keeps every transaction strictly single-currency and balanced.

### `exchange_rates` table

| Column        | Type    | Notes                                           |
|---------------|---------|--------------------------------------------------|
| id            | INTEGER | Primary key, auto-increment                      |
| base_currency | TEXT    | ISO 4217 code (e.g., `USD`)                     |
| quote_currency| TEXT    | ISO 4217 code (e.g., `CAD`)                     |
| rate          | TEXT    | Stored as TEXT, use `Decimal`. The value of 1 unit of base in quote currency. |
| effective_date| TEXT    | ISO 8601 date                                    |
| created_at    | TEXT    | ISO 8601 timestamp                               |

**Usage:** When reporting, the system looks up the most recent rate on or before the report date for each currency pair. The owner sets rates manually via `set_exchange_rate`. No live feeds.

### `periods` table

| Column     | Type    | Notes                                                |
|------------|---------|-------------------------------------------------------|
| id         | INTEGER | Primary key, auto-increment                           |
| year       | INTEGER | e.g., 2026                                           |
| month      | INTEGER | 1–12                                                 |
| status     | TEXT    | `open` or `closed`                                   |
| closed_at  | TEXT    | ISO 8601 timestamp, nullable                         |

**Constraints:**
- Transactions cannot be posted to a closed period.
- Closing a period creates a journal entry that rolls net income (revenue minus expenses) into a Retained Earnings equity account.

---

## Write Operations

### 1. `create_account`

**Parameters:**
- `name` (str, required) — must be unique
- `type` (str, required) — one of: asset, liability, equity, revenue, expense
- `currency` (str, conditional) — required for asset/liability, forbidden for revenue/expense
- `parent_id` (int, optional) — FK to an existing account
- `status` (str, optional) — default `active`

**Validation:**
- Name uniqueness
- Currency rules per account type
- Parent must exist if specified
- Parent should be the same type (don't nest an expense under an asset)

**Returns:** The created account record.

### 2. `update_account`

**Parameters:**
- `account_id` (int, required)
- `name` (str, optional) — new name, must remain unique
- `status` (str, optional) — `active`, `hidden`, or `dormant`
- `parent_id` (int, optional) — change parent, or `null` to remove

**Validation:**
- Account must exist
- Cannot change account type or currency (create a new account instead)
- If setting to `dormant`, warn if there are recent transactions (last 90 days)
- Name uniqueness if changing name

**Returns:** The updated account record.

### 3. `record_transaction`

**Parameters:**
- `date` (str, required) — ISO 8601 date
- `memo` (str, required)
- `lines` (list, required) — each line contains:
  - `account_id` (int)
  - `amount` (str or Decimal)
  - `direction` (str) — `debit` or `credit`

**Validation:**
- At least two lines
- All amounts positive
- Sum of debits must exactly equal sum of credits
- All asset/liability/equity accounts in the transaction must share the same currency — this determines the transaction's currency (stored on the transaction record)
- If the transaction contains only revenue/expense accounts (unusual but valid, e.g., reclassifying an expense), a currency must be provided explicitly
- Date must fall within an open period
- No lines may reference a `dormant` account

**Returns:** The created transaction with all lines.

### 4. `void_transaction`

**Parameters:**
- `transaction_id` (int, required)

**Behavior:**
- Marks the original transaction as `voided = 1`
- Creates a new reversing transaction: same lines but with debit/credit swapped
- The reversing entry has `void_of` pointing to the original
- Date of the reversal is today (not the original date), unless today falls in a closed period, in which case reject

**Validation:**
- Transaction must exist and not already be voided
- The current period must be open

**Returns:** Both the updated original and the new reversing transaction.

### 5. `close_period`

**Parameters:**
- `year` (int, required)
- `month` (int, required)
- `reporting_currency` (str, required) — the currency to compute net income in

**Behavior:**
- Sums all revenue and expense transactions for the given month
- Converts any foreign-currency amounts using the most recent exchange rate on or before the last day of the month
- Creates a journal entry debiting/crediting Revenue/Expense summary accounts and crediting/debiting Retained Earnings for the net income
- Marks the period as `closed`

**Validation:**
- Period must be currently `open` (or not yet exist — create it as closed)
- All prior periods should be closed (warn if not, but allow override)

**Returns:** The closing journal entry and the period record.

### 6. `set_exchange_rate`

**Parameters:**
- `base_currency` (str, required) — e.g., `USD`
- `quote_currency` (str, required) — e.g., `CAD`
- `rate` (str or Decimal, required) — "1 base = X quote"
- `effective_date` (str, required) — ISO 8601 date

**Validation:**
- Both currencies must be valid ISO 4217 codes
- Rate must be positive
- Should warn (not reject) if a rate already exists for this pair and date

**Returns:** The created exchange rate record.

---

## Read Operations

### 7. `list_accounts`

**Parameters (all optional):**
- `type` — filter by account type
- `status` — filter by status (default: return `active` and `hidden`, exclude `dormant`)
- `currency` — filter by currency

**Returns:** List of account records, including parent name if applicable.

### 8. `list_transactions`

**Parameters (all optional):**
- `account_id` — transactions involving this account
- `start_date`, `end_date` — date range filter
- `search` — substring match on memo
- `include_voided` — boolean, default false
- `limit` — max results, default 50

**Returns:** List of transactions with their lines. Most recent first.

### 9. `report_balance`

Reports on asset, liability, and equity accounts — point-in-time balances.

**Parameters:**
- `as_of_date` (str, required) — balances as of this date
- `reporting_currency` (str, optional) — if provided, convert all balances to this currency using the most recent rate on or before `as_of_date`. If omitted, report each account in its native currency.
- `detail` (bool, optional, default false) — if true, include the list of transactions contributing to each account's balance

**Behavior:**
- For each asset/liability/equity account, compute the net balance from all non-voided transaction lines up to and including `as_of_date`
- Group by account type (assets, liabilities, equity)
- If `reporting_currency` is set, convert using exchange rates and include both native and converted amounts

**Returns:** Structured report with account balances grouped by type. Includes a total for each group and a check that Assets = Liabilities + Equity (in reporting currency, if converting).

### 10. `report_income`

Reports on revenue and expense accounts — activity over a period.

**Parameters:**
- `start_date` (str, required)
- `end_date` (str, required)
- `reporting_currency` (str, optional) — if provided, convert all amounts to this currency. If omitted, group and subtotal amounts by currency as recorded.
- `detail` (bool, optional, default false) — if true, include the individual transactions

**Behavior:**
- For each revenue/expense account, sum all non-voided transaction lines within the date range
- Group by type (revenue, expenses)
- Compute net income (revenue minus expenses)
- If `reporting_currency` is NOT set, report amounts grouped by currency (e.g., "Office Supplies: $450 CAD, $120 USD") so the owner can see the raw breakdown
- If `reporting_currency` is set, convert each transaction's amount using the exchange rate effective on the transaction date (or nearest prior rate), then sum in the reporting currency

**Returns:** Structured report with revenue and expense totals by account, net income, grouped by type.

---

## CLI Design

The CLI should be usable for quick operations and scriptable for bulk tasks. Example command signatures:

```bash
# Account management
bookkeeper account create --name "RBC Chequing" --type asset --currency CAD
bookkeeper account list --type expense
bookkeeper account update --id 3 --name "Office Supplies" --status hidden

# Transactions
bookkeeper txn record --date 2026-05-01 --memo "Office supplies" \
  --debit "Office Supplies:45.00" --credit "RBC Chequing:45.00"
bookkeeper txn void --id 17
bookkeeper txn list --account "RBC Chequing" --start 2026-04-01 --end 2026-04-30

# Reports
bookkeeper report balance --as-of 2026-05-06 --currency CAD
bookkeeper report income --start 2026-04-01 --end 2026-04-30 --currency CAD --detail

# Exchange rates
bookkeeper fx set --base USD --quote CAD --rate 1.3700 --date 2026-05-06

# Period management
bookkeeper period close --year 2026 --month 4 --currency CAD
```

The CLI should also support `bookkeeper init` to create/initialize the database file, and `bookkeeper account list` should be the default if you just need a quick look.

---

## MCP Server Design

Each core library function is exposed as an MCP tool. Tool names should match the function names. Tool descriptions should be written from the perspective of instructing an LLM agent — tell it *when* to use the tool and *what* the parameters mean in accounting terms.

Example tool description for `record_transaction`:

> Record a financial transaction using double-entry bookkeeping. Every transaction must have at least two lines that balance. When the user describes spending money, receiving money, or moving money between accounts, use this tool. Debits increase asset/expense accounts; credits increase liability/equity/revenue accounts.

The MCP server should return results as structured JSON, not formatted text. Let the LLM agent format the response for the user conversationally.

### Agent System Prompt Guidance

The MCP server README or documentation should include a suggested system prompt snippet for the agent that covers:

- The chart of accounts and what each account is for
- Common transaction patterns (e.g., "paying for something with the credit card" = debit expense, credit credit card payable)
- When to use `report_balance` vs `report_income`
- That the agent should confirm transaction details before recording them
- That the agent should never guess account names — always use `list_accounts` first if unsure

---

## Technical Decisions

**Language:** Python 3.10+

**Database:** SQLite via the `sqlite3` stdlib module. Database file stored at a configurable path (default: `~/.bookkeeper/books.db`).

**Decimal handling:** All monetary amounts stored as TEXT in the database and handled as `decimal.Decimal` in Python. Never use float for money.

**Dependencies (keep minimal):**
- `click` for CLI
- `mcp` (FastMCP SDK) for MCP server
- Standard library for everything else (sqlite3, decimal, dataclasses, datetime)

**Configuration:** Database path and default reporting currency configurable via environment variables (`BOOKKEEPER_DB`, `BOOKKEEPER_CURRENCY`) or a simple config file (`~/.bookkeeper/config.toml`).

**Error handling:** The core library should raise specific exception types (e.g., `UnbalancedTransactionError`, `ClosedPeriodError`, `AccountNotFoundError`). The CLI should catch these and print human-readable messages. The MCP server should catch these and return structured error responses.

**Testing:** The core library should be testable with pytest against an in-memory SQLite database. Prioritize tests for: balanced transaction enforcement, voiding behavior, period closing, and currency conversion in reports.

---

## What Is Explicitly Out of Scope (For Now)

- Bank import / OFX / CSV parsing
- Receipt or document attachment
- Invoicing or accounts receivable tracking
- Multi-user access or authentication
- Web UI
- Automated exchange rate fetching
- Tax-specific reporting
- Audit logging beyond what the transaction ledger itself provides
- Budgeting or forecasting

These can all be added incrementally without changing the core data model.