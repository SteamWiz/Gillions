# Domain Model Specification
## Gillions Double-Entry Bookkeeping System

**Version:** 1.0  
**Last Updated:** 2026-01-24

---

## 1. Core Domain Entities

### 1.1 Account

The fundamental entity representing a financial account in the chart of accounts.

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | UUID | Yes | Unique identifier |
| `code` | String | Yes | Hierarchical account code (e.g., "Assets:Bank:Checking") |
| `name` | String | Yes | Human-readable account name |
| `account_type` | AccountType | Yes | Type of account (Asset, Liability, etc.) |
| `description` | String | No | Optional detailed description |
| `currency` | String | Yes | ISO 4217 currency code (default: "USD") |
| `parent_id` | UUID | No | Parent account for hierarchy |
| `is_active` | Boolean | Yes | Whether account accepts new transactions |
| `created_at` | DateTime | Yes | Account creation timestamp |
| `updated_at` | DateTime | Yes | Last modification timestamp |

#### Account Code Structure

Accounts use a hierarchical colon-separated code system:

```
<Type>:<Category>:<Subcategory>:<Detail>

Examples:
- Assets:Bank:Checking
- Assets:Bank:Savings
- Expenses:Food:Groceries
- Expenses:Food:Dining
- Income:Salary:Primary
- Liabilities:CreditCard:Visa
```

**Rules:**
- Minimum 1 level, maximum 5 levels
- Each level: alphanumeric, hyphens, underscores only
- Case-insensitive but stored in title case
- Must start with valid account type

#### Account Types

```python
class AccountType(Enum):
    ASSET = "Asset"
    LIABILITY = "Liability"
    EQUITY = "Equity"
    INCOME = "Income"
    EXPENSE = "Expense"
```

**Normal Balance:**
- Assets: Debit
- Liabilities: Credit
- Equity: Credit
- Income: Credit
- Expenses: Debit

#### Business Rules

1. Account codes must be unique within the chart of accounts
2. Parent account must exist before creating child accounts
3. Cannot delete accounts with transactions
4. Cannot change account type after creation
5. Inactive accounts cannot receive new transactions but retain history

#### Example

```python
Account(
    id=UUID("..."),
    code="Assets:Bank:Checking",
    name="Primary Checking Account",
    account_type=AccountType.ASSET,
    description="Main checking account at Example Bank",
    currency="USD",
    parent_id=UUID("..."),  # Points to "Assets:Bank"
    is_active=True,
    created_at=datetime(2026, 1, 1),
    updated_at=datetime(2026, 1, 1)
)
```

---

### 1.2 Transaction

Represents a financial transaction consisting of multiple journal entries that must balance.

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | UUID | Yes | Unique identifier |
| `date` | Date | Yes | Transaction date |
| `description` | String | Yes | Transaction description |
| `reference` | String | No | External reference (check number, invoice, etc.) |
| `entries` | List[JournalEntry] | Yes | Journal entries (minimum 2) |
| `tags` | List[String] | No | Optional tags for categorization |
| `notes` | String | No | Additional notes |
| `created_at` | DateTime | Yes | Record creation timestamp |
| `created_by` | String | No | User who created (for future multi-user) |

#### Business Rules

1. Must have at least 2 journal entries
2. Total debits must equal total credits
3. All entries must reference valid, active accounts
4. Transaction date cannot be in the future
5. Once created, transactions are immutable (can only be reversed)
6. All amounts must be positive (direction indicated by debit/credit)

#### Example

```python
Transaction(
    id=UUID("..."),
    date=date(2026, 1, 15),
    description="Grocery shopping at Whole Foods",
    reference="Receipt #12345",
    entries=[
        JournalEntry(...),  # Debit: Expenses:Food:Groceries $85.50
        JournalEntry(...)   # Credit: Assets:Bank:Checking $85.50
    ],
    tags=["groceries", "food"],
    notes="Weekly grocery run",
    created_at=datetime(2026, 1, 15, 18, 30),
    created_by="user"
)
```

---

### 1.3 JournalEntry

Represents a single line in a transaction (either debit or credit to an account).

#### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `id` | UUID | Yes | Unique identifier |
| `transaction_id` | UUID | Yes | Parent transaction |
| `account_id` | UUID | Yes | Account being debited/credited |
| `entry_type` | EntryType | Yes | DEBIT or CREDIT |
| `amount` | Decimal | Yes | Amount (always positive) |
| `currency` | String | Yes | ISO 4217 currency code |
| `sequence` | Integer | Yes | Order within transaction (for display) |

#### Entry Type

```python
class EntryType(Enum):
    DEBIT = "Debit"
    CREDIT = "Credit"
```

#### Business Rules

1. Amount must be positive (> 0)
2. Amount precision: 2 decimal places for most currencies
3. Account must exist and be active
4. Currency must match account currency
5. Cannot modify after transaction is saved

#### Example

