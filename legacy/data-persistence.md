# Data Persistence Specification
## Gillions Database Schema & Storage Strategy

**Version:** 1.0  
**Last Updated:** 2026-01-24

---

## 1. Overview

### 1.1 Storage Technology

**Primary Storage:** SQLite 3.35+

**Rationale:**
- **ACID Compliance**: Ensures data integrity for financial transactions
- **Zero Configuration**: No server setup required
- **Single File**: Easy backup and portability
- **Cross-Platform**: Works on all major operating systems
- **Performance**: Sufficient for personal finance scale (thousands of transactions)
- **Reliability**: Battle-tested, widely used database engine
- **SQL Support**: Rich query capabilities for reporting

### 1.2 Database Location

**Default Path:** `~/.gillions/data/gillions.db`

**Configurable via:**
- Command-line option: `--data-dir`
- Configuration file: `config.yaml`
- Environment variable: `GILLIONS_DATA_DIR`

### 1.3 File Structure

```
~/.gillions/
├── data/
│   ├── gillions.db          # Main SQLite database
│   └── gillions.db-wal      # Write-Ahead Log (WAL mode)
├── config.yaml              # User configuration
├── backups/                 # Automatic backups
│   ├── gillions-2026-01-24.db
│   └── gillions-2026-01-23.db
└── exports/                 # Export files
    └── 2026-01-monthly.json
```

---

## 2. Database Schema

### 2.1 Schema Version Management

**Table:** `schema_version`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `version` | INTEGER | PRIMARY KEY | Schema version number |
| `applied_at` | TIMESTAMP | NOT NULL | When migration was applied |
| `description` | TEXT | NOT NULL | Migration description |

**Purpose:** Track schema migrations for upgrades.

---

### 2.2 Accounts Table

**Table:** `accounts`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | UUID as text |
| `code` | TEXT | UNIQUE, NOT NULL | Hierarchical account code |
| `name` | TEXT | NOT NULL | Account name |
| `account_type` | TEXT | NOT NULL | Asset, Liability, Equity, Income, Expense |
| `description` | TEXT | NULL | Optional description |
| `currency` | TEXT | NOT NULL, DEFAULT 'USD' | ISO 4217 currency code |
| `parent_id` | TEXT | NULL, FOREIGN KEY | Parent account UUID |
| `is_active` | INTEGER | NOT NULL, DEFAULT 1 | Boolean: 1=active, 0=inactive |
| `created_at` | TIMESTAMP | NOT NULL | Creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL | Last update timestamp |

**Indexes:**
```sql
CREATE UNIQUE INDEX idx_accounts_code ON accounts(code);
CREATE INDEX idx_accounts_type ON accounts(account_type);
CREATE INDEX idx_accounts_parent ON accounts(parent_id);
CREATE INDEX idx_accounts_active ON accounts(is_active);
```

**Constraints:**
```sql
-- Account type must be valid
CHECK (account_type IN ('Asset', 'Liability', 'Equity', 'Income', 'Expense'))

-- Currency must be 3-character code
CHECK (length(currency) = 3)

-- is_active is boolean
CHECK (is_active IN (0, 1))
```

**Example Data:**
```sql
INSERT INTO accounts VALUES (
    '550e8400-e29b-41d4-a716-446655440000',
    'Assets:Bank:Checking',
    'Checking Account',
    'Asset',
    'Primary checking account',
    'USD',
    '550e8400-e29b-41d4-a716-446655440001',  -- Parent: Assets:Bank
    1,
    '2026-01-01 00:00:00',
    '2026-01-01 00:00:00'
);
```

---

### 2.3 Transactions Table

**Table:** `transactions`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | UUID as text |
| `date` | DATE | NOT NULL | Transaction date (YYYY-MM-DD) |
| `description` | TEXT | NOT NULL | Transaction description |
| `reference` | TEXT | NULL | External reference |
| `notes` | TEXT | NULL | Additional notes |
| `created_at` | TIMESTAMP | NOT NULL | Record creation timestamp |
| `created_by` | TEXT | NULL | User identifier (future use) |

**Indexes:**
```sql
CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_description ON transactions(description);
```

**Example Data:**
```sql
INSERT INTO transactions VALUES (
    'a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d',
    '2026-01-15',
    'Grocery shopping at Whole Foods',
    'Receipt #12345',
    'Weekly grocery run',
    '2026-01-15 18:30:00',
    'user'
);
```

