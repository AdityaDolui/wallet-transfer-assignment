Wallet Transfer Service - Design Document
Problem Statement

The objective is to build a wallet-to-wallet transfer service that guarantees:

Idempotent request processing
Double-entry ledger consistency
Safe concurrent execution
Atomic transaction processing
Retry-safe behavior
Transfer state management

The system must behave correctly even when duplicate requests, retries, concurrent updates, and partial failures occur.

Key Design Decisions

Before implementation, the following architectural decisions were made.

Idempotency

A dedicated idempotency_records table is used.

Each request contains:

idempotencyKey

A SHA-256 request fingerprint (requestHash) is generated from:

fromWalletId
toWalletId
amount

Behavior:

Scenario	Result
Same key + same payload	Return original response
Same key + different payload	409 Conflict
New key	Process transfer

This guarantees duplicate requests never create duplicate transfers.

Concurrency

The service uses PostgreSQL row-level locking through JPA pessimistic locking.

@Lock(PESSIMISTIC_WRITE)

Equivalent SQL:

SELECT *
FROM wallets
FOR UPDATE;

This prevents two concurrent transfers from spending the same balance.

Deadlock Prevention

Wallets are always locked in deterministic order.

Lower Wallet ID
        ↓
Higher Wallet ID

Every transaction follows the same ordering strategy.

This eliminates circular waiting conditions.

Transaction Safety

The complete transfer workflow executes inside a single database transaction.

@Transactional

If any operation fails:

balance update
transfer persistence
ledger creation
idempotency persistence

the entire transaction rolls back.

This guarantees atomicity.

Ledger Consistency

Every transfer generates exactly:

1 Debit Entry
1 Credit Entry

Example:

Wallet A → Wallet B
Amount = 100

Ledger:

Wallet A   DEBIT   100
Wallet B   CREDIT  100

Therefore:

Total Debits = Total Credits

for every transfer.

Why PostgreSQL?
Why SQL?

The assignment requires:

ACID transactions
Row-level locking
Referential integrity
Strong consistency
Transactional guarantees

These are native strengths of relational databases.

Why PostgreSQL Instead of MySQL?

PostgreSQL provides:

Mature transaction handling
Strong locking semantics
Reliable concurrency control
Excellent support for financial workloads
Widely adopted in banking and fintech systems

Since this assignment focuses on correctness and consistency, PostgreSQL is a strong fit.

High Level Architecture
                +------------------+
                |  REST Controller |
                +---------+--------+
                          |
                          v
                +------------------+
                | Transfer Service |
                +---------+--------+
                          |
          +---------------+----------------+
          |               |                |
          v               v                v
+----------------+ +--------------+ +------------------+
| Wallet Repo    | | Transfer Repo| | Idempotency Repo |
+----------------+ +--------------+ +------------------+
          |
          v
+----------------------+
| Ledger Entry Repo    |
+----------------------+
          |
          v
+----------------------+
| PostgreSQL Database  |
+----------------------+
Domain Modeling & Database Design

The system is modeled around four core business concepts.

Domain Overview
Wallet
Transfer
LedgerEntry
IdempotencyRecord

Each domain object owns a specific responsibility.

Entity Relationship Diagram
+------------+
|  Wallet    |
+------------+
| id         |
| balance    |
+------------+
      |
      | 1
      |
      | *
+------------+
| Transfer   |
+------------+
| id         |
| amount     |
| status     |
+------------+
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
Relationship Model
Wallet
 ├── Source Wallet in many Transfers
 ├── Destination Wallet in many Transfers
 └── Has many Ledger Entries

Transfer
 ├── References Source Wallet
 ├── References Destination Wallet
 ├── Creates exactly 2 Ledger Entries
 └── Has exactly 1 Idempotency Record

LedgerEntry
 ├── Belongs to one Wallet
 └── Belongs to one Transfer

IdempotencyRecord
 └── References one Transfer
Table Design
wallets

Stores current spendable balance.

Column	Description
id	Wallet identifier
balance	Current balance
version	Future optimistic locking support
created_at	Audit timestamp
Responsibility

Represents current wallet state.

transfers

Stores business transaction information.

Column	Description
id	Transfer identifier
idempotency_key	Client retry key
from_wallet_id	Source wallet
to_wallet_id	Destination wallet
amount	Transfer amount
status	Transfer state
created_at	Audit timestamp
Responsibility

