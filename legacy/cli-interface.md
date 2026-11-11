# CLI Interface Specification
## Gillions Command-Line Interface

**Version:** 1.0  
**Last Updated:** 2026-01-24

---

## 1. Overview

The Gillions CLI provides a comprehensive command-line interface for managing personal finances using double-entry bookkeeping principles. The interface is designed to be intuitive, consistent, and powerful.

### 1.1 Design Principles

- **Consistency**: Similar commands follow similar patterns
- **Discoverability**: Comprehensive help at every level
- **Safety**: Confirmations for destructive operations
- **Clarity**: Clear, informative output with visual formatting
- **Efficiency**: Support for both interactive and scripted usage

### 1.2 Command Structure

```
gillions [GLOBAL_OPTIONS] <command> [COMMAND_OPTIONS] [ARGUMENTS]
```

---

## 2. Global Options

Available for all commands:

| Option | Short | Description |
|--------|-------|-------------|
| `--help` | `-h` | Show help message |
| `--version` | `-v` | Show version information |
| `--config PATH` | `-c` | Path to config file (default: ~/.gillions/config.yaml) |
| `--data-dir PATH` | `-d` | Path to data directory (default: ~/.gillions/data) |
| `--verbose` | | Enable verbose output |
| `--quiet` | `-q` | Suppress non-essential output |
| `--format FORMAT` | `-f` | Output format: table, json, csv (default: table) |
| `--no-color` | | Disable colored output |

---

## 3. Command Groups

### 3.1 Initialization Commands

#### `gillions init`

Initialize a new bookkeeping system.

**Usage:**
```bash
gillions init [OPTIONS]
```

**Options:**
| Option | Description | Default |
|--------|-------------|---------|
| `--currency CURRENCY` | Default currency (ISO 4217) | USD |
| `--template TEMPLATE` | Account template (personal, business, minimal) | personal |
| `--force` | Overwrite existing data | false |

**Example:**
```bash
gillions init --currency USD --template personal
```

**Output:**
```
✓ Created data directory: ~/.gillions/data
✓ Initialized database
✓ Created default chart of accounts (personal template)
✓ Created 15 default accounts

Your bookkeeping system is ready!
Run 'gillions account list' to see your accounts.
```

**Template Account Structures:**

**Personal Template:**
```
Assets
  ├── Bank
  │   ├── Checking
  │   └── Savings
  ├── Cash
  └── Investments
Liabilities
  ├── CreditCard
  └── Loans
Equity
  └── OpeningBalance
Income
  ├── Salary
  └── Other
Expenses
  ├── Food
  ├── Housing
  ├── Transportation
  ├── Utilities
  └── Personal
```

---

### 3.2 Account Management Commands

#### `gillions account list`

List all accounts in the chart of accounts.

**Usage:**
```bash
gillions account list [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--type TYPE` | Filter by account type (asset, liability, equity, income, expense) |
| `--active-only` | Show only active accounts |
| `--tree` | Display as hierarchical tree |
| `--with-balances` | Include current balances |

**Examples:**
```bash
# List all accounts
gillions account list

# List only expense accounts
gillions account list --type expense

# Show account tree with balances
gillions account list --tree --with-balances
```

**Output (table format):**
```
Code                          Name                    Type      Active  Balance
────────────────────────────────────────────────────────────────────────────────
Assets:Bank:Checking          Checking Account        Asset     Yes     $2,450.00
Assets:Bank:Savings           Savings Account         Asset     Yes     $5,000.00
Expenses:Food:Groceries       Grocery Expenses        Expense   Yes       $342.50
Income:Salary:Primary         Primary Salary          Income    Yes    -$5,000.00
```

**Output (tree format):**
```
Assets ($7,450.00)
├── Bank ($7,450.00)
│   ├── Checking ($2,450.00)
│   └── Savings ($5,000.00)
└── Cash ($0.00)

Expenses ($342.50)
└── Food ($342.50)
    └── Groceries ($342.50)
```

#### `gillions account add`

Create a new account.

**Usage:**
```bash
gillions account add <CODE> <NAME> [OPTIONS]
```

**Arguments:**
| Argument | Description |
|----------|-------------|
| `CODE` | Account code (e.g., "Assets:Bank:Checking") |
| `NAME` | Account name |

**Options:**
| Option | Description | Default |
|--------|-------------|---------|
| `--type TYPE` | Account type (required if not inferrable from code) | |
| `--description TEXT` | Account description | |
| `--currency CURRENCY` | Currency code | USD |

