# Funds Flow Design

The funds flow defines how money moves into, within, and out of the platform.

It connects external payment systems (Stripe) with the internal ledger, ensuring deterministic, auditable, and idempotent financial operations.

---

## Design Goals

The funds flow is designed to guarantee:

- deterministic financial settlement
- ledger-backed accounting integrity
- idempotent processing under retries and duplicates
- separation between external events and internal financial truth

---

## Core Principles

### 1. Ledger is the Source of Truth

All financial state is derived from the ledger.

External systems (e.g., Stripe) are treated as:

- input signals
- not authoritative financial records

---

### 2. Separation of Business Events vs Financial Settlement

The system separates:

- business-level events (payment success, payout initiated)
- financial settlement (ledger posting)

Settlement occurs only after external data is finalized and all required financial attributes (e.g., fees, net amounts) are known.

---

### 3. Idempotent Financial Boundary

All financial operations are idempotent at the ledger level.

Each operation uses:

- refType
- refId

If a ledger transaction already exists:

- settlement is skipped
- no double posting can occur

---

## Flow Overview

The system supports four primary flows:

- deposit (user → platform)
- withdrawal (platform → user)
- trade settlement (internal)
- dividend distribution (internal)

All flows:

- pass through the ledger
- follow double-entry accounting
- are atomic and replay-safe

External flows (deposit, withdrawal) are asynchronous and event-driven, while internal flows (trade, dividend) are synchronous and ledger-atomic.

---

## Deposit Flow (Stripe → Platform)

### Deferred Settlement Mechanism

If BalanceTransaction is not yet available:

- settlement is not attempted
- the payment is marked as pending settlement
- a scheduled retry process periodically re-evaluates the payment

Settlement is executed only when:

- BalanceTransaction becomes available
- fee and net amounts are finalized

This converts external eventual consistency into deterministic internal behavior.

### Flow Stages

1. User initiates deposit
2. Stripe Checkout / PaymentIntent created
3. Webhook events update business state
4. Charge is finalized by Stripe
5. BalanceTransaction becomes available
6. Ledger settlement executed

---

### Key Characteristics

- Stripe events are asynchronous and unordered
- BalanceTransaction may not be immediately available
- settlement is deferred until financial data (fee and net) is finalized

---

### Settlement Rules

Ledger posting occurs only when:

- BalanceTransaction exists
- deposit is not already settled

---

### Ledger Posting

Principal:

- Debit: System Vault
- Credit: User Balance

Fee:

- Debit: User Balance
- Credit: Stripe Fee Account

---

### Invariants

- no deposit is settled more than once
- user balance reflects net amount
- Stripe fee is explicitly recorded

---

## Withdrawal Flow (Platform → Stripe → Bank)

### Flow Stages

1. User requests withdrawal
2. Funds are reserved (available → locked)
3. Platform approves request
4. Transfer to connected account
5. Stripe payout executed
6. payout.paid webhook received
7. Ledger settlement executed

---

### Key Characteristics

- payout execution is asynchronous
- withdrawal is pre-funded (amount + estimated fee) before payout
- pre-funding ensures payout execution does not fail due to insufficient connected-account funds
- Stripe fee is known only at payout finalization
- settlement occurs only at terminal state

---

### Two-Phase Withdrawal Model

Withdrawal consists of:

1. internal phase (fund locking, approval, transfer)
2. external phase (Stripe payout execution)

Ledger settlement occurs only after the external phase completes successfully.

---

### Settlement Rules

Ledger posting occurs only when:

- payout is in terminal state (paid)
- BalanceTransaction is available
- withdrawal not already settled

---

### Ledger Posting

Principal:

- Debit: User Balance
- Credit: System Vault

Fee:

- Debit: User Balance
- Credit: Stripe Fee Account

---

### Invariants

- funds must be pre-locked before payout
- no withdrawal is settled more than once
- user balance includes fee deduction

---

## Internal Flows

### Trade Settlement

- triggered by matching engine
- fully internal
- ledger-enforced atomic settlement

### Dividend Distribution

- distributes revenue to asset holders
- funds sourced from dividend pool
- ledger ensures proportional allocation

---

## Retry & Consistency Model

The system tolerates:

- duplicate events
- out-of-order delivery
- temporary external inconsistency

---

### Retry Strategy

- pending operations are retried until completion or terminal failure
- settlement is attempted only when required data is available
- retries remain safe due to ledger-level idempotence

---

### Exactly-Once Guarantee

Financial settlement is guaranteed to execute exactly once.

This is enforced by:

- ledger-level idempotence (refType + refId)
- transactional insertion of ledger entries
- retry-safe execution

Even under:

- duplicate webhook delivery
- system restarts
- partial failures

No duplicate financial posting can occur.

---

## Failure Boundaries

Failures are isolated into two layers:

### External Layer (Stripe)

- eventual consistency
- asynchronous execution

### Internal Layer (Ledger)

- strict transactional guarantees
- atomic settlement

---

## Relationship with Ledger and Matching Engine

- matching engine → produces trades
- funds flow → processes external money movement
- ledger → enforces financial truth

The ledger serves as the central authority ensuring consistency across all flows.

---