---

### 2.4 Journal Entries Table

**Table:** `journal_entries`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | TEXT | PRIMARY KEY | UUID as text |
| `transaction_id` | TEXT | NOT NULL, FOREIGN KEY | Parent transaction |
| `account_id` | TEXT | NOT NULL, FOREIGN KEY | Account being debited/credited |
| `entry_type` | TEXT | NOT NULL | 'Debit' or 'Credit' |
| `amount` | DECIMAL(19,4) | NOT NULL | Amount (always positive) |
| `currency` | TEXT | NOT NULL | ISO 4217 currency code |
| `sequence` | INTEGER | NOT NULL | Display order within transaction |

**Indexes:**
```sql
CREATE INDEX idx_entries_transaction ON journal_entries(transaction_id);
CREATE INDEX idx_entries_account ON journal_entries(account_id);
CREATE INDEX idx_entries_type ON journal_entries(entry_type);
```

**Constraints:**
```sql
-- Entry type must be valid
CHECK (entry_type IN ('Debit', 'Credit'))

-- Amount must be positive
CHECK (amount > 0)

-- Currency must be 3-character code
CHECK (length(currency) = 3)

-- Foreign key to transactions
FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE

-- Foreign key to accounts
FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT
```

**Example Data:**
```sql
INSERT INTO journal_entries VALUES (
    'e1f2a3b4-c5d6-4e7f-8a9b-0c1d2e3f4a5b',
    'a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d',  -- transaction_id
    '550e8400-e29b-41d4-a716-446655440000',  -- account_id (Checking)
    'Credit',
    85.50,
    'USD',
    2
);
```

---

### 2.5 Transaction Tags Table

**Table:** `transaction_tags`

Many-to-many relationship between transactions and tags.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `transaction_id` | TEXT | NOT NULL, FOREIGN KEY | Transaction UUID |
| `tag` | TEXT | NOT NULL | Tag name |

**Primary Key:** `(transaction_id, tag)`

**Indexes:**
```sql
CREATE INDEX idx_tags_transaction ON transaction_tags(transaction_id);
CREATE INDEX idx_tags_tag ON transaction_tags(tag);
```

**Constraints:**
```sql
FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE
```

**Example Data:**
```sql
INSERT INTO transaction_tags VALUES (
    'a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d',
    'groceries'
);
INSERT INTO transaction_tags VALUES (
    'a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d',
    'food'
);
```

---

### 2.6 Configuration Table

**Table:** `config`

Key-value store for application configuration.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `key` | TEXT | PRIMARY KEY | Configuration key |
| `value` | TEXT | NOT NULL | Configuration value (JSON) |
| `updated_at` | TIMESTAMP | NOT NULL | Last update timestamp |

**Example Data:**
```sql
INSERT INTO config VALUES (
    'default_currency',
    '"USD"',
    '2026-01-01 00:00:00'
);
INSERT INTO config VALUES (
    'fiscal_year_start',
    '"01-01"',
    '2026-01-01 00:00:00'
);
```

---

## 3. Database Initialization

### 3.1 Schema Creation Script

