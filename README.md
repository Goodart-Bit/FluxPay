# Welcome to FluxPay

The **“FluxPay”** application is a backend system for processing financial payments and maintaining a reliable record of account transactions.

The application simulates a simplified payment platform where users can transfer money between accounts.

The system must be able to:

- Create and process payments.
- Prevent duplicate payments from being processed.
- Maintain an accurate financial record of transactions.
- Process events asynchronously.
- Handle failures and retry processing when appropriate.
- Keep an auditable history of important financial operations.

The application should prioritize **correctness and consistency over throughput** when processing financial operations.

Your task is to implement the application according to the specifications below.

---

# Application Lifecycle

The application consists of several stages involved in processing a payment.

A simplified payment lifecycle is:

```text
Payment Created
      ↓
Payment Processing
      ↓
Payment Authorized
      ↓
Payment Captured
      ↓
Payment Settled
```

A payment may also fail during processing:

```text
Payment Processing
       ↓
     Failed
```

A successfully captured payment may subsequently be refunded:

```text
Payment Captured
       ↓
    Refunded
```

The main flow is:

1. A client submits a payment request.
2. The application validates the request.
3. The payment is created.
4. The payment is published as an event.
5. A payment processor consumes the event.
6. The payment is validated and processed.
7. The corresponding financial entries are recorded.
8. The payment status is updated.
9. A resulting event is published.
10. Other components may react to the event independently.

The system must be able to recover from failures without corrupting the financial state.

---

# Domain Model

The system contains at least the following concepts:

- **Account**
- **Payment**
- **Ledger Entry**
- **Payment Event**

## Account

An account represents a customer's financial account.

Example:

```text
Account
--------------------------------
id: 1001
customerId: 42
currency: USD
balance: 1500.00
status: ACTIVE
```

An account may have one of the following statuses:

```text
ACTIVE
FROZEN
CLOSED
```

Money must not be transferred from an account that cannot perform transactions.

---

## Payment

A payment represents a request to move money from one account to another.

Example:

```text
Payment
--------------------------------
id: 7f8c...
sourceAccount: 1001
destinationAccount: 1002
amount: 100.00
currency: USD
status: CREATED
createdAt: 2026-10-06T10:30:00
```

Possible payment states include:

```text
CREATED
PROCESSING
AUTHORIZED
CAPTURED
SETTLED
FAILED
REFUNDED
```

Not every state transition is valid.

For example, a payment that has already been refunded should not be captured again.

The application must therefore ensure that payment state changes follow the defined lifecycle.

---

# Money

Financial amounts must not be represented using floating-point types.

For example, avoid using:

```java
double amount;
```

for monetary values.

Use an appropriate representation that prevents floating-point rounding errors.

The system should preserve the exact monetary value used in a transaction.

For example:

```text
100.10 + 0.20
```

must produce:

```text
100.30
```

rather than a floating-point approximation.

---

# API

The application should expose a REST API.

## Create Payment

```http
POST /payments
```

Example request:

```json
{
  "sourceAccountId": 1001,
  "destinationAccountId": 1002,
  "amount": 100.00,
  "currency": "USD"
}
```

Example response:

```json
{
  "id": "7f8c...",
  "status": "CREATED"
}
```

---

## Get Payment

```http
GET /payments/{paymentId}
```

The endpoint should return the current state of the payment.

---

## Refund Payment

```http
POST /payments/{paymentId}/refund
```

A payment may only be refunded when its current state allows it.

---

## Get Account

```http
GET /accounts/{accountId}
```

The endpoint should return the account's current balance and status.

---

# Idempotency

Payment requests must be idempotent.

Clients may provide an idempotency key:

```http
Idempotency-Key: 8f3c2d...
```

Consider the following sequence:

```text
Client
   │
   │ POST /payments
   │ Idempotency-Key: ABC
   ▼
Server
   │
   │ Payment created
   ▼
Network failure
```

The client may retry:

```text
Client
   │
   │ POST /payments
   │ Idempotency-Key: ABC
   ▼
Server
```

The second request must not create another payment.

The system should return the result associated with the original request.

The same idempotency key should not be allowed to represent different payment requests.

---

# Ledger

The application must maintain a financial ledger.

The ledger represents the financial history of the system independently from the current account balance.

When a payment transfers money:

```text
Account A → Account B
$100
```

the system should record corresponding financial entries.

For example:

```text
Ledger Entry
--------------------------------
paymentId: 7f8c...
accountId: 1001
type: DEBIT
amount: 100.00

Ledger Entry
--------------------------------
paymentId: 7f8c...
accountId: 1002
type: CREDIT
amount: 100.00
```

