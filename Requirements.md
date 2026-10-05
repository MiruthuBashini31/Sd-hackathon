# SALESTORM – Requirements Assessment and Documentation

## 1. Introduction

SALESTORM is a high-scale flash-sale e-commerce platform designed to handle a situation where a very large number of customers attempt to purchase a limited quantity of products at almost the same time.

The primary scenario considered in this system is a flash sale in which approximately 10,000 customers may attempt to purchase a product while only 100 units are available. The system must remain reliable, responsive and consistent even under extreme concurrency.

The most important requirement is that the system must never sell more inventory than is actually available. If only 100 units exist, the platform must allow at most 100 successful purchases. This requirement becomes challenging when thousands of requests arrive simultaneously.

The system must therefore combine high scalability with strong consistency at critical points. It must also handle duplicate requests, payment failures, service failures, reservation expiration, network timeouts and other real-world failure scenarios.

The goal of this document is to identify, analyze and organize the requirements necessary to design and implement the SALESTORM platform.

---

# 2. Problem Statement

Traditional e-commerce systems are generally designed for normal traffic patterns. During a flash sale, however, traffic can increase dramatically within seconds.

For example:

* Available inventory: 100 units
* Potential customers: 10,000
* Requests: thousands arriving almost simultaneously
* Successful purchases allowed: maximum 100

The main challenge is not simply processing requests quickly. The system must process them **correctly**.

A poorly designed system may allow two customers to purchase the same final unit because both requests read the inventory before either request updates it.

Therefore, SALESTORM must guarantee:

1. No inventory overselling.
2. No duplicate reservations.
3. No duplicate payments.
4. Reliable order creation.
5. Safe recovery from failures.
6. Scalability under extreme traffic.
7. Secure processing of customer and payment information.

---

# 3. Objectives

The major objectives of SALESTORM are:

### 3.1 Inventory Accuracy

The inventory count must always represent the actual business state.

If 100 units exist, the system must never create more than 100 successful reservations or completed sales.

### 3.2 High Concurrency

The system must support thousands of customers attempting to purchase simultaneously.

### 3.3 Low Response Time

Customers should receive quick responses for operations such as:

* Product viewing
* Cart operations
* Reservation requests
* Order status queries

### 3.4 Reliability

Failures in individual services should not cause data corruption or loss of successful transactions.

### 3.5 Scalability

The platform should support horizontal scaling so that additional application instances can be added as traffic increases.

### 3.6 Security

Customer authentication, authorization, payment communication and sensitive information must be protected.

### 3.7 Observability

Operations teams should be able to identify failures, latency problems, inventory issues and payment problems through logs, metrics and tracing.

---

# 4. Scope

## 4.1 In Scope

The SALESTORM system includes:

* Customer management
* Product discovery
* Product catalogue
* Shopping cart
* Inventory management
* Inventory reservation
* Checkout
* Payment processing
* Order creation
* Order tracking
* Shipment management
* Notifications
* Authentication and authorization
* Concurrency management
* Idempotency
* Failure recovery
* Monitoring and logging
* Scalability
* Security

## 4.2 Out of Scope

The following are considered outside the core scope:

* Physical warehouse automation
* Manufacturing
* Supplier management
* Accounting and taxation systems
* Advanced recommendation engines
* Full customer relationship management
* Physical delivery operations

External systems may be represented as integrations where required.

---

# 5. Stakeholders

The major stakeholders are:

### Customer

The customer browses products, adds products to a cart, attempts a purchase, makes payment and tracks the resulting order.

### Administrator

The administrator manages products, inventory, deals and system configuration.

### Payment Provider

The external payment provider processes customer payments and returns payment status.

### Shipping Provider

The shipping provider handles shipment creation, tracking and delivery updates.

### Operations Team

The operations team monitors system health, failures, performance and security events.

### Development Team

The development team designs, implements, tests and maintains the system.

---

# 6. Functional Requirements

## FR-01: Customer Registration

The system shall allow customers to create accounts.

Customer information may include:

