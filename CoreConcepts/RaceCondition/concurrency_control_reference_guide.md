# Concurrency Control in System Design: A Reference Guide

When multiple users or services attempt to read and modify the same data simultaneously, race conditions can occur, leading to data corruption, lost updates, or invalid states. Database systems use different concurrency control strategies to manage these challenges.

---

### Table of Contents
* [1. Pessimistic Locking](#1-pessimistic-locking)
  * [What is it?](#what-is-it)
  * [Implementation Example (SQL)](#implementation-example-sql)
  * [Pros & Cons](#pros--cons)
  * [When to Use](#when-to-use)
* [2. Avoiding Deadlocks in Pessimistic Locking](#2-avoiding-deadlocks-in-pessimistic-locking)
* [3. Optimistic Locking](#3-optimistic-locking)
  * [What is it?](#what-is-it-1)
  * [Implementation Example](#implementation-example)
  * [Pros & Cons](#pros--cons-1)
  * [When to Use](#when-to-use-1)
* [4. Conditional Writing (CAS)](#4-conditional-writing-cas)
  * [What is it?](#what-is-it-2)
  * [Examples](#examples)
  * [Relationship to Optimistic Locking](#relationship-to-optimistic-locking)
* [5. Summary Matrix](#5-summary-matrix)

---

## 1. Pessimistic Locking

### What is it?
Pessimistic locking operates on the assumption that **conflicts are highly likely**. When a transaction wants to read or modify a record, it locks the resource immediately so that no other transaction can read, update, or delete it until the lock is released.

### Implementation Example (SQL)
```sql
BEGIN TRANSACTION;
-- Lock the row immediately
SELECT balance FROM accounts WHERE user_id = 101 FOR UPDATE;

-- Perform business logic checks
UPDATE accounts SET balance = balance - 100 WHERE user_id = 101;
COMMIT;
```

### Pros & Cons
* **Pros:** Guaranteed consistency; eliminates race conditions; safe under high contention.
* **Cons:** Reduced concurrency (blocking); performance overhead; risk of deadlocks.

### When to Use
High-contention environments where data correctness is paramount (e.g., financial ledger transfers, high-demand seat reservations).

---

## 2. Avoiding Deadlocks in Pessimistic Locking

A deadlock occurs when two or more transactions hold locks that the other needs, creating a circular dependency. Strategies to prevent and mitigate deadlocks include:

1. **Enforce Strict, Global Lock Ordering:** Always acquire locks in a predictable, sorted order (e.g., always locking the lower account ID before the higher account ID).
2. **Minimize Transaction Duration:** Keep external calls (HTTP, messaging queues) outside transactions and narrow the locked boundary.
3. **Use Non-Blocking Clauses (`NOWAIT` / `SKIP LOCKED`):** Instead of hanging indefinitely, use `FOR UPDATE NOWAIT` to fail fast or `SKIP LOCKED` for worker queues.
4. **Implement Resilient Retries:** Catch deadlock errors at the application layer and retry with randomized exponential backoff and jitter.
5. **Ensure Fine-Grained Locking:** Target specific primary keys to use row-level locks instead of triggering table-level locks.

---

## 3. Optimistic Locking

### What is it?
Optimistic locking operates on the assumption that **conflicts are rare**. Transactions read data without locks, and verify whether data has changed right before committing the write.

### Implementation Example
Typically implemented using a `version` column:
```sql
UPDATE accounts 
SET balance = balance - 100, version = version + 1 
WHERE user_id = 101 AND version = 3;
```
If `0 rows` are updated, another transaction modified the row concurrently, requiring a retry or rollback.

### Pros & Cons
* **Pros:** High throughput; non-blocking; zero deadlocks.
* **Cons:** Retry overhead under heavy contention; potential starvation risk.

### When to Use
Low-contention environments with a high ratio of reads to writes (e.g., updating user profiles, collaborative document editing).

---

## 4. Conditional Writing (CAS)

### What is it?
Conditional writing (Compare-And-Swap) is a technique where a database write executes only if specific conditions evaluate to true *at the exact moment the write occurs*, packed into a single atomic statement.

### Examples
* **State Machine Guards:**
  ```sql
  UPDATE orders SET status = 'CANCELLED' WHERE order_id = 5001 AND status = 'PENDING';
  ```
* **Atomic Counters:**
  ```sql
  UPDATE inventory SET stock_count = stock_count - 1 WHERE item_id = 101 AND stock_count > 0;
  ```

### Relationship to Optimistic Locking
* **Conditional writing is the *mechanism* / database feature.**
* **Optimistic locking is the *design pattern*.** 
Every time you implement optimistic locking, you use conditional writing (checking version numbers), but conditional writing can also be used for independent tasks like state transitions and atomic counters.

---

## 5. Summary Matrix

| Feature | Pessimistic Locking | Optimistic Locking / Conditional Writing |
| :--- | :--- | :--- |
| **Assumption** | Conflicts are frequent | Conflicts are rare |
| **When is data locked?** | Immediately upon reading | Only during the final write commit |
| **Concurrency** | Low (transactions block each other) | High (non-blocking) |
| **Failure Mode** | Waits for locks (or times out/errors) | Returns 0 rows affected; requires retry |
| **Best Used For** | Financial transactions, inventory counts | User profiles, state transitions, counters |