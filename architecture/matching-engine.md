# Matching Engine Design

The matching engine is responsible for order execution in the trading system.

It ensures deterministic, fair, and efficient matching of buy and sell orders while maintaining strict consistency with the ledger.

---

## Design Goals

The matching engine is designed to guarantee:

- price-time priority (fairness)
- deterministic execution
- high throughput with controlled concurrency
- strict consistency with settlement (ledger-backed)

---

## Core Principles

### 1. Price-Time Priority

Orders are matched based on:

1. best price
2. earliest submission time

This ensures fairness and prevents manipulation.

---

### 2. Deterministic Matching

Given the same order book state and incoming order:

- the matching result must always be identical

This is critical for:

- debugging
- replay
- auditability

---

### 3. Separation from Settlement

The matching engine does NOT move funds.

It only:

- determines matched trades
- produces trade instructions

Actual balance mutation is handled by the ledger during settlement.

---

## Order Book Model

Each asset has its own order book.

The order book consists of:

- bid side (buy orders) → sorted by price DESC, time ASC
- ask side (sell orders) → sorted by price ASC, time ASC

---

## Order Lifecycle

An order goes through the following states:

- NEW → accepted into the system
- OPEN → resting in the order book
- PARTIALLY_FILLED → partially matched
- FILLED → fully executed
- CANCELLED → removed by user or system

---

## Matching Flow

When a new order arrives:

1. Validate order (price, quantity, user balance pre-check)
2. Attempt to match against opposite side of the book in bounded batches
3. Generate one or more trades
4. Update order states
5. Forward trade results to settlement (ledger)

---

## Processing Model

The matching engine processes orders using a partitioned, message-driven model.

- orders are published to a message queue
- partition key = instrumentId
- all orders for the same instrument are processed sequentially

This guarantees:

- strict ordering per instrument
- no concurrent matching for the same order book
- deterministic execution

At the same time:

- different instruments are processed in parallel
- horizontal scalability is achieved through partition distribution

---

## Pre-Trade Validation & Reservation

Before an order enters the matching phase, the system performs strict pre-trade checks.

### Buy Order

- sufficient available balance must exist
- required funds are moved from:
  - available → locked

### Sell Order

- sufficient asset quantity must exist
- required shares are moved from:
  - available → locked

This reservation guarantees:

- no double spending
- no overselling
- deterministic settlement

The matching engine operates only on already-reserved orders.

## Matching Rules

### Buy Order

A buy order matches when:

```
buy_price >= best_ask_price
```

### Sell Order

A sell order matches when:

```
sell_price <= best_bid_price
```

Matching continues until:

- the incoming order is fully filled, or
- no more matching price levels exist

---

## Partial Fill Handling

If a match cannot fully satisfy an order:

- the remaining quantity stays in the order book
- the order transitions to PARTIALLY_FILLED or OPEN

---

## Trade Generation

Each successful match produces a trade:

- buyer order id
- seller order id
- price (resting order price / maker price)
- quantity
- timestamp

These trades are immutable and form the basis for settlement.

---

## Concurrency Model

To ensure consistency:

- matching is executed per-asset (partitioned)
- each order book is processed sequentially

This avoids:

- race conditions
- double matching
- inconsistent fills

Scaling is achieved by:

- distributing different assets across partitions

---

## Failure Handling

Matching must be:

- idempotent
- replayable

If failure occurs before settlement:

- matching can be safely retried
- no financial state is affected

---

## Invariants

The matching engine guarantees:

- no order is matched more than its quantity
- all matched orders must have pre-reserved capital
- no trade is generated without a valid opposing order
- total matched quantity never exceeds available liquidity

These invariants ensure correctness before settlement.

---

## Relationship with Ledger

The matching engine produces:

- trade events

The ledger:

- validates and settles those trades
- updates balances

The matching engine never directly modifies financial state.

---