* Customer ID
* Name
* Email
* Phone number
* Address
* Authentication information

Passwords must never be stored as plain text.

---

## FR-02: Customer Authentication

The system shall authenticate customers before allowing protected operations.

Examples include:

* Viewing personal orders
* Making purchases
* Managing carts
* Viewing payment information

Authentication tokens should be used for subsequent requests.

---

## FR-03: Product Discovery

Customers shall be able to:

* View products
* Search products
* Filter products
* View product details
* View product availability

Product information should preferably be served through a cache when appropriate to reduce database load.

---

## FR-04: Shopping Cart

Customers shall be able to:

* Add products to the cart
* Remove products
* Change quantity
* View cart contents
* Proceed to checkout

The cart should not be considered the final inventory reservation unless the business rules explicitly require it.

---

## FR-05: Inventory Management

The system shall maintain inventory information for each product.

Inventory should distinguish between:

* Available quantity
* Reserved quantity
* Sold quantity

For example:

```text
Total Inventory = 100

Available = 80
Reserved = 15
Sold = 5
```

The system must maintain the relationship:

```text
Available + Reserved + Sold = Total Inventory
```

---

# 7. Inventory Reservation Requirements

Inventory reservation is one of the most critical components of SALESTORM.

When a customer attempts to purchase an item, the system should first reserve inventory.

A successful reservation changes the inventory state:

```text
AVAILABLE
     ↓
RESERVED
```

After successful payment:

```text
RESERVED
     ↓
CONFIRMED
     ↓
SOLD
```

If payment fails:

```text
RESERVED
     ↓
RELEASED
     ↓
AVAILABLE
```

If the reservation expires:

```text
RESERVED
     ↓
EXPIRED
     ↓
RELEASED
```

The reservation must have an expiration time to prevent inventory from remaining blocked indefinitely.

---

# 8. Concurrency Requirements

Concurrency is the central technical challenge of SALESTORM.

Consider the following situation:

```text
Inventory = 1

Customer A → Check inventory
Customer B → Check inventory
```

If both customers see one available item before either update occurs, both may attempt to purchase it.

Therefore, the following approach is unsafe:

```text
1. Read inventory
2. Check if quantity > 0
3. Decrease inventory
```

Instead, the system should perform the validation and update atomically.

A suitable logical operation is:

```text
Decrease inventory only if available_quantity > 0
```

The operation should return success only when the inventory update was actually performed.

This prevents two concurrent requests from successfully claiming the same inventory unit.

---

# 9. Idempotency Requirements

The system must support idempotency for operations that may be retried.

Examples include:

* Inventory reservation
* Payment
* Order creation

Suppose a customer sends:

```text
POST /reservation
Idempotency-Key: ABC123
```

Because of a network problem, the request is sent again with the same key.

The system must recognize:

```text
ABC123 = already processed
```

and return the original result instead of creating another reservation.

This protects the system against:

* Double-clicks
* Network retries
* Client retries
* Gateway retries
* Duplicate messages

---

# 10. Checkout Requirements

The checkout process should validate:

1. Customer identity
2. Cart contents
3. Product availability
4. Reservation status
5. Pricing
6. Discounts
7. Payment details

The system should create a consistent checkout transaction before payment is initiated.

Checkout should not directly assume that a product is available merely because it was available when the product page was viewed.

The authoritative inventory service must confirm the reservation.

---

# 11. Payment Requirements

The payment subsystem must support:

### Successful Payment

```text
Reservation
     ↓
Payment
     ↓
SUCCESS
     ↓
Order Confirmation
```

### Failed Payment

```text
Reservation
     ↓
Payment
     ↓
FAILED
     ↓
Release Reservation
```

### Payment Timeout

A timeout does not necessarily mean that the payment failed.

The payment provider may have processed the payment while the response was lost.

Therefore:

```text
Payment Request
      ↓
Timeout
      ↓
Payment Status Check
      ↓
SUCCESS / FAILED / UNKNOWN
```

The system must not blindly retry a payment operation in a way that could charge the customer twice.

---