Every completed transfer must maintain the following invariant:

```text
Total Debits = Total Credits
```

A payment must not result in money appearing or disappearing from the system.

Ledger entries should be treated as historical records.

Once a financial transaction has been recorded, avoid modifying historical entries simply to make the current state look correct.

If a correction is necessary, consider how the correction itself can be represented as a new financial event.

---

# Account Balance

The account balance represents the current available balance.

For example:

```text
Account A
Balance: $500
```

After transferring `$100`:

```text
Account A
Balance: $400
```

and:

```text
Account B
Balance: $600
```

The system must not allow concurrent operations to produce an invalid balance.

Consider what should happen when two requests attempt to spend the same available funds at approximately the same time.

Example:

```text
Initial balance: $100

Request A: transfer $80
Request B: transfer $80
```

Only one transaction should be allowed to consume the available funds.

The database and application should work together to preserve this invariant.

---

# Transactions

Financial state changes must be atomic.

For example, consider:

```text
1. Debit source account
2. Create ledger entry
3. Credit destination account
4. Create ledger entry
5. Update payment status
```

If the application fails between steps 2 and 3, the system must not leave the database in a partially completed state.

Consider which operations should belong to the same database transaction and which operations can safely occur asynchronously.

---

# Event Processing

The application must use **Apache Kafka** for asynchronous communication between components.

At minimum, the system should publish payment-related events.

Example topics:

```text
payments
payment-results
ledger-events
```

You may introduce additional topics where appropriate.

A payment event might look like:

```json
{
  "eventId": "91ac...",
  "eventType": "PAYMENT_CREATED",
  "paymentId": "7f8c...",
  "timestamp": "2026-10-06T10:30:00",
  "sourceAccountId": 1001,
  "destinationAccountId": 1002,
  "amount": 100.00,
  "currency": "USD"
}
```

Events should contain enough information for consumers to process them without depending unnecessarily on another component's internal implementation.

---

# Kafka Consumers

Kafka consumers should process payment events asynchronously.

For example:

```text
                 Kafka
                   │
             payments topic
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Payment     Fraud      Audit
     Service     Service    Service
```

Different consumers may perform different responsibilities.

A failure in one consumer should not automatically prevent unrelated consumers from processing the event.

Consider how consumer groups should be configured.

---

# Message Ordering

Some financial events must be processed in the correct order.

For example:

```text
PAYMENT_CREATED
       ↓
PAYMENT_CAPTURED
       ↓
PAYMENT_REFUNDED
```

A refund must not be processed before the payment has been captured.

Consider how Kafka message keys and partitions can help preserve ordering for events belonging to the same payment.

---

# Duplicate Events

Kafka consumers must be able to handle duplicate messages safely.

Consider:

```text
PAYMENT_CREATED
PAYMENT_CREATED
```

being delivered to a consumer.

The system must not:

```text
Debit $100
Debit another $100
```

because the same event was processed twice.

Processing an event more than once should not corrupt the financial state.

---

# Retry Handling

Temporary failures may occur while processing an event.

For example:

```text
Payment Event
     ↓
Consumer
     ↓
Database unavailable
```

The application should retry failures that may be temporary.

However, not every failure should be retried indefinitely.

For example:

```text
Invalid payment
Malformed event
Non-existent account
```

may require a different treatment.

Define an appropriate retry policy.

---

# Dead Letter Queue

Messages that cannot be successfully processed after the configured number of attempts should not remain stuck indefinitely.

Consider using a dead-letter topic.

Example:

```text
payments
   │
   ▼
Consumer
   │
   ├── success ──────► processed
   │
   └── failure
          │
          ▼
        retry
          │
          ▼
        retry
          │
          ▼
       dead-letter
```

A dead-letter message should retain enough information to understand why processing failed.

---

# Failure Scenarios

The application should be designed with failures in mind.

Consider the following scenarios:

### Scenario 1 — Consumer crashes

```text
Payment event
     ↓
Consumer
     ↓
Application crashes
```

What happens when the consumer starts again?

---

### Scenario 2 — Duplicate request

```text
Client
   ↓
POST /payments
   ↓
Payment created

Client retries the request
   ↓
POST /payments
```

The customer must not be charged twice.

---

### Scenario 3 — Duplicate Kafka message

```text
PAYMENT_CREATED
PAYMENT_CREATED
```

The payment should only be processed once.

---

### Scenario 4 — Database failure

```text
Kafka
  ↓
Consumer
  ↓
Database unavailable
```

The event should not simply disappear.

---

### Scenario 5 — Insufficient funds

```text
Balance: $50

Payment: $100
```