**Examples:**
```bash
# Add a new checking account
gillions account add "Assets:Bank:NewChecking" "New Checking Account"

# Add expense account with description
gillions account add "Expenses:Entertainment:Streaming" "Streaming Services" \
  --description "Netflix, Spotify, etc."
```

**Output:**
```
✓ Created account: Assets:Bank:NewChecking
  Name: New Checking Account
  Type: Asset
  Currency: USD
```

#### `gillions account show`

Show detailed information about an account.

**Usage:**
```bash
gillions account show <CODE> [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--with-transactions` | Include recent transactions |
| `--limit N` | Number of transactions to show (default: 10) |

**Example:**
```bash
gillions account show "Assets:Bank:Checking" --with-transactions --limit 5
```

**Output:**
```
Account: Assets:Bank:Checking
Name: Checking Account
Type: Asset
Currency: USD
Status: Active
Created: 2026-01-01

Current Balance: $2,450.00

Recent Transactions:
Date        Description              Debit      Credit     Balance
────────────────────────────────────────────────────────────────────
2026-01-25  Pay Visa bill                       $129.99    $2,450.00
2026-01-20  Transfer to savings                 $500.00    $2,579.99
2026-01-15  Grocery shopping                     $85.50    $3,079.99
```

#### `gillions account deactivate`

Deactivate an account (prevents new transactions).

**Usage:**
```bash
gillions account deactivate <CODE>
```

**Example:**
```bash
gillions account deactivate "Assets:Bank:OldChecking"
```

#### `gillions account rename`

Rename an account.

**Usage:**
```bash
gillions account rename <CODE> <NEW_NAME>
```

---

### 3.3 Transaction Commands

#### `gillions tx add`

Record a new transaction.

**Usage:**
```bash
gillions tx add [OPTIONS]
```

**Options:**
| Option | Description | Required |
|--------|-------------|----------|
| `--date DATE` | Transaction date (YYYY-MM-DD) | No (default: today) |
| `--description TEXT` | Transaction description | Yes |
| `--debit ACCOUNT AMOUNT` | Debit entry (can be repeated) | Yes |
| `--credit ACCOUNT AMOUNT` | Credit entry (can be repeated) | Yes |
| `--reference TEXT` | External reference | No |
| `--tag TAG` | Tag (can be repeated) | No |
| `--note TEXT` | Additional notes | No |

**Examples:**

**Simple expense:**
```bash
gillions tx add \
  --description "Grocery shopping" \
  --debit "Expenses:Food:Groceries" 85.50 \
  --credit "Assets:Bank:Checking" 85.50
```

**Split transaction:**
```bash
gillions tx add \
  --description "Costco shopping" \
  --debit "Expenses:Food:Groceries" 120.00 \
  --debit "Expenses:Household:Supplies" 45.00 \
  --credit "Assets:Bank:Checking" 165.00 \
  --tag groceries \
  --tag household
```

**Income:**
```bash
gillions tx add \
  --date 2026-01-31 \
  --description "Monthly salary" \
  --debit "Assets:Bank:Checking" 5000.00 \
  --credit "Income:Salary:Primary" 5000.00 \
  --reference "Payroll #2026-01"
```

**Output:**
```
✓ Transaction recorded successfully

Transaction ID: 550e8400-e29b-41d4-a716-446655440000
Date: 2026-01-15
Description: Grocery shopping

Entries:
  Debit   Expenses:Food:Groceries      $85.50
  Credit  Assets:Bank:Checking         $85.50
                                      ────────
  Total                                $85.50

Books remain balanced ✓
```

#### `gillions tx add-interactive`

Add transaction using interactive prompts.

**Usage:**
```bash
gillions tx add-interactive
```

**Interactive Flow:**
```
Transaction date [2026-01-24]: 
Description: Grocery shopping
Reference (optional): 

Entry 1:
  Account code: Expenses:Food:Groceries
  Debit or Credit [D/C]: D
  Amount: 85.50

Entry 2:
  Account code: Assets:Bank:Checking
  Debit or Credit [D/C]: C
  Amount: 85.50

Add another entry? [y/N]: n

Summary:
  Date: 2026-01-24
  Description: Grocery shopping
  Debit   Expenses:Food:Groceries  $85.50
  Credit  Assets:Bank:Checking     $85.50

Confirm and save? [Y/n]: y
✓ Transaction saved
```

#### `gillions tx list`

List transactions.

**Usage:**
```bash
gillions tx list [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--from DATE` | Start date (YYYY-MM-DD) |
| `--to DATE` | End date (YYYY-MM-DD) |
| `--account CODE` | Filter by account |
| `--tag TAG` | Filter by tag |
| `--limit N` | Maximum number of transactions (default: 50) |
| `--sort FIELD` | Sort by: date, amount, description (default: date) |
| `--reverse` | Reverse sort order |