# 12. Order Requirements

An order should only become confirmed after the required payment and reservation conditions are satisfied.

A possible order lifecycle is:

```text
CREATED
   ↓
PAYMENT_PENDING
   ↓
CONFIRMED
   ↓
PROCESSING
   ↓
SHIPPED
   ↓
OUT_FOR_DELIVERY
   ↓
DELIVERED
```

Failure states should also be supported where appropriate.

The system must prevent invalid transitions.

For example:

```text
DELIVERED → PAYMENT_PENDING
```

should not be allowed.

---

# 13. Payment-to-Order Failure Scenario

One important failure scenario is:

```text
Customer
   ↓
Payment
   ↓
SUCCESS
   ↓
Order Service
   ↓
FAILURE
```

The customer has already paid, so the system cannot simply discard the transaction.

A reliable solution is to publish a payment-success event:

```text
Payment Service
      ↓
PaymentSuccessful Event
      ↓
Message Queue
      ↓
Order Service
```

If Order Service is temporarily unavailable, the message remains available for later processing.

This approach reduces the possibility of losing successful payments.

---

# 14. Reservation Expiration

Reservations should have a time limit.

For example:

```text
Reservation ID: R1001
Created: 10:00
Expires: 10:05
```

If payment is not completed by 10:05:

```text
R1001
 ↓
EXPIRED
 ↓
RELEASE INVENTORY
```

The inventory becomes available to other customers.

The expiration process should be safe to execute multiple times without corrupting inventory.

---

# 15. Non-Functional Requirements

## NFR-01: Scalability

The platform must support horizontal scaling.

Application services should be stateless where possible so multiple instances can run behind a load balancer.

```text
Load Balancer
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
API  API  API
```

---

## NFR-02: Performance

Frequently accessed information should be cached.

Examples:

* Product information
* Product images
* Category data
* Non-critical availability information

However, cached inventory information must not be treated as the final authority when making a reservation.

---

## NFR-03: Availability

Critical services should avoid single points of failure.

Multiple application instances and appropriate database/high-availability mechanisms should be considered.

---

## NFR-04: Consistency

Strong consistency is particularly important for:

* Inventory reservation
* Payment state
* Order state

Eventual consistency may be acceptable for:

* Notifications
* Analytics
* Some product display information

---

## NFR-05: Reliability

The system should recover from temporary failures without losing important business transactions.

Retry mechanisms should be controlled and should not create duplicate operations.

---

## NFR-06: Security

The system must use:

* HTTPS
* Authentication
* Authorization
* Input validation
* Rate limiting
* Secure secret storage
* Audit logging
* Secure payment integration

Sensitive payment information should not be unnecessarily stored by the application.

---

# 16. Availability and Resilience

SALESTORM should assume that failures will occur.

Possible failures include:

* Payment provider unavailable
* Database temporarily unavailable
* Order service unavailable
* Network timeout
* Message processing failure
* Cache failure
* High traffic spike

The architecture should isolate failures instead of allowing one failing service to bring down the entire platform.

---

# 17. Retry Requirements

Retries should be used carefully.

A retry is appropriate for temporary failures such as:

```text
Timeout
Temporary network error
Temporary service unavailability
```

However, retries should use:

* Exponential backoff
* Maximum retry count
* Idempotency
* Dead-letter handling where appropriate

A retry without idempotency can create duplicate transactions.

---

# 18. Circuit Breaker

External services such as payment and shipping providers may become unavailable.

A circuit breaker can prevent repeated requests to an unhealthy dependency.

The states are:

```text
CLOSED
   ↓
Repeated failures
   ↓
OPEN
   ↓
Stop requests
   ↓
HALF-OPEN
   ↓
Test request
   ↓
CLOSED
```

This protects the system from cascading failures.

---

# 19. Message Queue Requirements

Asynchronous communication should be used where immediate responses are not necessary.

Suitable operations include:

* Payment success events
* Order creation events
* Notification events
* Shipment updates
* Analytics events

Example:

```text
Payment
   ↓
Message Queue
   ↓
Order
   ↓
Notification
```

The queue provides buffering during traffic spikes and temporary downstream failures.

---

# 20. Database Requirements

The system may use a relational database for transactional information.

Important entities include:

### Customer

```text
customer_id
name
email
phone
address
created_at
```

### Product

```text
product_id
name
description
price
category_id
status
```

### Inventory

```text
inventory_id
product_id
available_quantity
reserved_quantity
sold_quantity
version
updated_at
```

### Reservation

```text
reservation_id
customer_id
product_id
quantity
status
expires_at
idempotency_key
created_at
```

### Order

```text
order_id
customer_id
status
total_amount
created_at
updated_at
```

### Payment

```text
payment_id
order_id
provider_reference
amount
status
created_at
updated_at
```

---

# 21. API Requirements

The system should expose REST or equivalent APIs.

Examples:

### Product

```text
GET /products
GET /products/{productId}
```

### Cart

```text
POST /cart/items
GET /cart
DELETE /cart/items/{itemId}
```

### Reservation

```text
POST /reservations
GET /reservations/{reservationId}
DELETE /reservations/{reservationId}
```

### Payment

```text
POST /payments
GET /payments/{paymentId}
```

### Order

```text
POST /orders
GET /orders/{orderId}
GET /orders
```

Sensitive operations must require authentication and authorization.

---

# 22. Security Requirements

Security must be applied at multiple layers.

### Network Security

All external communication should use HTTPS.

### Authentication

Customers must be authenticated before accessing protected resources.

### Authorization

A customer should only be able to access their own orders and payment information.

### Input Validation

All API inputs must be validated.

### Rate Limiting

Rate limits should prevent malicious clients from overwhelming APIs.

### Secrets

Database credentials, API keys and payment credentials must not be hard-coded.

### Audit Logs

Important operations should be recorded.

Examples:

```text
Reservation created
Payment completed
Order created
Inventory adjusted
Administrative inventory change
```

---

# 23. Observability Requirements

The system should provide three major forms of observability:

### Logs

Useful for investigating individual events.

### Metrics

Useful for understanding system health.

Important metrics include:

* Requests per second
* Response latency
* Error rate
* Reservation success rate
* Reservation failure rate
* Payment failure rate
* Queue depth
* Database performance

### Distributed Tracing

A single customer transaction should be traceable across services.

Example:

```text
API Gateway
   ↓
Checkout
   ↓
Inventory
   ↓
Payment
   ↓
Order
```

A common correlation/trace ID should allow engineers to follow the request.

---

# 24. Business Rules

The following rules should be enforced:

### BR-01

A product cannot be successfully reserved when available inventory is zero.

### BR-02

Successful reservations cannot exceed available inventory.

### BR-03

A reservation must have an expiration time.

### BR-04

Expired reservations must release inventory.

### BR-05

A payment request must be idempotent.

### BR-06

An order cannot be confirmed without the required successful payment state.

### BR-07

An order must not be created twice for the same successful transaction.

### BR-08

Customers cannot access another customer's private order information.

### BR-09

Inventory modifications must be auditable.

### BR-10

Invalid state transitions must be rejected.

---

# 25. Assumptions

The design makes the following assumptions:

1. Product information is relatively read-heavy.
2. Flash-sale traffic is significantly higher than normal traffic.
3. Inventory is limited and must be strongly controlled.
4. Payment is handled through an external provider.
5. Shipping may also be handled by an external provider.
6. Temporary service failures are expected.
7. Network requests may be duplicated.
8. Customers may refresh pages or click buttons multiple times.
9. Some operations can be asynchronous.
10. The inventory service is the authoritative source for reservation decisions.

---

# 26. Constraints

Major constraints include:

* Limited inventory
* High concurrency
* External payment dependencies
* Network unreliability
* Database transaction requirements
* Response-time expectations
* Security requirements
* Operational complexity
* Cost of infrastructure

The architecture must balance correctness, performance, complexity and cost.

---

# 27. Requirements Traceability Matrix