```python
JournalEntry(
    id=UUID("..."),
    transaction_id=UUID("..."),
    account_id=UUID("..."),  # Assets:Bank:Checking
    entry_type=EntryType.CREDIT,
    amount=Decimal("85.50"),
    currency="USD",
    sequence=2
)
```

---

### 1.4 ChartOfAccounts

Container for managing the complete account hierarchy.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `accounts` | Dict[String, Account] | All accounts indexed by code |
| `root_accounts` | List[Account] | Top-level accounts (no parent) |

#### Methods

- `add_account(account: Account) -> None`
- `get_account(code: String) -> Account`
- `get_children(parent_code: String) -> List[Account]`
- `get_hierarchy() -> Dict`
- `validate_hierarchy() -> bool`

#### Business Rules

1. Each account type must have at least one root account
2. Hierarchy depth limited to 5 levels
3. No circular references in parent-child relationships

---

### 1.5 Balance

Represents the calculated balance of an account at a point in time.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `account_id` | UUID | Account identifier |
| `account_code` | String | Account code |
| `account_name` | String | Account name |
| `account_type` | AccountType | Account type |
| `debit_total` | Decimal | Sum of all debits |
| `credit_total` | Decimal | Sum of all credits |
| `balance` | Decimal | Net balance |
| `as_of_date` | Date | Balance calculation date |

#### Calculation Rules

**Balance Calculation by Account Type:**

```python
# Assets and Expenses (normal debit balance)
balance = debit_total - credit_total

# Liabilities, Equity, and Income (normal credit balance)
balance = credit_total - debit_total
```

**Display Convention:**
- Positive balance = normal balance
- Negative balance = abnormal balance (shown in red/parentheses)

---

## 2. Value Objects

### 2.1 Money

Immutable value object representing a monetary amount.

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str  # ISO 4217 code
    
    def __post_init__(self):
        # Validate amount precision
        # Validate currency code
        pass
    
    def __add__(self, other: Money) -> Money:
        # Must be same currency
        pass
    
    def __sub__(self, other: Money) -> Money:
        pass
```

### 2.2 DateRange

Value object for date range queries.

```python
@dataclass(frozen=True)
class DateRange:
    start_date: date
    end_date: date
    
    def contains(self, date: date) -> bool:
        pass
    
    def overlaps(self, other: DateRange) -> bool:
        pass
```

---

## 3. Domain Services

### 3.1 TransactionValidator

Validates transaction business rules.

#### Methods

```python
class TransactionValidator:
    def validate_balance(self, transaction: Transaction) -> ValidationResult:
        """Ensure debits equal credits"""
        pass
    
    def validate_accounts(self, transaction: Transaction) -> ValidationResult:
        """Ensure all accounts exist and are active"""
        pass
    
    def validate_amounts(self, transaction: Transaction) -> ValidationResult:
        """Ensure amounts are positive and properly formatted"""
        pass
    
    def validate_date(self, transaction: Transaction) -> ValidationResult:
        """Ensure date is valid"""
        pass
```

### 3.2 BalanceCalculator

Calculates account balances.

#### Methods

```python
class BalanceCalculator:
    def calculate_balance(
        self, 
        account: Account, 
        as_of_date: date = None
    ) -> Balance:
        """Calculate balance for single account"""
        pass
    
    def calculate_trial_balance(
        self, 
        as_of_date: date = None
    ) -> List[Balance]:
        """Calculate balances for all accounts"""
        pass
    
    def verify_books_balance(self) -> bool:
        """Verify total debits = total credits"""
        pass
```

### 3.3 AccountHierarchyManager

Manages account hierarchy operations.

#### Methods

```python
class AccountHierarchyManager:
    def get_account_path(self, account: Account) -> List[Account]:
        """Get full path from root to account"""
        pass
    
    def get_descendants(self, account: Account) -> List[Account]:
        """Get all child accounts recursively"""
        pass
    
    def calculate_rollup_balance(
        self, 
        parent: Account, 
        as_of_date: date = None
    ) -> Balance:
        """Calculate balance including all descendants"""
        pass
```

---

## 4. Common Transaction Patterns

### 4.1 Income Receipt

Recording salary or other income.

```
Transaction: "Monthly salary"
Date: 2026-01-31
Entries:
  - Debit:  Assets:Bank:Checking        $5,000.00
  - Credit: Income:Salary:Primary       $5,000.00
```

### 4.2 Expense Payment

Paying for goods or services.

```
Transaction: "Grocery shopping"
Date: 2026-01-15
Entries:
  - Debit:  Expenses:Food:Groceries     $85.50
  - Credit: Assets:Bank:Checking        $85.50
```

### 4.3 Transfer Between Accounts

Moving money between your own accounts.

```
Transaction: "Transfer to savings"
Date: 2026-01-20
Entries:
  - Debit:  Assets:Bank:Savings         $500.00
  - Credit: Assets:Bank:Checking        $500.00
