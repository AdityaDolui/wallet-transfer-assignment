# Wallet Transfer Service - Design Document

## Problem Statement

- Idempotent request processing
- Double-entry ledger consistency
- Safe concurrent execution
- Atomic transaction processing
- Retry-safe behavior
- Transfer state management

The system must behave correctly even when:

- Duplicate requests occur
- Clients retry requests
- Concurrent updates happen
- Partial failures occur

---

# Key Design Decisions

Before implementation, the following architectural decisions were made.

## Idempotency Strategy

The service guarantees **exactly-once API semantics** using a dedicated `idempotency_records` table.

Each request contains:

```json
{
  "idempotencyKey": "abc123"
}
```

A SHA-256 request fingerprint (`requestHash`) is generated from:

```text
fromWalletId
toWalletId
amount
```

### Processing Rules

| Scenario | Result |
|-----------|----------|
| Same key + Same payload | Return original response |
| Same key + Different payload | 409 Conflict |
| New key | Process transfer |

### Why Request Hashing?

Without request hashing:

```json
{
  "idempotencyKey": "abc123",
  "amount": 100
}
```

and

```json
{
  "idempotencyKey": "abc123",
  "amount": 500
}
```

would incorrectly be treated as the same request.

The request fingerprint ensures that the same idempotency key cannot be reused for a different business operation.

### Source of Truth

The **idempotency_records** table is the authoritative source for idempotency handling.

The `transfers` table does **not** store `idempotency_key`.

Reason:

- Separation of concerns
- Cleaner domain model
- Dedicated retry management
- Avoid duplicated responsibility

---

## Concurrency Strategy

The service uses PostgreSQL row-level locking through JPA pessimistic locking.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
```

Equivalent SQL:

```sql
SELECT *
FROM wallets
WHERE id IN (:sourceWalletId, :destinationWalletId)
ORDER BY id
FOR UPDATE;
```

### Why?

This locks only the wallets participating in the transfer.

Benefits:

- Prevents double spending
- Prevents lost updates
- Prevents race conditions

### Example

Initial balance:

```text
Wallet A = 100
```

Concurrent requests:

```text
Transfer A = 80
Transfer B = 80
```

Without locking:

```text
Both read balance = 100
Both succeed

Final balance = -60
```

With locking:

```text
Transfer A acquires lock
Transfer B waits

Transfer A completes

Transfer B rechecks balance

Transfer B fails due to insufficient funds
```

Final result:

```text
Balance = 20
```

---

## Deadlock Prevention

Wallet rows are always locked in deterministic order.

```text
Smaller Wallet ID
        ↓
Larger Wallet ID
```

Every transaction follows the same ordering strategy.

This eliminates circular wait conditions and prevents deadlocks.

---

## Transaction Safety

The complete transfer workflow executes inside a single database transaction.

```java
@Transactional
```

If any operation fails:

- Wallet balance update
- Transfer persistence
- Ledger entry creation
- Idempotency record persistence

the entire transaction rolls back.

This guarantees atomicity.

---

## Ledger Consistency

Every transfer generates exactly:

```text
1 Debit Entry
1 Credit Entry
```

Example:

```text
Wallet A → Wallet B
Amount = 100
```

Ledger:

```text
Wallet A   DEBIT   100
Wallet B   CREDIT  100
```

Invariant:

```text
Total Debits = Total Credits
```

This guarantees accounting consistency.

---

# Why PostgreSQL?

## Why SQL?

The assignment requires:

- ACID transactions
- Strong consistency
- Referential integrity
- Row-level locking
- Transaction isolation

These are native strengths of relational databases.

---

## Why PostgreSQL Instead of MySQL?

PostgreSQL provides:

- Mature transaction handling
- Strong locking semantics
- Reliable concurrency control
- Advanced indexing capabilities
- Excellent support for financial workloads

Since correctness is more important than raw throughput for this assignment, PostgreSQL is a natural choice.

---

# High Level Architecture

```text
                +------------------+
                | REST Controller  |
                +---------+--------+
                          |
                          v
                +------------------+
                | Transfer Service |
                +---------+--------+
                          |
        +-----------------+------------------+
        |                 |                  |
        v                 v                  v