The payment should fail without creating an invalid financial state.

---

### Scenario 6 — Concurrent payments

```text
Balance: $100

Payment A: $80
Payment B: $80
```

The system must prevent both payments from successfully spending the same funds.

---

# Refunds

The application should support refunds.

A refund reverses the financial effect of a previously completed payment.

For example:

```text
Original payment:

Account A ──$100──► Account B
```

A refund produces the opposite financial movement:

```text
Account B ──$100──► Account A
```

The original ledger entries should remain part of the historical record.

The refund should be represented as its own operation.

Consider what should happen when:

- A payment is already refunded.
- A payment has failed.
- A payment is still processing.
- A partial refund is requested.
- The refund amount exceeds the original payment.

---

# Audit Trail

Important financial operations should be auditable.

The system should record events such as:

```text
PAYMENT_CREATED
PAYMENT_AUTHORIZED
PAYMENT_CAPTURED
PAYMENT_FAILED
PAYMENT_REFUNDED
ACCOUNT_FROZEN
```

An audit record should contain information such as:

```text
eventId
eventType
entityId
timestamp
```

Audit records should provide a chronological history of important changes.

---

# Persistence

Use **PostgreSQL** as the primary relational database.

The database should persist:

- Accounts
- Payments
- Ledger entries
- Idempotency records
- Audit information

Database schema changes should be versioned.

Avoid relying on manually modifying the database when starting the application.

---

# Domain Boundaries

The application contains several areas of responsibility.

At minimum, consider separating responsibilities related to:

```text
Payment Processing
Account Management
Ledger
Event Processing
Audit
```

These areas may communicate through well-defined interfaces or events.

Avoid allowing one component to directly manipulate another component's internal state.

For example, a payment-processing component should not need to know how ledger entries are internally stored.

---

# Spring Framework

Use **Spring Boot** to build the application.

Use Spring components for:

- REST controllers
- Application services
- Persistence
- Kafka producers
- Kafka consumers
- Configuration

Dependencies should be provided to classes rather than created internally.

Configuration values should be externalized.

Example:

```properties
spring.datasource.url=...
spring.kafka.bootstrap-servers=...
```

Do not hard-code infrastructure configuration inside application logic.

---

# API Models

The classes exposed through the REST API do not necessarily need to be the same classes used internally by the domain or persistence layers.

For example:

```text
HTTP Request
     ↓
Request Model
     ↓
Application Logic
     ↓
Domain Model
     ↓
Persistence
```

Consider what information should be exposed to clients and what information should remain internal.

The API should not expose database implementation details unnecessarily.

---

# Validation

Validate incoming requests before processing them.

Examples:

```text
amount > 0
source account exists
destination account exists
currency is supported
source != destination
idempotency key is valid
```

Validation failures should produce appropriate HTTP responses.

Do not rely exclusively on the database to validate user input.

---

# Error Handling

The API should return meaningful HTTP responses.

For example:

```text
400 Bad Request
```

for invalid input.

```text
404 Not Found
```

when a requested resource does not exist.

```text
409 Conflict
```

when the requested operation conflicts with the current state.

```text
422 Unprocessable Entity
```

may be appropriate when the request is structurally valid but cannot be processed.

Errors should have a consistent response format.

Example:

```json
{
  "timestamp": "2026-10-06T10:30:00",
  "status": 409,
  "error": "INSUFFICIENT_FUNDS",
  "message": "The account does not have enough available funds."
}
```

---

# Testing

The application should contain automated tests.

At minimum, test:

### Unit tests

- Payment validation
- Payment state transitions
- Balance calculations
- Ledger calculations
- Idempotency behavior
- Refund rules
- Event processing logic

### Integration tests

Test interactions with:

- PostgreSQL
- Kafka

Tests should verify actual behavior rather than only checking whether methods were called.

Consider using containers for infrastructure dependencies during integration tests.

---

# Important Invariants

The following conditions should always hold:

### Financial balance

An account cannot spend more money than its available balance.

### Ledger consistency

```text
Total Debits = Total Credits
```

for every completed transfer.

### Payment uniqueness

A single idempotency key must not create multiple payments.

### State validity

A payment cannot move arbitrarily between states.

For example:

```text
REFUNDED → CAPTURED
```

must not be allowed.

### Event processing

Processing the same financial event multiple times must not produce an incorrect financial result.

These invariants should be covered by automated tests.

---

# Observability

The application should provide enough information to diagnose problems.

Consider exposing:

- Application logs
- Payment identifiers
- Event identifiers
- Correlation identifiers
- Processing duration
- Failed message counts
- Consumer lag

Sensitive financial information should not be written to logs unnecessarily.