```

### 4.4 Credit Card Purchase

Purchasing on credit.

```
Transaction: "Amazon purchase"
Date: 2026-01-10
Entries:
  - Debit:  Expenses:Shopping:Online    $129.99
  - Credit: Liabilities:CreditCard:Visa $129.99
```

### 4.5 Credit Card Payment

Paying off credit card balance.

```
Transaction: "Pay Visa bill"
Date: 2026-01-25
Entries:
  - Debit:  Liabilities:CreditCard:Visa $129.99
  - Credit: Assets:Bank:Checking        $129.99
```

### 4.6 Split Transaction

Single transaction affecting multiple expense categories.

```
Transaction: "Costco shopping"
Date: 2026-01-18
Entries:
  - Debit:  Expenses:Food:Groceries     $120.00
  - Debit:  Expenses:Household:Supplies $45.00
  - Debit:  Expenses:Personal:Clothing  $35.00
  - Credit: Assets:Bank:Checking        $200.00
```

### 4.7 Opening Balance

Initial account setup with equity.

```
Transaction: "Opening balance - Checking"
Date: 2026-01-01
Entries:
  - Debit:  Assets:Bank:Checking        $2,500.00
  - Credit: Equity:OpeningBalance       $2,500.00
```

---

## 5. Domain Invariants

### 5.1 The Accounting Equation

Must always hold true:

```
Assets = Liabilities + Equity
```

### 5.2 Transaction Balance

For every transaction:

```
Sum(Debits) = Sum(Credits)
```

### 5.3 Trial Balance

At any point in time:

```
Sum(All Debit Balances) = Sum(All Credit Balances)
```

### 5.4 Account Hierarchy

- No orphaned accounts (except root accounts)
- No circular references
- Maximum depth of 5 levels

---

## 6. Domain Events

Future extensibility for event-driven features.

### 6.1 Potential Events

```python
class TransactionCreated(DomainEvent):
    transaction_id: UUID
    date: date
    amount: Decimal

class AccountCreated(DomainEvent):
    account_id: UUID
    code: str
    account_type: AccountType

class BalanceThresholdExceeded(DomainEvent):
    account_id: UUID
    threshold: Decimal
    current_balance: Decimal
```

---

## 7. Validation Rules Summary

### Account Validation
- ✓ Unique account code
- ✓ Valid account type
- ✓ Valid parent reference
- ✓ Code format compliance
- ✓ Currency code validation

### Transaction Validation
- ✓ Minimum 2 entries
- ✓ Debits equal credits
- ✓ All accounts exist and active
- ✓ Valid date (not future)
- ✓ Positive amounts
- ✓ Currency consistency

### Journal Entry Validation
- ✓ Positive amount
- ✓ Valid account reference
- ✓ Currency matches account
- ✓ Proper decimal precision

---

## 8. Data Integrity Constraints

### Database Level
- Primary keys on all entities
- Foreign key constraints
- Unique constraints on account codes
- Check constraints on amounts (> 0)
- Not null constraints on required fields

### Application Level
- Transaction atomicity (all entries or none)
- Balance verification before commit
- Immutability of posted transactions
- Audit trail preservation

---

## 9. Example Domain Model Usage

### Creating a Transaction

```python
# Create accounts
checking = Account(
    code="Assets:Bank:Checking",
    name="Checking Account",
    account_type=AccountType.ASSET,
    currency="USD"
)

groceries = Account(
    code="Expenses:Food:Groceries",
    name="Grocery Expenses",
    account_type=AccountType.EXPENSE,
    currency="USD"
)

# Create transaction
transaction = Transaction(
    date=date.today(),
    description="Weekly groceries",
    entries=[
        JournalEntry(
            account_id=groceries.id,
            entry_type=EntryType.DEBIT,
            amount=Decimal("85.50"),
            currency="USD",
            sequence=1
        ),
        JournalEntry(
            account_id=checking.id,
            entry_type=EntryType.CREDIT,
            amount=Decimal("85.50"),
            currency="USD",
            sequence=2
        )
    ]
)

# Validate
validator = TransactionValidator()
result = validator.validate_balance(transaction)
if result.is_valid:
    repository.save(transaction)
```

### Calculating Balance

```python
calculator = BalanceCalculator()

# Single account balance
balance = calculator.calculate_balance(
    account=checking,
    as_of_date=date(2026, 1, 31)
)

print(f"{balance.account_name}: ${balance.balance}")

# Trial balance (all accounts)
trial_balance = calculator.calculate_trial_balance(
    as_of_date=date(2026, 1, 31)
)

for balance in trial_balance:
    print(f"{balance.account_code}: ${balance.balance}")
```

---

## 10. Future Enhancements

### Multi-Currency Support
- Exchange rates
- Currency conversion
- Realized/unrealized gains

### Budgeting
- Budget entities
- Budget vs. actual comparison
- Variance analysis

### Reconciliation
- Cleared/uncleared status
- Bank statement matching
- Reconciliation reports

### Recurring Transactions
- Template transactions
- Scheduled posting
- Automatic generation
