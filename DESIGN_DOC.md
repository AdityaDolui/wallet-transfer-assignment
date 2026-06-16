# Wallet Transfer Service

## Problem Statement

The objective is to build a wallet-to-wallet transfer service that guarantees:

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

A dedicated `idempotency_records` table is used.

Each request contains:

```text
idempotencyKey
```

A SHA-256 request fingerprint (`requestHash`) is generated using:

```text
fromWalletId
toWalletId
amount
```

### Behavior

| Scenario | Result |
|-----------|----------|
| Same key + Same payload | Return original response |
| Same key + Different payload | 409 Conflict |
| New key | Process transfer |

### Why Request Hashing?

Without request hashing:

```json
{
  "idempotencyKey":"abc123",
  "amount":100
}
```

and

```json
{
  "idempotencyKey":"abc123",
  "amount":500
}
```

would incorrectly be treated as the same request.

The request fingerprint ensures that the same idempotency key cannot be reused for a different business operation.

---

## Concurrency Strategy

The service uses:

```java
@Lock(PESSIMISTIC_WRITE)
```

Equivalent PostgreSQL statement:

```sql
SELECT *
FROM wallets
FOR UPDATE;
```

### Why?

This prevents:

- Double spending
- Lost updates
- Race conditions

Example:

```text
Wallet Balance = 100

Transfer A = 80
Transfer B = 80
```

Without locking:

```text
Both requests read 100
Both succeed
Final balance = -60
```

With pessimistic locking:

```text
Transfer A acquires lock
Transfer B waits

Transfer A completes
Transfer B rechecks balance

Transfer B fails
```

Result:

```text
Final Balance = 20
```

---

## Deadlock Prevention

Wallets are always locked in deterministic order.

```text
Lower Wallet ID
       ↓
Higher Wallet ID
```

This prevents circular wait conditions.

---

## Transaction Safety

The entire transfer workflow executes inside a single transaction.

```java
@Transactional
```

If any step fails:

- Wallet update
- Transfer save
- Ledger creation
- Idempotency persistence

everything is rolled back.

This guarantees atomicity.

---

## Ledger Consistency

Every transfer creates:

```text
1 Debit Entry
1 Credit Entry
```

Example:

```text
Wallet A -> Wallet B
Amount = 100
```

Ledger:

```text
Wallet A   DEBIT    100
Wallet B   CREDIT   100
```

Invariant:

```text
Total Debit = Total Credit
```

---

# Why PostgreSQL?

## Why SQL?

The assignment requires:

- ACID transactions
- Strong consistency
- Referential integrity
- Row-level locking

These are native strengths of relational databases.

---

## Why PostgreSQL Instead of MySQL?

PostgreSQL provides:

- Mature transaction handling
- Strong concurrency control
- Reliable locking semantics
- Excellent support for financial systems

Since correctness is more important than raw throughput, PostgreSQL is a natural choice.

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

## Core Domains

| Domain | Responsibility |
|----------|----------------|
| Wallet | Current balance |
| Transfer | Business transaction |
| LedgerEntry | Accounting records |
| IdempotencyRecord | Retry protection |

---

# Entity Relationship Diagram

```text
+----------------+
|    Wallet      |
+----------------+
| id             |
| balance        |
+----------------+
       |
       | 1
       |
       | *
+----------------+
|   Transfer     |
+----------------+
| id             |
| amount         |
| status         |
+----------------+
       |
       | 1
       |
       | 2
+----------------+
| Ledger Entry   |
+----------------+
| DEBIT/CREDIT   |
+----------------+

Transfer
     |
     | 1
     |
     | 1
+--------------------+
| IdempotencyRecord  |
+--------------------+
```

---

# Database Schema

## wallets

| Column | Type | Description |
|----------|----------|-------------|
| id | UUID | Primary Key |
| balance | BIGINT | Current balance |
| version | BIGINT | Future optimistic locking |
| created_at | TIMESTAMP | Audit timestamp |

### Responsibility

Stores the current spendable balance.

---

## transfers

| Column | Type | Description |
|----------|----------|-------------|
| id | UUID | Primary Key |
| idempotency_key | VARCHAR | Retry key |
| from_wallet_id | UUID | Source wallet |
| to_wallet_id | UUID | Destination wallet |
| amount | BIGINT | Transfer amount |
| status | VARCHAR | PENDING / PROCESSED / FAILED |
| created_at | TIMESTAMP | Audit timestamp |