```sql
-- Enable foreign keys
PRAGMA foreign_keys = ON;

-- Enable WAL mode for better concurrency
PRAGMA journal_mode = WAL;

-- Schema version tracking
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    applied_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    description TEXT NOT NULL
);

INSERT INTO schema_version VALUES (1, CURRENT_TIMESTAMP, 'Initial schema');

-- Accounts table
CREATE TABLE accounts (
    id TEXT PRIMARY KEY,
    code TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    account_type TEXT NOT NULL CHECK (account_type IN ('Asset', 'Liability', 'Equity', 'Income', 'Expense')),
    description TEXT,
    currency TEXT NOT NULL DEFAULT 'USD' CHECK (length(currency) = 3),
    parent_id TEXT,
    is_active INTEGER NOT NULL DEFAULT 1 CHECK (is_active IN (0, 1)),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (parent_id) REFERENCES accounts(id) ON DELETE RESTRICT
);

CREATE UNIQUE INDEX idx_accounts_code ON accounts(code);
CREATE INDEX idx_accounts_type ON accounts(account_type);
CREATE INDEX idx_accounts_parent ON accounts(parent_id);
CREATE INDEX idx_accounts_active ON accounts(is_active);

-- Transactions table
CREATE TABLE transactions (
    id TEXT PRIMARY KEY,
    date DATE NOT NULL,
    description TEXT NOT NULL,
    reference TEXT,
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by TEXT
);

CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_description ON transactions(description);

-- Journal entries table
CREATE TABLE journal_entries (
    id TEXT PRIMARY KEY,
    transaction_id TEXT NOT NULL,
    account_id TEXT NOT NULL,
    entry_type TEXT NOT NULL CHECK (entry_type IN ('Debit', 'Credit')),
    amount DECIMAL(19,4) NOT NULL CHECK (amount > 0),
    currency TEXT NOT NULL CHECK (length(currency) = 3),
    sequence INTEGER NOT NULL,
    FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE,
    FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT
);

CREATE INDEX idx_entries_transaction ON journal_entries(transaction_id);
CREATE INDEX idx_entries_account ON journal_entries(account_id);
CREATE INDEX idx_entries_type ON journal_entries(entry_type);

-- Transaction tags table
CREATE TABLE transaction_tags (
    transaction_id TEXT NOT NULL,
    tag TEXT NOT NULL,
    PRIMARY KEY (transaction_id, tag),
    FOREIGN KEY (transaction_id) REFERENCES transactions(id) ON DELETE CASCADE
);

CREATE INDEX idx_tags_transaction ON transaction_tags(transaction_id);
CREATE INDEX idx_tags_tag ON transaction_tags(tag);

-- Configuration table
CREATE TABLE config (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

## 4. SQLAlchemy ORM Models

### 4.1 Account Model

```python
from sqlalchemy import Column, String, Integer, DateTime, ForeignKey, CheckConstraint
from sqlalchemy.orm import relationship
from datetime import datetime
import uuid

