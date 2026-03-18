# YQ Music Asset Platform
Digital asset platform with exchange-grade matching engine, ledger, and deterministic settlement

Music asset platform for fractional ownership, trading, revenue distribution, and settlement.

## Overview

YQ is a multi-domain platform that combines:

- **music domain** for asset onboarding, catalog, and revenue aggregation
- **trade domain** for order placement, matching, ledger, and settlement
- **account domain** for identity, balances, deposits, and withdrawals

The platform is designed as a modular backend system with strong emphasis on transactional integrity, clear domain boundaries, and operational scalability.

The system separates external event-driven flows (payments) from internal deterministic flows (matching and settlement), ensuring financial correctness under concurrency, retries, and partial failures.

---

## System at a Glance

```
Order → Matching Engine → Trades → Ledger → Balances
Stripe → Funds Flow → Ledger → Balances
Revenue → Distribution → Ledger → Balances
```

---

## Core Capabilities

### Asset and Catalog
- manage music assets and related metadata
- support fractional ownership and rights allocation
- aggregate revenue and performance data from external sources

### Trading
- order placement and cancellation
- bid / ask order book
- deterministic matching engine (price-time priority, partitioned execution)
- trade execution and post-trade processing

### Ledger and Settlement
- double-entry ledger as financial source of truth
- available vs locked balance model
- idempotent settlement with strong invariants
- dividend / revenue distribution via ledger

### Payments and Funds Flow
- Stripe-based deposit and withdrawal flows
- deferred settlement based on finalized financial data
- exactly-once financial posting via ledger idempotence

---

## Architecture Highlights

- domain-oriented service design across account, music, and trade
- per-instrument serialized matching via message partitioning (JMS)
- event-driven architecture for external integrations
- ledger-enforced financial correctness and idempotence
- modular persistence design using MySQL
- Stripe integration with deterministic settlement and retry model

---

## Representative Design Areas

### 1. Matching and Trading Flow
- validate and accept orders
- reserve balances (available → locked)
- deterministic matching (price-time priority)
- batch execution and trade generation
- ledger-backed atomic settlement

### 2. Ledger Model
- double-entry accounting structure
- clear separation between user balances and platform-side control accounts
- idempotence boundary around settlement and money movement

### 3. Revenue Distribution
- ingest external revenue data for music assets
- calculate distributable amounts
- allocate funds to holders based on ownership rules
- credit balances through ledger entries

### 4. Deposit and Withdrawal
- asynchronous Stripe event handling
- separation of payment success vs financial settlement
- deferred settlement with retry mechanism
- pre-funded withdrawal and controlled payout execution
- ledger-enforced exactly-once posting

---


## Design Documents

- [System Overview](./architecture/system-overview.md)
- [Matching Engine](./architecture/matching-engine.md)
- [Ledger Design](./architecture/ledger-design.md)
- [Funds Flow](./architecture/funds-flow.md)

---

## Selected Technical Stack

Java / Spring Boot · MyBatis · MySQL · ActiveMQ / JMS · Stripe · AWS

---

## Repository Roadmap

This public repository is intended as an architecture and portfolio showcase. It will gradually include:

- architecture notes
- domain models and schema excerpts
- selected code samples
- sequence diagrams and flow descriptions

---

## Why This Project Matters

This project demonstrates the design of a financial system that combines trading, accounting, and external payment integration with strong guarantees around correctness, idempotence, and consistency. It reflects a focus on building systems that behave deterministically even under real-world conditions such as retries, partial failures, and asynchronous external dependencies.