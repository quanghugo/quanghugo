---
title: When Virtual Threads Fail During Rolling Deployments
description: How in-memory timeout threads can cause production issues during Kubernetes rolling deployments, and why we need persistent timeout mechanisms.
slug: rolling-virtual-thread
date: 2025-12-30 00:00:00+0000
categories:
    - Design System
    - Virtual Thread
    - Redis
    - Production Issues
    - Syntax
# weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## The Problem

Systems that rely on in-memory timeout mechanisms can encounter critical issues during rolling deployments. When a deployment occurs while timeout operations are in progress, these operations may be terminated before completion, leading to state inconsistencies and system failures.

Consider, for example, an e-commerce order processing system. When a customer places an order, the system attempts to process their payment confirmation. The system enforces a strict 60-second timeout window to receive payment confirmation. If payment confirmation is not received within this timeframe, the system should automatically return a "Payment Timeout" response and update the order status accordingly.

Under normal operating conditions, this mechanism works perfectly. However, during rolling deployments, some orders may exceed their timeout limits by 2-3x, creating a dangerous inconsistency between what the client displayed and the actual backend state. Orders that should have timed out after 60 seconds may remain in a **PENDING_PAYMENT** status for 120-180 seconds, causing inconsistent system behavior and user confusion.

---

## Understanding the System Architecture

### Virtual Threads Overview

Many systems leverage **Java Virtual Threads** (introduced in Java 19 as a preview and finalized in Java 21) to handle timeout mechanisms efficiently. Virtual threads are lightweight threads managed by the Java Virtual Machine, allowing applications to create millions of threads without the overhead of traditional platform threads.

For timeout implementations, systems typically spawn a virtual thread that sleeps for the duration of the timeout period (e.g., 60 seconds). When the timeout expires, the thread executes a callback function that handles the timeout scenario.

### The Timeout Mechanism

To illustrate this, let's examine an e-commerce order processing system. In such a system, the timeout flow works as follows:

1. **Order Creation**: When a customer places an order, the system creates an order record with status `PENDING_PAYMENT`
2. **Timer Initialization**: A virtual thread is spawned to handle the timeout countdown
3. **Payment Processing**: The system attempts to process and confirm the payment
4. **Success Path**: If payment is confirmed, the timer thread is interrupted and the order status is updated to `CONFIRMED`
5. **Timeout Path**: If payment confirmation is not received within 60 seconds, the timer thread executes the timeout callback, updating the order status to `PAYMENT_TIMEOUT`

This design works reliably in stable environments, but reveals a critical flaw during deployment scenarios.

---

## The Deployment Failure Scenario

During a rolling deployment in Kubernetes, systems using in-memory timeout mechanisms can experience failures. Consider a scenario where an e-commerce system is processing orders during a deployment. The following sequence of events occurs:

```mermaid

    participant User as Customer
    participant Pod as Service Pod
    participant K8s as Kubernetes
    participant DB as Database

    User->>Pod: Place order request
    Pod->>DB: Create order (PENDING_PAYMENT)
    Pod->>Pod: Spawn virtual thread<br/>sleep(60s) → timeout callback
    Note over Pod: Timer counting: 59s, 58s, 57s...
    
    K8s->>Pod: SIGTERM signal
    Pod->>Pod: JVM shutdown initiated
    Pod->>Pod: All threads terminated
    Note over Pod: Timeout thread killed<br/>before callback executes
    
    Pod->>DB: Pod terminates
    Note over DB: Order remains in PENDING_PAYMENT state
    Note over User: Client shows timeout<br/>Backend still waiting
```

### What Happens During Rolling Deployment

In an e-commerce system, this failure scenario unfolds as follows:

1. **Normal Operation**: An order is created and a virtual thread starts the 60-second countdown
2. **Deployment Triggered**: Kubernetes initiates a rolling deployment, sending a `SIGTERM` signal to the pod
3. **JVM Shutdown**: The Java Virtual Machine begins its shutdown sequence
4. **Thread Termination**: All threads, including the timeout virtual thread, are terminated immediately
5. **Callback Never Executes**: The timeout callback that should update the order status never fires
6. **State Inconsistency**: The order remains in `PENDING_PAYMENT` status indefinitely, while the client has already shown a timeout error