**Examples:**
```bash
# Recent transactions
gillions tx list --limit 10

# Transactions for specific account
gillions tx list --account "Assets:Bank:Checking"

# Date range
gillions tx list --from 2026-01-01 --to 2026-01-31

# Tagged transactions
gillions tx list --tag groceries
```

**Output:**
```
ID       Date        Description              Amount     Accounts
────────────────────────────────────────────────────────────────────────
550e...  2026-01-25  Pay Visa bill            $129.99    Liabilities:CreditCard:Visa → Assets:Bank:Checking
a1b2...  2026-01-20  Transfer to savings      $500.00    Assets:Bank:Checking → Assets:Bank:Savings
c3d4...  2026-01-15  Grocery shopping          $85.50    Expenses:Food:Groceries ← Assets:Bank:Checking
```

#### `gillions tx show`

Show detailed transaction information.

**Usage:**
```bash
gillions tx show <TRANSACTION_ID>
```

**Example:**
```bash
gillions tx show 550e8400-e29b-41d4-a716-446655440000
```

**Output:**
```
Transaction: 550e8400-e29b-41d4-a716-446655440000
Date: 2026-01-15
Description: Grocery shopping
Reference: Receipt #12345
Tags: groceries, food
Created: 2026-01-15 18:30:00

Journal Entries:
  1. Debit   Expenses:Food:Groceries      $85.50
  2. Credit  Assets:Bank:Checking         $85.50
                                         ────────
     Total                                $85.50

Notes:
  Weekly grocery run
```

#### `gillions tx delete`

Delete a transaction (with confirmation).

**Usage:**
```bash
gillions tx delete <TRANSACTION_ID> [--force]
```

**Note:** In production, consider implementing "void" instead of delete to maintain audit trail.

---

### 3.4 Balance & Reporting Commands

#### `gillions balance`

Show account balances.

**Usage:**
```bash
gillions balance [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--account CODE` | Show balance for specific account |
| `--type TYPE` | Filter by account type |
| `--as-of DATE` | Balance as of date (default: today) |
| `--tree` | Show hierarchical tree with rollup balances |
| `--non-zero` | Show only accounts with non-zero balances |

**Examples:**
```bash
# All balances
gillions balance

# Specific account
gillions balance --account "Assets:Bank:Checking"

# All assets
gillions balance --type asset --tree

# Historical balance
gillions balance --as-of 2025-12-31
```

**Output:**
```
Account Balances as of 2026-01-24

Assets
  Bank
    Checking                              $2,450.00
    Savings                               $5,000.00
  Total Bank                              $7,450.00
Total Assets                              $7,450.00

Liabilities
  CreditCard
    Visa                                    $129.99
Total Liabilities                           $129.99

Equity
  OpeningBalance                          $7,000.00
Total Equity                              $7,000.00

Income
  Salary
    Primary                               $5,000.00
Total Income                              $5,000.00

Expenses
  Food
    Groceries                               $342.50
  Housing
    Rent                                  $1,500.00
Total Expenses                            $1,842.50

Net Worth: $7,320.01
```

#### `gillions report trial-balance`

Generate trial balance report.

**Usage:**
```bash
gillions report trial-balance [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--as-of DATE` | Report date (default: today) |

**Output:**
```
Trial Balance
As of: 2026-01-24

Account                              Debit        Credit
────────────────────────────────────────────────────────────
Assets:Bank:Checking              $2,450.00
Assets:Bank:Savings               $5,000.00
Liabilities:CreditCard:Visa                      $129.99
Equity:OpeningBalance                          $7,000.00
Income:Salary:Primary                          $5,000.00
Expenses:Food:Groceries             $342.50
Expenses:Housing:Rent             $1,500.00
                                 ──────────   ──────────
Total                            $9,292.50    $9,292.50

✓ Books are balanced
```

#### `gillions report income-statement`

Generate income statement (profit & loss).

**Usage:**
```bash
gillions report income-statement [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--from DATE` | Start date (required) |
| `--to DATE` | End date (default: today) |
| `--period PERIOD` | Predefined period: month, quarter, year, ytd |

**Examples:**
```bash
# Current month
gillions report income-statement --period month

# Custom date range
gillions report income-statement --from 2026-01-01 --to 2026-01-31

# Year to date
gillions report income-statement --period ytd
```

**Output:**
```
Income Statement
Period: 2026-01-01 to 2026-01-31

INCOME
  Salary
    Primary                                    $5,000.00
  Total Income                                 $5,000.00

EXPENSES
  Food
    Groceries                                    $342.50
  Housing
    Rent                                       $1,500.00
  Transportation
    Gas                                          $120.00
  Utilities
    Electric                                      $85.00
    Internet                                      $60.00
  Total Expenses                               $2,107.50

NET INCOME                                     $2,892.50
```

