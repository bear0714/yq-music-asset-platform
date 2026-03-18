# System Overview

The YQ Music Asset Platform is a distributed system designed for:

- Digital asset trading (music rights shares)
- Order matching and execution
- Double-entry ledger accounting
- Payment and settlement (Stripe integration)

---

## High-Level Architecture

```text
Client (Web / Admin)
        ↓
   CloudFront / DNS
        ↓
      ALB (HTTPS)
        ↓
  Nginx / Web Layer
        ↓
 Spring Cloud Gateway
        ↓
----------------------------------
|   Orchestrator Layer           |
|--------------------------------|
| home    | public-facing flows  |
| member  | user operations      |
| admin   | admin workflows      |
----------------------------------
        ↓
----------------------------------
|   Domain Services Layer        |
|--------------------------------|
| account | identity / auth      |
| music   | asset domain         |
| trade   | trading & ledger core|
----------------------------------
        ↓
----------------------------------
|   Supporting Services          |
|--------------------------------|
| projection   | read models     |
| notification | SMS / email     |
----------------------------------
        ↓
 Infrastructure Layer
 (ActiveMQ, Redis, Config, Eureka)
        ↓
 Database Layer (MySQL / Aurora)
 ```

 ---

## Service Layering Strategy

The system separates orchestration from domain logic:

- Orchestrator services (home, member, admin) handle API composition and workflow coordination, preventing business workflows from leaking into domain services
- Domain services (account, music, trade) encapsulate core business rules and invariants
- Supporting services (projection, notification) provide read optimization and asynchronous communication

This separation ensures:

- domain logic is not coupled to frontend or API concerns
- workflows can evolve without breaking core invariants
- services remain independently scalable and testable

## Domain Ownership

Each domain service owns its data and enforces its invariants:

- account → user profile, identity, authentication
- music → asset definitions, rights shares, metadata
- trade → orders, matching engine, balances, ledger (source of truth), financial state

Cross-domain access is performed via APIs or events — direct database access is not allowed.

## Event-Driven Communication

The system uses asynchronous messaging (JMS / ActiveMQ) to enforce controlled execution and decoupling:

- order processing and matching (instrument-level serialization)
- projection updates for read models
- notification triggers (email / SMS)

This enables:

- decoupling between services
- controlled concurrency (e.g., per-instrument matching)
- resilience against partial failures

## Core Guarantees

The platform is designed around strict financial correctness:

- All asset movements are enforced through a double-entry ledger
- Matching and settlement are executed atomically
- No balance mutation exists outside ledger-backed transactions
- Idempotency is enforced across external integrations (e.g., Stripe)

These guarantees ensure consistency under concurrency, retries, and partial failures.

## Design Philosophy

The system prioritizes correctness over latency in financial flows, while using asynchronous processing and read-model projections to maintain responsiveness for user-facing operations.