### Foreign Keys

```text
from_wallet_id -> wallets.id
to_wallet_id   -> wallets.id
```

---

## ledger_entries

| Column | Type | Description |
|----------|----------|-------------|
| id | UUID | Primary Key |
| transfer_id | UUID | Parent transfer |
| wallet_id | UUID | Related wallet |
| entry_type | VARCHAR | DEBIT / CREDIT |
| amount | BIGINT | Entry amount |
| created_at | TIMESTAMP | Audit timestamp |

### Foreign Keys

```text
transfer_id -> transfers.id
wallet_id   -> wallets.id
```

---

## idempotency_records

| Column | Type | Description |
|----------|----------|-------------|
| idempotency_key | VARCHAR | Primary Key |
| request_hash | VARCHAR | Request fingerprint |
| transfer_id | UUID | Related transfer |
| response_payload | TEXT | Stored response |
| status | VARCHAR | PENDING / COMPLETED |
| created_at | TIMESTAMP | Audit timestamp |

### Foreign Keys

```text
transfer_id -> transfers.id
```

---

# Transfer Workflow

```text
1. Receive Request

2. Generate Request Hash

3. Check Idempotency Table

4. Lock Source Wallet

5. Lock Destination Wallet

6. Validate Balance

7. Create Transfer (PENDING)

8. Debit Source Wallet

9. Credit Destination Wallet

10. Create Debit Ledger Entry

11. Create Credit Ledger Entry

12. Mark Transfer PROCESSED

13. Store Response

14. Commit Transaction
```

---

# Transfer State Machine

```text
PENDING
   |
   +------> PROCESSED

   |
   +------> FAILED
```

Allowed transitions:

```text
PENDING -> PROCESSED
PENDING -> FAILED
```

Rejected transitions:

```text
PROCESSED -> PENDING
FAILED -> PROCESSED
```

---

# SOLID Principles

## Single Responsibility Principle

Each class has a single responsibility.

Examples:

- TransferController
- TransferServiceImpl
- LedgerEntryFactory
- WalletRepository

---

## Open Closed Principle

The design supports extension without modifying existing behavior.

Example:

```text
DEBIT
CREDIT
```

can later become:

```text
DEBIT
CREDIT
REVERSAL
REFUND
```

---

## Liskov Substitution Principle

Services are consumed through abstractions.

```java
TransferService
```

can be replaced by:

```java
TransferServiceImpl
```

without changing consumers.

---

## Interface Segregation Principle

Interfaces expose only relevant operations.

---

## Dependency Inversion Principle

Controllers depend on service abstractions.

Services depend on repository abstractions.

---

# Design Patterns

| Pattern | Usage |
|-----------|---------|
| Repository Pattern | Database abstraction |
| Service Layer Pattern | Business orchestration |
| Factory Pattern | Ledger creation |
| Builder Pattern | DTO and Entity construction |
| Rich Domain Model | Business rules inside entities |
| State Pattern | Transfer lifecycle |
| Idempotency Record Pattern | Retry-safe processing |

---

# Testing Strategy

## Unit Tests

- Balance validation
- State transitions
- Idempotency validation

## Integration Tests

- Transfer execution
- Ledger creation
- Database persistence

Using:

```text
PostgreSQL Testcontainers
```

## Concurrency Test

Scenario:

```text
Balance = 100

Transfer A = 80
Transfer B = 80
```

Expected:

```text
One success
One failure
Final balance = 20
```

---

# Future Improvements

## Outbox Pattern

Reliable event publishing.

## Saga Pattern

If system evolves into microservices.

## Transfer History API

Transfer tracking and auditing.

## Event Driven Architecture

Notifications and reporting.

## Ledger Derived Balances

Compute balances directly from ledger.

## Optimistic Locking

Possible performance optimization.

---

# Tradeoffs

A modular monolith was chosen instead of microservices.

### Advantages

- Faster delivery
- Simpler deployment
- Strong transactional consistency
- Easier testing

### Future Evolution

If the platform grows:

```text
Wallet Service
Ledger Service
Notification Service
```

can be separated and coordinated using Saga Pattern.