+---------------+ +---------------+ +------------------+
| Wallet Repo   | | Transfer Repo | | Idempotency Repo |
+---------------+ +---------------+ +------------------+
                          |
                          v
                +------------------+
                | Ledger Repository|
                +------------------+
                          |
                          v
                +------------------+
                |   PostgreSQL     |
                +------------------+
```

---

# Domain Modeling

The system is modeled around four business entities.

| Domain | Responsibility |
|----------|----------------|
| Wallet | Current balance |
| Transfer | Business transaction |
| LedgerEntry | Accounting record |
| IdempotencyRecord | Retry protection |

Each entity owns a specific business responsibility.

---

# Entity Relationship Diagram

```text
+------------------+
|     Wallet       |
+------------------+
| id               |
| balance          |
+------------------+
        ^
        |
        |
+------------------+
|    Transfer      |
+------------------+
| id               |
| from_wallet_id   |
| to_wallet_id     |
| amount           |
| status           |
+------------------+
        |
        |
        v
+------------------+
|  Ledger Entry    |
+------------------+
| transfer_id      |
| wallet_id        |
| entry_type       |
| amount           |
+------------------+

Transfer
    |
    |
    v
+----------------------+
| Idempotency Record   |
+----------------------+
| request_hash         |
| response_payload     |
+----------------------+
```

---

# Database Schema Design

The database model is intentionally split into separate tables so each table has a single responsibility.

---

## wallets

Stores the current spendable balance.

| Column | Type | Constraints |
|----------|----------|-------------|
| id | UUID | Primary Key |
| balance | BIGINT | NOT NULL |
| version | BIGINT | NOT NULL |
| created_at | TIMESTAMP | NOT NULL |

### Constraints

```sql
PRIMARY KEY (id)
```

### Indexes

```sql
PRIMARY KEY (id)
```

### Responsibility

Represents the current state of a wallet.

---

## transfers

Stores business transaction information.

| Column | Type | Constraints |
|----------|----------|-------------|
| id | UUID | Primary Key |
| from_wallet_id | UUID | Foreign Key |
| to_wallet_id | UUID | Foreign Key |
| amount | BIGINT | NOT NULL |
| status | VARCHAR(30) | NOT NULL |
| created_at | TIMESTAMP | NOT NULL |

### Constraints

```sql
PRIMARY KEY(id)

FOREIGN KEY(from_wallet_id)
REFERENCES wallets(id)

FOREIGN KEY(to_wallet_id)
REFERENCES wallets(id)
```

### Indexes

```sql
CREATE INDEX idx_transfer_source_wallet
ON transfers(from_wallet_id);

CREATE INDEX idx_transfer_destination_wallet
ON transfers(to_wallet_id);
```

### Responsibility

Represents the business transfer lifecycle.

---

## ledger_entries

Stores immutable accounting records.

| Column | Type | Constraints |
|----------|----------|-------------|
| id | UUID | Primary Key |
| transfer_id | UUID | Foreign Key |
| wallet_id | UUID | Foreign Key |
| entry_type | VARCHAR(20) | NOT NULL |
| amount | BIGINT | NOT NULL |
| created_at | TIMESTAMP | NOT NULL |

### Constraints

```sql
PRIMARY KEY(id)

FOREIGN KEY(transfer_id)
REFERENCES transfers(id)

FOREIGN KEY(wallet_id)
REFERENCES wallets(id)
```

### Indexes

```sql
CREATE INDEX idx_ledger_transfer
ON ledger_entries(transfer_id);

CREATE INDEX idx_ledger_wallet
ON ledger_entries(wallet_id);
```

### Responsibility

Provides an immutable accounting trail.

---

## idempotency_records

Stores retry metadata and original response information.

| Column | Type | Constraints |
|----------|----------|-------------|
| idempotency_key | VARCHAR(255) | Primary Key |
| request_hash | VARCHAR(128) | NOT NULL |
| transfer_id | UUID | Foreign Key |
| response_payload | TEXT | NULL |
| status | VARCHAR(30) | NOT NULL |
| created_at | TIMESTAMP | NOT NULL |

### Constraints

```sql
PRIMARY KEY(idempotency_key)