#### `gillions report balance-sheet`

Generate balance sheet.

**Usage:**
```bash
gillions report balance-sheet [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--as-of DATE` | Report date (default: today) |

**Output:**
```
Balance Sheet
As of: 2026-01-31

ASSETS
  Current Assets
    Bank
      Checking                               $2,450.00
      Savings                                $5,000.00
    Cash                                       $100.00
  Total Current Assets                       $7,550.00

  Investments                                $10,000.00

Total Assets                                $17,550.00

LIABILITIES
  Current Liabilities
    Credit Card
      Visa                                     $129.99
  Total Current Liabilities                    $129.99

Total Liabilities                              $129.99

EQUITY
  Opening Balance                            $7,000.00
  Net Income (YTD)                           $2,892.50
Total Equity                                 $9,892.51

Total Liabilities & Equity                  $10,022.50

Note: Assets should equal Liabilities + Equity
```

#### `gillions report register`

Account register (ledger) report.

**Usage:**
```bash
gillions report register <ACCOUNT_CODE> [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--from DATE` | Start date |
| `--to DATE` | End date |
| `--limit N` | Maximum transactions |

**Example:**
```bash
gillions report register "Assets:Bank:Checking" --from 2026-01-01
```

**Output:**
```
Account Register: Assets:Bank:Checking
Period: 2026-01-01 to 2026-01-31

Date        Description              Debit      Credit     Balance
────────────────────────────────────────────────────────────────────
2026-01-01  Opening balance       $2,500.00                $2,500.00
2026-01-15  Grocery shopping                     $85.50    $2,414.50
2026-01-20  Transfer to savings                 $500.00    $1,914.50
2026-01-25  Pay Visa bill                       $129.99    $1,784.51
2026-01-31  Monthly salary        $5,000.00                $6,784.51
                                 ──────────   ──────────
Total                             $7,500.00    $715.49

Ending Balance: $6,784.51
```

#### `gillions report summary`

Quick financial summary.

**Usage:**
```bash
gillions report summary [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--period PERIOD` | month, quarter, year, ytd (default: month) |

**Output:**
```
Financial Summary - January 2026

Net Worth:                    $7,320.01
  Assets:                     $7,450.00
  Liabilities:                 -$129.99

This Month:
  Income:                     $5,000.00
  Expenses:                   $2,107.50
  Net Income:                 $2,892.50
  Savings Rate:                   57.8%

Top Expenses:
  1. Housing:Rent              $1,500.00 (71.2%)
  2. Food:Groceries              $342.50 (16.3%)
  3. Transportation:Gas          $120.00  (5.7%)

Account Balances:
  Checking:                   $2,450.00
  Savings:                    $5,000.00
  Credit Card (Visa):          -$129.99
```

---

### 3.5 Utility Commands

#### `gillions verify`

Verify data integrity and balance.

**Usage:**
```bash
gillions verify [OPTIONS]
```

**Options:**
| Option | Description |
|--------|-------------|
| `--fix` | Attempt to fix issues (use with caution) |

**Output:**
```
Running integrity checks...

✓ All transactions balance (debits = credits)
✓ All account references valid
✓ No orphaned accounts
✓ Trial balance verified
✓ Accounting equation holds (Assets = Liabilities + Equity)

Database integrity: OK
```

#### `gillions export`

Export data to various formats.

**Usage:**
```bash
gillions export [OPTIONS] <OUTPUT_FILE>
```

**Options:**
| Option | Description |
|--------|-------------|
| `--format FORMAT` | json, csv, ledger (default: json) |
| `--from DATE` | Start date |
| `--to DATE` | End date |
| `--accounts-only` | Export only chart of accounts |
| `--transactions-only` | Export only transactions |

**Examples:**
```bash
# Export everything to JSON
gillions export --format json backup.json

# Export transactions to CSV
gillions export --format csv --transactions-only transactions.csv

# Export to Ledger format
gillions export --format ledger --from 2026-01-01 --to 2026-12-31 2026.ledger
```

#### `gillions import`

Import data from various formats.

**Usage:**
```bash
gillions import [OPTIONS] <INPUT_FILE>
```

**Options:**
| Option | Description |
|--------|-------------|
| `--format FORMAT` | json, csv (auto-detected if not specified) |
| `--dry-run` | Validate without importing |
| `--skip-duplicates` | Skip duplicate transactions |