class Account(Base):
    __tablename__ = 'accounts'
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    code = Column(String, unique=True, nullable=False)
    name = Column(String, nullable=False)
    account_type = Column(String, nullable=False)
    description = Column(String)
    currency = Column(String, nullable=False, default='USD')
    parent_id = Column(String, ForeignKey('accounts.id'))
    is_active = Column(Integer, nullable=False, default=1)
    created_at = Column(DateTime, nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime, nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    parent = relationship('Account', remote_side=[id], backref='children')
    journal_entries = relationship('JournalEntry', back_populates='account')
    
    # Constraints
    __table_args__ = (
        CheckConstraint("account_type IN ('Asset', 'Liability', 'Equity', 'Income', 'Expense')"),
        CheckConstraint("length(currency) = 3"),
        CheckConstraint("is_active IN (0, 1)"),
    )
```

### 4.2 Transaction Model

```python
class Transaction(Base):
    __tablename__ = 'transactions'
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    date = Column(Date, nullable=False)
    description = Column(String, nullable=False)
    reference = Column(String)
    notes = Column(String)
    created_at = Column(DateTime, nullable=False, default=datetime.utcnow)
    created_by = Column(String)
    
    # Relationships
    entries = relationship('JournalEntry', back_populates='transaction', cascade='all, delete-orphan')
    tags = relationship('TransactionTag', back_populates='transaction', cascade='all, delete-orphan')
```

### 4.3 JournalEntry Model

```python
from decimal import Decimal

class JournalEntry(Base):
    __tablename__ = 'journal_entries'
    
    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    transaction_id = Column(String, ForeignKey('transactions.id'), nullable=False)
    account_id = Column(String, ForeignKey('accounts.id'), nullable=False)
    entry_type = Column(String, nullable=False)
    amount = Column(Numeric(19, 4), nullable=False)
    currency = Column(String, nullable=False)
    sequence = Column(Integer, nullable=False)
    
    # Relationships
    transaction = relationship('Transaction', back_populates='entries')
    account = relationship('Account', back_populates='journal_entries')
    
    # Constraints
    __table_args__ = (
        CheckConstraint("entry_type IN ('Debit', 'Credit')"),
        CheckConstraint("amount > 0"),
        CheckConstraint("length(currency) = 3"),
    )
```

---

## 5. Common Queries

### 5.1 Account Balance Calculation

```sql
-- Calculate balance for a specific account
SELECT 
    a.id,
    a.code,
    a.name,
    a.account_type,
    COALESCE(SUM(CASE WHEN je.entry_type = 'Debit' THEN je.amount ELSE 0 END), 0) as total_debits,
    COALESCE(SUM(CASE WHEN je.entry_type = 'Credit' THEN je.amount ELSE 0 END), 0) as total_credits,
    CASE 
        WHEN a.account_type IN ('Asset', 'Expense') THEN
            COALESCE(SUM(CASE WHEN je.entry_type = 'Debit' THEN je.amount ELSE -je.amount END), 0)
        ELSE
            COALESCE(SUM(CASE WHEN je.entry_type = 'Credit' THEN je.amount ELSE -je.amount END), 0)
    END as balance
FROM accounts a
LEFT JOIN journal_entries je ON a.id = je.account_id
LEFT JOIN transactions t ON je.transaction_id = t.id
WHERE a.code = 'Assets:Bank:Checking'
    AND (t.date <= '2026-01-31' OR t.date IS NULL)
GROUP BY a.id, a.code, a.name, a.account_type;
```

### 5.2 Trial Balance

```sql
-- Generate trial balance for all accounts
SELECT 
    a.code,
    a.name,
    a.account_type,
    COALESCE(SUM(CASE WHEN je.entry_type = 'Debit' THEN je.amount ELSE 0 END), 0) as debit_total,
    COALESCE(SUM(CASE WHEN je.entry_type = 'Credit' THEN je.amount ELSE 0 END), 0) as credit_total
FROM accounts a
LEFT JOIN journal_entries je ON a.id = je.account_id
LEFT JOIN transactions t ON je.transaction_id = t.id
WHERE a.is_active = 1
    AND (t.date <= '2026-01-31' OR t.date IS NULL)
GROUP BY a.id, a.code, a.name, a.account_type
HAVING debit_total != 0 OR credit_total != 0
ORDER BY a.code;
```

### 5.3 Account Register (Ledger)

```sql
-- Get all transactions for an account with running balance
WITH account_transactions AS (
    SELECT 
        t.date,
        t.description,
        je.entry_type,
        je.amount,
        t.id as transaction_id
    FROM journal_entries je
    JOIN transactions t ON je.transaction_id = t.id
    JOIN accounts a ON je.account_id = a.id
    WHERE a.code = 'Assets:Bank:Checking'
        AND t.date BETWEEN '2026-01-01' AND '2026-01-31'
    ORDER BY t.date, t.created_at
)
SELECT 
    date,
    description,
    CASE WHEN entry_type = 'Debit' THEN amount ELSE 0 END as debit,
    CASE WHEN entry_type = 'Credit' THEN amount ELSE 0 END as credit,
    SUM(CASE WHEN entry_type = 'Debit' THEN amount ELSE -amount END) 
        OVER (ORDER BY date, transaction_id) as balance
FROM account_transactions
ORDER BY date, transaction_id;
```

### 5.4 Income Statement Data

```sql
-- Get income and expense totals for a period
SELECT 
    a.account_type,
    a.code,
    a.name,
    SUM(CASE WHEN je.entry_type = 'Debit' THEN je.amount ELSE -je.amount END) as amount
FROM accounts a
JOIN journal_entries je ON a.id = je.account_id
JOIN transactions t ON je.transaction_id = t.id
WHERE a.account_type IN ('Income', 'Expense')
    AND t.date BETWEEN '2026-01-01' AND '2026-01-31'
GROUP BY a.account_type, a.code, a.name
ORDER BY a.account_type, a.code;
```

### 5.5 Transaction Validation

```sql
-- Verify all transactions balance (debits = credits)
SELECT 
    t.id,
    t.date,
    t.description,
    SUM(CASE WHEN je.entry_type = 'Debit' THEN je.amount ELSE 0 END) as total_debits,
    SUM(CASE WHEN je.entry_type = 'Credit' THEN je.amount ELSE 0 END) as total_credits,
    ABS(SUM(CASE WHEN je.entry_type = 'Debit' THEN je.amount ELSE -je.amount END)) as difference
FROM transactions t
JOIN journal_entries je ON t.id = je.transaction_id
GROUP BY t.id, t.date, t.description
HAVING difference > 0.001  -- Allow for tiny rounding errors
ORDER BY t.date;
```

---

## 6. Data Integrity & Validation

### 6.1 Database-Level Integrity

**Foreign Key Constraints:**
- Enabled via `PRAGMA foreign_keys = ON`
- Prevent orphaned records
- Cascade deletes for dependent data

**Check Constraints:**
- Account types must be valid
- Amounts must be positive
- Entry types must be Debit or Credit
- Currency codes must be 3 characters

**Unique Constraints:**
- Account codes must be unique
- Prevents duplicate accounts

### 6.2 Application-Level Validation

**Before Insert/Update:**
1. Validate transaction balances (debits = credits)
2. Verify all accounts exist and are active
3. Check date validity (not in future)
4. Validate amount precision
5. Verify currency consistency

**Triggers (Optional):**
```sql
-- Update updated_at timestamp automatically
CREATE TRIGGER update_account_timestamp 
AFTER UPDATE ON accounts
BEGIN
    UPDATE accounts SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;
```

---

## 7. Backup & Recovery

### 7.1 Backup Strategy

**Automatic Backups:**
- Daily backup before first transaction
- Stored in `~/.gillions/backups/`
- Retention: Keep last 30 days

**Manual Backup:**
```bash
gillions export --format json backup.json
```

**Database Backup:**
```bash
# SQLite backup command
sqlite3 ~/.gillions/data/gillions.db ".backup ~/.gillions/backups/manual-backup.db"
```

### 7.2 Recovery

**From JSON Export:**
```bash
gillions import backup.json
```

**From Database Backup:**
```bash
cp ~/.gillions/backups/gillions-2026-01-24.db ~/.gillions/data/gillions.db
```

---

## 8. Performance Optimization

### 8.1 Indexes

All critical query paths are indexed:
- Account lookups by code
- Transaction lookups by date
- Journal entry lookups by account and transaction
- Tag lookups

### 8.2 Query Optimization

**Use Prepared Statements:**
- Prevent SQL injection
- Improve query performance
- Handled automatically by SQLAlchemy

**Batch Operations:**
- Insert multiple transactions in single transaction
- Use bulk insert for initial data load

**Connection Pooling:**
- SQLAlchemy manages connection pool
- Reuse connections for better performance

### 8.3 WAL Mode

Write-Ahead Logging (WAL) mode enabled:
- Better concurrency
- Faster writes
- Atomic commits

```sql
PRAGMA journal_mode = WAL;
```

---

## 9. Migration Strategy

### 9.1 Schema Versioning

Track schema version in `schema_version` table.

**Current Version:** 1

### 9.2 Migration Process

**Using Alembic (SQLAlchemy migration tool):**

```python
# alembic/versions/001_initial_schema.py
def upgrade():
    # Create tables
    pass

def downgrade():
    # Drop tables
    pass
```

**Migration Commands:**
```bash
# Generate migration
alembic revision --autogenerate -m "Add new column"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

### 9.3 Future Schema Changes

**Planned Enhancements:**
- Add `budgets` table
- Add `recurring_transactions` table
- Add `reconciliation_status` to journal_entries
- Add `exchange_rates` table for multi-currency

---

## 10. Data Export Formats

### 10.1 JSON Export Schema

```json
{
  "version": "1.0",
  "exported_at": "2026-01-24T18:00:00Z",
  "currency": "USD",
  "accounts": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "code": "Assets:Bank:Checking",
      "name": "Checking Account",
      "type": "Asset",
      "description": "Primary checking account",
      "currency": "USD",
      "parent_code": "Assets:Bank",
      "is_active": true,
      "created_at": "2026-01-01T00:00:00Z"
    }
  ],
  "transactions": [
    {
      "id": "a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d",
      "date": "2026-01-15",
      "description": "Grocery shopping",
      "reference": "Receipt #12345",
      "tags": ["groceries", "food"],
      "notes": "Weekly grocery run",
      "entries": [
        {
          "account_code": "Expenses:Food:Groceries",
          "type": "Debit",
          "amount": "85.50",
          "currency": "USD"
        },
        {
          "account_code": "Assets:Bank:Checking",
          "type": "Credit",
          "amount": "85.50",
          "currency": "USD"
        }
      ]
    }
  ]
}
```

### 10.2 CSV Export Format

**Transactions CSV:**
```csv
Date,Description,Account,Debit,Credit,Balance,Tags,Reference
2026-01-15,Grocery shopping,Expenses:Food:Groceries,85.50,0.00,85.50,"groceries,food",Receipt #12345
2026-01-15,Grocery shopping,Assets:Bank:Checking,0.00,85.50,2414.50,"groceries,food",Receipt #12345
```

### 10.3 Ledger Format Export

Compatible with Ledger CLI format:

```ledger
2026/01/15 Grocery shopping
    ; Receipt #12345
    ; Tags: groceries, food
    Expenses:Food:Groceries              $85.50
    Assets:Bank:Checking                $-85.50
```

---

## 11. Security Considerations

### 11.1 File Permissions

Database file should be readable/writable only by owner:
```bash
chmod 600 ~/.gillions/data/gillions.db
```

### 11.2 SQL Injection Prevention

- Use parameterized queries (SQLAlchemy ORM)
- Never concatenate user input into SQL
- Validate all input before database operations

### 11.3 Data Encryption (Future)

Consider encrypting database file at rest:
- SQLCipher extension for SQLite
- OS-level encryption (FileVault, BitLocker)

---

## 12. Testing Strategy

### 12.1 Database Tests

**Unit Tests:**
- Model validation
- Constraint enforcement
- Relationship integrity

**Integration Tests:**
- Transaction creation and retrieval
- Balance calculations
- Report generation
- Data export/import

**Test Database:**
- Use in-memory SQLite for fast tests
- Separate test database file for integration tests

```python
# Test database setup
@pytest.fixture
def test_db():
    engine = create_engine('sqlite:///:memory:')
    Base.metadata.create_all(engine)
    return engine
```

---

## 13. Monitoring & Maintenance

### 13.1 Database Health Checks

```sql
-- Check database integrity
PRAGMA integrity_check;

-- Check foreign key integrity
PRAGMA foreign_key_check;

-- Analyze query performance
EXPLAIN QUERY PLAN SELECT ...;
```

### 13.2 Vacuum & Optimization

```sql
-- Reclaim unused space
VACUUM;

-- Update statistics for query optimizer
ANALYZE;
```

### 13.3 Database Size Monitoring

```bash
# Check database size
ls -lh ~/.gillions/data/gillions.db

# Check table sizes
sqlite3 gillions.db "SELECT name, SUM(pgsize) FROM dbstat GROUP BY name;"
```

---

## 14. Example: Complete Transaction Flow

### 14.1 Insert Transaction

```python
from sqlalchemy.orm import Session
from decimal import Decimal
from datetime import date

def create_transaction(session: Session):
    # Create transaction
    transaction = Transaction(
        date=date(2026, 1, 15),
        description="Grocery shopping",
        reference="Receipt #12345"
    )
    
    # Get accounts
    groceries = session.query(Account).filter_by(code="Expenses:Food:Groceries").first()
    checking = session.query(Account).filter_by(code="Assets:Bank:Checking").first()
    
    # Create journal entries
    entry1 = JournalEntry(
        transaction=transaction,
        account=groceries,
        entry_type="Debit",
        amount=Decimal("85.50"),
        currency="USD",
        sequence=1
    )
    
    entry2 = JournalEntry(
        transaction=transaction,
        account=checking,
        entry_type="Credit",
        amount=Decimal("85.50"),
        currency="USD",
        sequence=2
    )
    
    # Add to session and commit
    session.add(transaction)
    session.add(entry1)
    session.add(entry2)
    session.commit()
    
    return transaction
```

### 14.2 Query Balance

```python
def get_account_balance(session: Session, account_code: str, as_of_date: date = None):
    from sqlalchemy import func, case
    
    if as_of_date is None:
        as_of_date = date.today()
    
    account = session.query(Account).filter_by(code=account_code).first()
    
    # Calculate debits and credits
    result = session.query(
        func.coalesce(
            func.sum(case((JournalEntry.entry_type == 'Debit', JournalEntry.amount), else_=0)),
            0
        ).label('total_debits'),
        func.coalesce(
            func.sum(case((JournalEntry.entry_type == 'Credit', JournalEntry.amount), else_=0)),
            0
        ).label('total_credits')
    ).join(Transaction).filter(
        JournalEntry.account_id == account.id,
        Transaction.date <= as_of_date
    ).first()
    
    # Calculate balance based on account type
    if account.account_type in ['Asset', 'Expense']:
        balance = result.total_debits - result.total_credits
    else:
        balance = result.total_credits - result.total_debits
    
    return balance
```

---

This data persistence specification provides a complete foundation for implementing the Gillions bookkeeping system with SQLite and SQLAlchemy.