Represents transfer lifecycle.

ledger_entries

Stores accounting records.

Column	Description
id	Ledger identifier
transfer_id	Parent transfer
wallet_id	Related wallet
entry_type	DEBIT/CREDIT
amount	Entry amount
created_at	Audit timestamp
Responsibility

Provides immutable audit trail.

idempotency_records

Stores retry information.

Column	Description
idempotency_key	Unique request key
request_hash	Request fingerprint
transfer_id	Related transfer
response_payload	Original response
status	Processing status
created_at	Audit timestamp
Responsibility

Provides exactly-once API semantics.

Transfer Workflow
1. Receive Request

2. Compute Request Hash

3. Check Idempotency Table

4. Lock Source & Destination Wallets

5. Validate Balance

6. Create Transfer (PENDING)

7. Debit Source Wallet

8. Credit Destination Wallet

9. Create Debit Ledger Entry

10. Create Credit Ledger Entry

11. Mark Transfer PROCESSED

12. Store Idempotency Response

13. Commit Transaction
Concurrency Strategy

The service uses pessimistic locking.

@Lock(PESSIMISTIC_WRITE)

Benefits:

Prevents double spending
Prevents lost updates
Guarantees balance correctness

Lock acquisition order is deterministic to prevent deadlocks.

Idempotency Strategy

The service implements the Idempotency Record Pattern.

First Request
Create idempotency record
Process transfer
Store response
Retry Request
Find idempotency record
Compare request hash
Return stored response
Invalid Retry
Same Key
Different Request Hash

Result:

409 Conflict
Double Entry Ledger Strategy

Each transfer creates:

1 Debit Entry
1 Credit Entry

Invariant:

Sum(Debits) = Sum(Credits)

This guarantees accounting consistency.

Transfer State Machine

Supported states:

PENDING
PROCESSED
FAILED

Allowed transitions:

PENDING → PROCESSED
PENDING → FAILED

Rejected transitions:

PROCESSED → PENDING
FAILED → PROCESSED
SOLID Principles
Single Responsibility Principle

Each class has one responsibility.

Examples:

TransferController
TransferServiceImpl
LedgerEntryFactory
WalletRepository
Open Closed Principle

New transfer types or ledger behaviors can be added without modifying existing abstractions.

Liskov Substitution Principle

Consumers depend on abstractions such as:

TransferService

not concrete implementations.

Interface Segregation Principle

Interfaces expose only required operations.

Example:

TransferService

contains transfer-related behavior only.

Dependency Inversion Principle

Controllers depend on service abstractions.

Services depend on repository abstractions.

Spring injects implementations.

Design Patterns
Repository Pattern

Used in:

WalletRepository
TransferRepository
LedgerEntryRepository
IdempotencyRepository

Purpose:

Persistence abstraction.

Service Layer Pattern

Used in:

TransferService
TransferServiceImpl

Purpose:

Business orchestration.

Factory Pattern

Used in:

LedgerEntryFactory

Purpose:

Centralized ledger creation.

Builder Pattern

Used in DTO and entity construction.

Purpose:

Readable object creation.

Rich Domain Model

Used in:

wallet.debit()
wallet.credit()

transfer.markProcessed()
transfer.markFailed()

Purpose:

Keep business rules inside domain objects.

State Transition Pattern

Used in Transfer entity.

Purpose:

Protect lifecycle integrity.

Idempotency Record Pattern

Used through:

idempotency_records
request_hash
stored_response

Purpose:

Guarantee exactly-once API behavior.

Testing Strategy
Unit Tests

Validate:

Balance checks
State transitions
Idempotency validation
Integration Tests

Validate:

Transfer execution
Ledger creation
Idempotency behavior

Using PostgreSQL Testcontainers.

Concurrency Test

Two simultaneous transfers attempt to debit the same wallet.

Expected result:

No double spending
Correct final balance
One successful transfer

This validates the locking strategy.

Future Improvements

If the system evolves beyond the scope of this assignment:

Outbox Pattern

For reliable event publishing.

Saga Pattern

If the application is split into microservices.

Ledger Derived Balances

Balance calculated directly from ledger entries.

Transfer History APIs

Audit and reporting support.

Event Driven Architecture

Real-time notifications and reporting.

Optimistic Locking

Potential throughput optimization for specific workloads.
