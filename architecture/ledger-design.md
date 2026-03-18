# Ledger Design

The ledger is the financial source of truth for the platform.

All asset movements are recorded as double-entry transactions, ensuring that the system remains financially consistent under concurrency, retries, and partial failures.

---

## Design Goals

The ledger system is designed to guarantee:

- strict financial correctness
- atomic settlement of trades
- auditability of all asset movements
- idempotent processing of external events (e.g., Stripe)

---

## Core Principle

Balances are operational state.

The ledger is financial truth.

All balance mutations must correspond to a ledger transaction.

This rule is enforced at the system boundary.

---

## Financial Invariants

The system enforces the following invariants at all times:

- total debits always equal total credits
- no balance mutation occurs without a corresponding ledger entry
- no ledger entry exists without a corresponding business event (e.g., trade, deposit, withdrawal)

---

## Ledger Integrity

Ledger transactions are chained using hashing to provide tamper detection.

Any modification to historical records can be detected through hash validation, ensuring audit integrity of the financial history.

---

## Accounting Model

The system follows a double-entry accounting model:

For every transaction:

- total debits = total credits
- no asset is created or destroyed implicitly

This guarantees conservation of assets:
- all movements are explicitly accounted for within the ledger

Each transaction consists of:

- LedgerTransaction (header)
- LedgerEntries (lines)

---

## Account Types

The system defines the following logical accounts:

### 1. User Account

Represents user-owned assets.

Each balance consists of:

- available → usable funds
- locked → reserved for orders or withdrawal


### 2. System Accounts

Internal accounts used to facilitate settlement and platform operations.

Examples include:

- System Vault → settlement boundary
- Platform Fee → trading fee accumulation
- Stripe Fee → external payment processing costs
- Platform Holding → custody of issued music assets
- Dividend Pool → distribution of asset-generated revenue

These accounts ensure that all asset movements remain within the double-entry accounting framework.

---

## Balance Model

Each asset for each user maintains:

- available
- locked

Invariant:

```text
available + locked = total balance
```

This balance is a derived operational state and must always be consistent with the ledger.

Any divergence between balance and ledger indicates a system error.

---

## Scope of Ledger

The ledger model applies to all financial movements in the system, not only trade settlement.

This includes:

- trade settlement
- deposits (e.g., Stripe payments)
- withdrawals (payout processing)
- dividend distributions
- platform-level fund movements

All such operations must follow the same principles:

- double-entry accounting
- ledger-backed balance mutations
- atomic and idempotent execution

Detailed flow definitions for these operations are described in `funds-flow.md`.

---

## Settlement Invariants

The system enforces strict guarantees during trade and funds flow settlement.

These invariants ensure that no asset can be lost, duplicated, or partially applied during settlement.


### 1. Atomic Settlement

All settlement operations are executed within a single transaction:

- trade creation
- balance updates
- ledger transaction insertion
- order state updates

If any step fails, the entire transaction rolls back.

Partial settlement is not possible.

### 2. Ledger-Balance Consistency

All balance mutations during settlement are derived from ledger entries.

This ensures:

- balances cannot diverge from financial truth
- settlement is always fully auditable

### 3. Conservation of Assets

For every trade:

- total debits equal total credits
- no asset is created or destroyed implicitly

This applies across:

- users
- system accounts (e.g., vault, fee accounts)

### 4. No Orphan State

The system guarantees:

- no trade exists without corresponding ledger entries
- no ledger entry exists without a corresponding business event
- no balance mutation occurs without a corresponding ledger entry

### 5. Deterministic Execution

Settlement is deterministic:

- given the same inputs, the same ledger entries and balance changes are produced
- no external side effects exist outside the transaction boundary

### 6. Failure Safety

Under any failure scenario:

- no partial balance updates persist
- no partial ledger entries persist
- system state remains financially consistent

Failures result in a full rollback of the transaction.