For example, avoid logging complete payment credentials or other confidential information.

---

# Configuration

The following configuration should be externalized:

```properties
spring.datasource.url
spring.datasource.username
spring.datasource.password

spring.kafka.bootstrap-servers

payment.retry.max-attempts
payment.retry.backoff

payment.currency.default
```

Environment-specific configuration should not require modifying application source code.

---

# Docker

The application should be runnable locally using Docker Compose.

The development environment should include:

```text
FluxPay
   │
   ├── Spring Boot application
   ├── PostgreSQL
   └── Kafka
```

A developer should be able to start the required infrastructure with a single command.

---

# Suggested Project Structure

The following is intentionally not a complete implementation structure.

Use it as a starting point and decide how responsibilities should be organized.

```text
src/main/java
└── com.example.FluxPay
    │
    ├── account
    │
    ├── payment
    │
    ├── ledger
    │
    ├── event
    │
    ├── audit
    │
    └── configuration
```

The final structure should reflect the responsibilities of the application rather than simply grouping every class by technical type.

---

# Engineering Requirements

When implementing the application:

- Prefer small classes with clear responsibilities.
- Avoid unnecessary coupling between components.
- Keep business rules independent from infrastructure concerns where practical.
- Prefer immutable data where appropriate.
- Avoid duplicating business rules.
- Make invalid states difficult to represent.
- Keep financial operations explicit and traceable.
- Consider what happens when every external dependency fails.
- Prefer behavior that can be tested independently.
- Use meaningful names rather than comments to explain obvious code.

The application should remain understandable as its functionality grows.

---

# Design Considerations

The requirements intentionally leave several implementation decisions open.

Before implementing them, consider:

- How should payment state transitions be represented?
- Where should payment business rules live?
- How should different payment operations share common behavior?
- How should Kafka events be converted into application behavior?
- How should duplicate events be detected?
- How should idempotency records be persisted?
- Which operations should be synchronous?
- Which operations should be asynchronous?
- Where should database transactions begin and end?
- How should concurrent balance updates be handled?
- How should retryable and non-retryable failures be distinguished?
- What should happen when an event is successfully published but database processing fails?
- What should happen when database processing succeeds but publishing the next event fails?
- How can components evolve without requiring changes throughout the application?

There may be several valid solutions.

The goal is not to make every component as abstract as possible, but to create a design where responsibilities are clear and changes can be made without unnecessarily affecting unrelated parts of the system.

---

# Acceptance Criteria

The application is considered complete when:

- Payments can be created through the REST API.
- Payments follow a valid lifecycle.
- Accounts maintain correct balances.
- Transfers produce balanced ledger entries.
- Duplicate payment requests do not create duplicate payments.
- Duplicate Kafka events do not corrupt financial state.
- Failed Kafka processing can be retried.
- Permanently failing messages can be moved to a dead-letter topic.
- Refunds are supported.
- Important operations are auditable.
- PostgreSQL persists application state.
- Kafka is used for asynchronous event processing.
- The application handles concurrent financial operations safely.
- Automated tests cover the most important business rules.
- The complete application can be started locally using Docker Compose.

---

# Optional Extensions

Once the core application is complete, consider extending it with:

- Partial refunds
- Multiple currencies
- Exchange rates
- Scheduled payments
- Account limits
- Fraud detection
- Kafka Streams
- Transaction reconciliation
- Rate limiting
- Authentication and authorization
- Prometheus metrics
- Grafana dashboards
- Distributed tracing
- API documentation

These extensions should only be implemented after the core financial invariants are reliable.

A system that processes ten features incorrectly is less useful than a system that processes three features correctly.

---

# Final Challenge

Before considering the application complete, try to answer the following questions about your implementation:

1. What happens if the same payment request arrives twice?
2. What happens if the same Kafka event is consumed twice?
3. What happens if the application crashes halfway through a transfer?
4. What happens if PostgreSQL becomes unavailable?
5. What happens if Kafka becomes unavailable?
6. What happens if two payments spend the same balance simultaneously?
7. What happens if a refund request arrives twice?
8. How can you determine exactly what happened to a payment?
9. Can the ledger be used to reconstruct an account's balance?
10. Which parts of the application could be changed without modifying the rest of the system?
11. Where are the most important business rules located?
12. Which components know too much about other components?
13. What happens when a new payment type is introduced?
14. What happens when the event format changes?
15. How would you test the system if Kafka and PostgreSQL were not available?

If these questions are difficult to answer, the implementation is probably not finished yet.

The objective is not simply to make the application work.

The objective is to make it **correct, testable, maintainable, and resilient when things go wrong.**