**Example:**
```bash
gillions import --format csv --dry-run transactions.csv
```

#### `gillions config`

Manage configuration.

**Usage:**
```bash
gillions config <SUBCOMMAND>
```

**Subcommands:**
- `show` - Display current configuration
- `set <KEY> <VALUE>` - Set configuration value
- `get <KEY>` - Get configuration value
- `reset` - Reset to defaults

**Example:**
```bash
gillions config set default_currency EUR
gillions config show
```

---

## 4. Output Formats

### 4.1 Table Format (Default)

Human-readable tables with colors and formatting.

```bash
gillions balance --format table
```

### 4.2 JSON Format

Machine-readable JSON for scripting.

```bash
gillions balance --format json
```

**Output:**
```json
{
  "as_of_date": "2026-01-24",
  "accounts": [
    {
      "code": "Assets:Bank:Checking",
      "name": "Checking Account",
      "type": "Asset",
      "balance": "2450.00",
      "currency": "USD"
    }
  ],
  "summary": {
    "total_assets": "7450.00",
    "total_liabilities": "129.99",
    "net_worth": "7320.01"
  }
}
```

### 4.3 CSV Format

Spreadsheet-compatible CSV.

```bash
gillions tx list --format csv > transactions.csv
```

---

## 5. Configuration File

**Location:** `~/.gillions/config.yaml`

**Example:**
```yaml
# Default currency
default_currency: USD

# Date format for display
date_format: "%Y-%m-%d"

# Number format
decimal_places: 2
thousands_separator: ","

# Output preferences
default_output_format: table
use_color: true

# Database
database_path: ~/.gillions/data/gillions.db

# Reporting
fiscal_year_start: "01-01"  # MM-DD
```

---

## 6. Error Handling

### 6.1 Validation Errors

```
Error: Transaction does not balance
  Total debits:  $85.50
  Total credits: $80.00
  Difference:    $5.50

Please ensure debits equal credits.
```

### 6.2 Account Errors

```
Error: Account not found: "Assets:Bank:Unknown"

Did you mean one of these?
  - Assets:Bank:Checking
  - Assets:Bank:Savings

Use 'gillions account list' to see all accounts.
```

### 6.3 Date Errors

```
Error: Invalid date format: "2026-13-01"
Expected format: YYYY-MM-DD (e.g., 2026-01-15)
```

---

## 7. Shell Completion

Support for bash, zsh, and fish shell completion.

**Installation:**
```bash
# Bash
gillions --install-completion bash

# Zsh
gillions --install-completion zsh

# Fish
gillions --install-completion fish
```

**Features:**
- Command completion
- Account code completion
- Date completion
- Option completion

---

## 8. Examples & Common Workflows

### 8.1 Initial Setup

```bash
# Initialize system
gillions init --currency USD --template personal

# Add custom accounts
gillions account add "Expenses:Entertainment:Streaming" "Streaming Services"
gillions account add "Assets:Investments:Stocks" "Stock Portfolio"
```

### 8.2 Daily Usage

```bash
# Record expense
gillions tx add \
  --description "Coffee" \
  --debit "Expenses:Food:Dining" 5.50 \
  --credit "Assets:Cash" 5.50

# Check balance
gillions balance --account "Assets:Bank:Checking"

# View recent transactions
gillions tx list --limit 10
```

### 8.3 Monthly Review

```bash
# Generate monthly reports
gillions report income-statement --period month
gillions report summary --period month

# Check specific categories
gillions balance --type expense --non-zero

# Export for backup
gillions export --format json monthly-backup-$(date +%Y-%m).json
```

### 8.4 Year-End Review

```bash
# Annual income statement
gillions report income-statement --from 2026-01-01 --to 2026-12-31

# Balance sheet
gillions report balance-sheet --as-of 2026-12-31

# Verify integrity
gillions verify

# Export annual data
gillions export --format json --from 2026-01-01 --to 2026-12-31 2026-annual.json
```

---

## 9. Future CLI Enhancements

### 9.1 Planned Features

- **Budget commands**: `gillions budget create`, `gillions budget report`
- **Reconciliation**: `gillions reconcile <account>`
- **Recurring transactions**: `gillions recurring add`, `gillions recurring process`
- **Search**: `gillions search <query>`
- **Statistics**: `gillions stats --period month`
- **Visualization**: ASCII charts for spending trends

### 9.2 Interactive Mode

```bash
gillions shell
```

Launches interactive REPL for faster command execution.

---

## 10. Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 3 | Validation error (transaction doesn't balance, etc.) |
| 4 | Database error |
| 5 | Account not found |
| 6 | Transaction not found |
| 10 | Not initialized (run `gillions init` first) |