### Code Example

An example implementation in an e-commerce system might look like this:

```java
public void createOrder(OrderRequest request) {
    // Create order in database
    Order order = orderRepository.save(new Order(request, Status.PENDING_PAYMENT));
    
    // Spawn virtual thread for timeout
    Thread.ofVirtual().start(() -> {
        try {
            Thread.sleep(60_000); // 60 seconds
            handlePaymentTimeout(order.getId());
        } catch (InterruptedException e) {
            // Payment confirmed, timer interrupted - this is the happy path
            Thread.currentThread().interrupt();
        }
    });
    
    // Attempt to process payment
    processPaymentConfirmation(order.getId());
}

private void handlePaymentTimeout(Long orderId) {
    // This callback never executes if pod is killed during deployment
    orderRepository.updateStatus(orderId, Status.PAYMENT_TIMEOUT);
}
```

The critical issue is that `handlePaymentTimeout()` never executes if the JVM is terminated before the 60-second sleep completes. This problem is not unique to e-commerce systems—any system relying on in-memory timeout mechanisms faces the same risk.

---

## Why This Is Dangerous

This failure mode creates a dangerous inconsistency between client and server states. For instance, in an e-commerce system, this might manifest as:

| Client View | Backend Reality |
|-------------|-----------------|
| Shows "Order failed" or timeout error | Order still in `PENDING_PAYMENT` status |
| User creates a new order | Original order still active in database |
| User assumes order failed | System still waiting for payment confirmation |
| Payment confirmed late (after timeout) | Conflict: order already considered failed by client |

### Real-World Impact

This inconsistency leads to several critical problems across different system types. As an example, in an e-commerce system:

- **Ghost Orders**: Orders that appear failed to users but remain active in the system
- **Double Payment Processing**: A payment might be confirmed for an order that the user believes has already failed, leading to confusion
- **Data Integrity Issues**: The system state becomes inconsistent, making it difficult to track actual order status
- **User Trust Erosion**: Users experience unreliable behavior, damaging confidence in the platform
- **Debugging Complexity**: Production issues become difficult to trace because the failure is silent and state-dependent

Similar issues can occur in any system that relies on in-memory timeout mechanisms—ride-sharing platforms, job scheduling systems, booking systems, and more all face the same fundamental risk.

---

## Root Cause Analysis

The fundamental problem is architectural: **systems that rely entirely on in-memory timeout mechanisms lack persistence or recovery strategy**.

### Key Issues

1. **No State Persistence**: The timeout state exists only in memory. When the JVM terminates, this state is lost forever.

2. **No Graceful Shutdown Handling**: Applications don't handle shutdown signals to complete pending timeout operations before termination.

3. **No Recovery Mechanism**: There's no background process to detect and recover operations that should have timed out but didn't. In an e-commerce system, for example, orders that should have timed out remain in an intermediate state.

4. **Tight Coupling**: The timeout logic is tightly coupled to the application lifecycle. When the application dies, the timeout dies with it.

### Why Virtual Threads Don't Help Here

While virtual threads are excellent for handling many concurrent operations efficiently, they don't solve the fundamental problem of persistence. Virtual threads are still in-memory constructs that disappear when the JVM terminates. The solution requires moving the timeout mechanism outside of the application's memory space.

---

## The Solution: Persistent Timeout Mechanism

In our next article, we'll explore how to solve this problem by implementing a **persistent timeout mechanism using Redis Sorted Sets (ZSET)** combined with a **recovery worker pattern**.

This approach ensures that:

- Timeout state is persisted outside the application memory
- Timeouts survive pod restarts and deployments
- A dedicated worker process can recover and process missed timeouts
- Systems maintain consistency even during infrastructure changes

The solution leverages Redis ZSET's ability to store timeout timestamps as scores, allowing efficient querying of expired timeouts, while a background worker continuously processes these timeouts regardless of which pod originally created them. This pattern applies to e-commerce systems, ride-sharing platforms, job schedulers, and any other system requiring reliable timeout handling.

Stay tuned for the detailed implementation in the next post!