| ID     | Requirement           | Design Component                 | Validation          |
| ------ | --------------------- | -------------------------------- | ------------------- |
| FR-01  | Customer registration | Customer Service                 | API test            |
| FR-02  | Authentication        | Auth/API Gateway                 | Security test       |
| FR-03  | Product discovery     | Product Service/Cache            | Load test           |
| FR-04  | Cart                  | Cart Service                     | Functional test     |
| FR-05  | Inventory             | Inventory Service                | Consistency test    |
| FR-06  | Reservation           | Reservation Service              | Concurrency test    |
| FR-07  | Payment               | Payment Service                  | Integration test    |
| FR-08  | Order creation        | Order Service                    | Integration test    |
| FR-09  | Shipment              | Shipment Service                 | Workflow test       |
| FR-10  | Notification          | Notification Service             | Event test          |
| NFR-01 | Scalability           | Load Balancer/Horizontal Scaling | Load test           |
| NFR-02 | Performance           | Cache/Async Processing           | Performance test    |
| NFR-03 | Availability          | Redundancy                       | Failure test        |
| NFR-04 | Consistency           | Transactions/Concurrency Control | Race-condition test |
| NFR-05 | Reliability           | Queue/Retry                      | Failure injection   |
| NFR-06 | Security              | Gateway/Auth/HTTPS               | Security testing    |
| NFR-07 | Observability         | Logs/Metrics/Tracing             | Monitoring test     |

---

# 28. Acceptance Criteria

The solution is considered successful when:

### Inventory

* 100 available units result in no more than 100 successful reservations.
* Concurrent requests cannot oversell inventory.
* Expired reservations release inventory correctly.

### Idempotency

* Repeating the same reservation request does not create another reservation.
* Repeating the same payment request does not create another charge.
* Repeating order creation does not create duplicate orders.

### Payment

* Successful payment results in order processing.
* Failed payment releases the reservation.
* Payment timeout triggers safe status reconciliation.

### Reliability

* Temporary Order Service failure does not lose successful payment events.
* Messages can be retried safely.
* External service failures do not cause cascading system failure.

### Scalability

* Multiple application instances can process requests simultaneously.
* Traffic spikes can be absorbed using queues, caching and horizontal scaling.

### Security

* Unauthorized customers cannot access protected resources.
* Sensitive communication uses HTTPS.
* API inputs are validated.

---

# 29. Risk Assessment

| Risk                     | Impact   | Mitigation                        |
| ------------------------ | -------- | --------------------------------- |
| Inventory overselling    | Critical | Atomic concurrency control        |
| Duplicate payment        | Critical | Idempotency + reconciliation      |
| Order service failure    | High     | Queue + retry                     |
| Payment provider failure | High     | Circuit breaker                   |
| Traffic spike            | High     | Load balancing + caching + queue  |
| Database bottleneck      | High     | Scaling + indexing + optimization |
| Reservation leak         | Medium   | Expiration mechanism              |
| Duplicate messages       | Medium   | Idempotent consumers              |
| Security attack          | Critical | WAF + authentication + validation |
| Monitoring failure       | Medium   | Centralized observability         |

---

# 30. Recommended Technical Approach

The recommended architecture should use a combination of:

* API Gateway
* Load Balancer
* Stateless application services
* Inventory/Reservation Service
* Relational database for transactional data
* Cache for high-volume reads
* Message queue for asynchronous workflows
* Payment integration
* Order Service
* Shipment Service
* Notification Service
* Centralized logging
* Metrics
* Distributed tracing

The most critical design decision is to keep the inventory reservation operation atomic.

The system should not depend on:

```text
Read → Think → Update
```

as separate uncontrolled operations.

Instead, it should perform a safe conditional inventory update.

The conceptual operation is:

```text
IF available_quantity >= requested_quantity
THEN
    decrease available quantity
    increase reserved quantity
    create reservation
ELSE
    reject reservation
```

This ensures that the database becomes the final consistency boundary.

---

# 31. High-Level Transaction Flow

The complete purchase flow is:

```text
Customer
   ↓
API Gateway
   ↓
Checkout
   ↓
Inventory Reservation
   ↓
Atomic Inventory Update
   ↓
Reservation Created
   ↓
Payment
   ↓
Payment Successful
   ↓
Payment Event
   ↓
Message Queue
   ↓
Order Service
   ↓
Order Confirmed
   ↓
Shipment
   ↓
Notification
```

Failure example:

```text
Payment Failed
      ↓
Release Reservation
      ↓
Inventory Available
```

Another failure example:

```text
Payment Success
      ↓
Order Service Down
      ↓
Message Queue
      ↓
Retry
      ↓
Order Created
```

---

# 32. Scalability Assessment

The platform should scale different components independently.

Product browsing may receive millions of read requests while inventory reservation may receive a smaller number of highly critical write requests.

Therefore, scaling should not mean simply adding more database servers.

The architecture should use:

```text
CDN
 ↓
Cache
 ↓
Load Balancer
 ↓
Stateless Services
 ↓
Queue
 ↓
Database
```

This allows read-heavy traffic to be handled separately from transactional operations.

The inventory service should be carefully protected because it represents the business-critical consistency boundary.

---

# 33. Reliability Strategy

The system follows several reliability principles:

### Fail Fast

Reject impossible operations quickly, such as reservations when inventory is already zero.

### Retry Carefully

Retry only temporary failures and use idempotency.

### Isolate Failures

A failure in notifications should not prevent an order from being created.

### Asynchronous Processing

Use queues for operations that do not require an immediate response.

### Reconciliation

Periodically reconcile payment, reservation and order states to identify unusual situations.

### Observability

Monitor critical workflows continuously.

---

# 34. Final Requirement Assessment

The SALESTORM system is fundamentally a **high-concurrency transactional system**.

The most important requirements are therefore not the product catalogue or user interface. The highest-priority requirements are:

1. **Prevent inventory overselling.**
2. **Handle thousands of concurrent purchase attempts.**
3. **Prevent duplicate transactions.**
4. **Maintain correct payment and order state.**
5. **Recover safely from service failures.**
6. **Scale during traffic spikes.**
7. **Maintain security and observability.**

The architecture should prioritize correctness at the inventory and payment boundaries while using asynchronous processing for downstream workflows.

The central design principle is:

> **Fast traffic handling should be scalable, but the final inventory decision must be strongly consistent.**

For the 10,000-customer/100-unit scenario, the expected result is:

```text
10,000 purchase attempts
          ↓
Traffic Management
          ↓
Inventory Reservation
          ↓
100 successful reservations maximum
          ↓
Payment Processing
          ↓
Order Creation
          ↓
Fulfilment
```

The remaining customers should receive a controlled failure such as:

```text
OUT_OF_STOCK
```

rather than causing an inventory inconsistency.

---

# 35. Conclusion

SALESTORM requires an architecture capable of combining high throughput with strict transactional correctness.

The system must be designed around the reality that thousands of customers can make the same request simultaneously. The solution therefore cannot rely only on load balancing and horizontal scaling. It must also provide strong concurrency control at the inventory boundary.

Inventory reservation, idempotency, payment reconciliation, asynchronous messaging, retry mechanisms and state-based order management form the foundation of the design.

The most critical guarantee is simple:

**If 100 units are available, the system must never successfully sell more than 100 units.**

Everything else in the architecture should support this guarantee while maintaining good performance, availability, security and user experience.

A successful implementation should demonstrate not only that the system works during normal conditions, but also that it remains correct when requests are duplicated, services fail, payments time out, reservations expire and traffic increases dramatically.

Therefore, the final architecture should be evaluated against both **functional correctness** and **failure behavior**. A system that is extremely fast but occasionally sells the same item twice is not successful. Similarly, a system that is perfectly consistent but cannot handle flash-sale traffic is not sufficient.

The ideal SALESTORM solution balances:

**Scalability + Consistency + Reliability + Security + Observability**

with **inventory correctness** as the primary business guarantee.