FOREIGN KEY(transfer_id)
REFERENCES transfers(id)
```

### Indexes

```sql
CREATE UNIQUE INDEX idx_idempotency_key
ON idempotency_records(idempotency_key);

CREATE INDEX idx_request_hash
ON idempotency_records(request_hash);
```

### Responsibility

Provides exactly-once request processing and retry safety.

---

# Transfer Workflow

```text
1. Receive Request

2. Generate Request Hash

3. Check Idempotency Record

4. Lock Source Wallet

5. Lock Destination Wallet

6. Validate Balance

7. Create Transfer (PENDING)

8. Debit Source Wallet

9. Credit Destination Wallet

10. Create Debit Ledger Entry

11. Create Credit Ledger Entry

12. Mark Transfer PROCESSED

13. Persist Idempotency Record

14. Commit Transaction
```

---

# Transfer State Machine

Supported states:

```text
PENDING
PROCESSED
FAILED
```

Allowed transitions:

```text
PENDING → PROCESSED
PENDING → FAILED
```

Rejected transitions:

```text
PROCESSED → PENDING
FAILED → PROCESSED
```

---

# SOLID Principles

## Single Responsibility Principle

Each class has one responsibility.

Examples:

- TransferController
- TransferServiceImpl
- WalletRepository
- LedgerEntryFactory

---

## Open Closed Principle

The design supports extension without modifying existing behavior.

Example:

```text
DEBIT
CREDIT
```

can later evolve into:

```text
DEBIT
CREDIT
REFUND
REVERSAL
```

---

## Liskov Substitution Principle

Consumers depend on abstractions.

Example:

```java
TransferService
```

can be replaced by:

```java
TransferServiceImpl
```

without impacting consumers.

---

## Interface Segregation Principle

Interfaces expose only relevant behavior.

Example:

```java
TransferService
```

contains transfer-specific operations only.

---

## Dependency Inversion Principle

Controllers depend on service abstractions.

Services depend on repository abstractions.

Spring injects concrete implementations.

---

# Design Patterns

| Pattern | Usage |
|----------|---------|
| Repository Pattern | Database abstraction |
| Service Layer Pattern | Business orchestration |
| Factory Pattern | Ledger entry creation |
| Builder Pattern | DTO and entity construction |
| Rich Domain Model | Business rules inside entities |
| State Transition Pattern | Transfer lifecycle management |
| Idempotency Record Pattern | Retry-safe processing |

---

# Testing Strategy

## Unit Tests

Validate:

- Balance validation
- State transitions
- Ledger creation logic
- Idempotency validation

---

## Integration Tests

Validate:

- Transfer execution
- Ledger persistence
- Transaction boundaries
- Idempotency behavior

Using:

```text
PostgreSQL Testcontainers
```

---

## Concurrency Test

Scenario:

```text
Balance = 100

Transfer A = 80
Transfer B = 80
```

Expected result:

```text
One transfer succeeds

One transfer fails

Final balance = 20
```

This validates the pessimistic locking strategy.

---

# Future Improvements

The current implementation intentionally uses a modular monolith because the assignment prioritizes correctness, simplicity, and delivery speed.

Potential future enhancements:

## Outbox Pattern

Reliable event publishing after successful transactions.

## Saga Pattern

If the application evolves into multiple services.

Example:

```text
Wallet Service
Ledger Service
Notification Service
```

Saga can coordinate distributed workflows and compensating actions.

## Transfer History API

Support transfer auditing and reporting.

## Event-Driven Architecture

Real-time notifications and reporting pipelines.

## Ledger-Derived Balances

Calculate balances directly from ledger entries for stronger auditability.

## Optimistic Locking

Potential throughput optimization for specific workloads.

---

# Tradeoffs

A modular monolithic architecture was chosen instead of microservices.

### Advantages

- Faster delivery
- Simpler deployment
- Easier testing
- Strong transactional consistency
- No distributed transaction complexity

### Future Evolution

The current design can evolve into independently deployable services if business scale and organizational requirements justify the additional complexity.
