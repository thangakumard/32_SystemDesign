# Java Multithreading — Quick Interview Guide

Multithreading means **running multiple tasks concurrently** within the same Java process.

For a Senior Software Engineer interview, focus on these concepts:

1. Thread creation
2. `Runnable` / `Callable`
3. `ExecutorService`
4. Synchronization
5. Race conditions
6. `volatile`
7. Locks
8. Atomic variables
9. `CompletableFuture`
10. Deadlock

---

## 1. Creating a Thread

The simplest approach is extending `Thread`.

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " +
                           Thread.currentThread().getName());
    }
}

public class Main {
    public static void main(String[] args) {
        MyThread t1 = new MyThread();

        t1.start(); // Creates a new thread
    }
}
```

### Important

```java
t1.start(); // New thread
t1.run();   // Normal method call - NOT a new thread
```

Think:

```text
start()
  ↓
JVM creates thread
  ↓
run() executes on that thread
```

---

## 2. Runnable

Usually prefer `Runnable` over extending `Thread`.

```java
Runnable task = () -> {
    System.out.println(
        "Running: " + Thread.currentThread().getName()
    );
};

Thread t1 = new Thread(task);
Thread t2 = new Thread(task);

t1.start();
t2.start();
```

Both threads execute the same task.

```text
Main Thread
    |
    +---- Thread-1 → task
    |
    +---- Thread-2 → task
```

---

## 3. ExecutorService — Most Important

In real applications, you generally **don't create threads manually**.

Use a thread pool.

```java
ExecutorService executor =
    Executors.newFixedThreadPool(3);

Runnable task = () -> {
    System.out.println(
        "Running: " + Thread.currentThread().getName()
    );
};

executor.submit(task);
executor.submit(task);
executor.submit(task);
executor.submit(task);

executor.shutdown();
```

You created **3 threads**, but submitted **4 tasks**.

The fourth task waits for a thread to become available.

```text
          Thread Pool
       ┌───────────────┐
Task 1 ─→ Thread-1     │
Task 2 ─→ Thread-2     │
Task 3 ─→ Thread-3     │
Task 4 ─→ waiting      │
       └───────────────┘
```

### Why thread pools?

Creating a thread is relatively expensive.

A pool allows you to:

- Reuse threads
- Control concurrency
- Avoid creating thousands of threads
- Queue excess tasks

---

## 4. Callable — When You Need a Result

`Runnable` doesn't return a result.

`Callable` does.

```java
ExecutorService executor =
    Executors.newFixedThreadPool(2);

Callable<Integer> task = () -> {
    return 10 + 20;
};

Future<Integer> future = executor.submit(task);

Integer result = future.get();

System.out.println(result); // 30

executor.shutdown();
```

`future.get()` waits until the task finishes.

---

## 5. Race Condition

This is one of the most important multithreading concepts.

Suppose:

```java
class Counter {
    int count = 0;

    void increment() {
        count++;
    }
}
```

And two threads execute:

```java
Counter counter = new Counter();

Thread t1 = new Thread(() -> {
    for (int i = 0; i < 1000; i++)
        counter.increment();
});

Thread t2 = new Thread(() -> {
    for (int i = 0; i < 1000; i++)
        counter.increment();
});

t1.start();
t2.start();
```

You might expect:

```text
count = 2000
```

But you may get a smaller value.

Why?

Because:

```java
count++;
```

is actually approximately:

```text
READ count
ADD 1
WRITE count
```

Two threads can interfere.

```text
Thread 1             Thread 2

READ 10
                     READ 10
ADD 1
                     ADD 1
WRITE 11
                     WRITE 11

Expected: 12
Actual:   11
```

This is a **race condition**.

---

## 6. synchronized

The simplest solution:

```java
class Counter {
    private int count = 0;

    synchronized void increment() {
        count++;
    }

    int getCount() {
        return count;
    }
}
```

Now only one thread can execute `increment()` at a time.

```text
Thread 1 → LOCK → increment → UNLOCK

Thread 2 → waits

Thread 2 → LOCK → increment → UNLOCK
```

You can also synchronize a block:

```java
void increment() {

    synchronized (this) {
        count++;
    }
}
```

This is useful when only a small portion of the method needs protection.

---

## 7. AtomicInteger

For simple counters, `AtomicInteger` is often better.

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();

System.out.println(count.get());
```

Example:

```java
ExecutorService executor =
    Executors.newFixedThreadPool(4);

AtomicInteger counter = new AtomicInteger();

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> counter.incrementAndGet());
}

executor.shutdown();
```

Atomic operations provide thread safety without explicitly using `synchronized`.

Common atomic classes:

```text
AtomicInteger
AtomicLong
AtomicBoolean
AtomicReference
```

---

## 8. volatile

`volatile` is about **visibility**, not atomicity.

```java
class Worker {

    private volatile boolean running = true;

    void stop() {
        running = false;
    }

    void work() {
        while (running) {
            // do work
        }
    }
}
```

Without `volatile`, one thread might not immediately see another thread's update.

With:

```java
volatile boolean running;
```

changes are visible across threads.

### Important interview distinction

```text
volatile
   ↓
Visibility

synchronized
   ↓
Visibility + mutual exclusion

AtomicInteger
   ↓
Atomic operations
```

Don't use `volatile` to solve:

```java
count++;
```

because `count++` is not atomic.

---

## 9. Lock

Java provides explicit locks through `java.util.concurrent.locks`.

```java
Lock lock = new ReentrantLock();

void increment() {

    lock.lock();

    try {
        count++;
    } finally {
        lock.unlock();
    }
}
```

Why `finally`?

Because you must release the lock even if an exception occurs.

```text
lock()
  ↓
critical section
  ↓
unlock()
```

### How `ReentrantLock` Provides More Control

`synchronized` is simple and often the right choice, but `ReentrantLock` provides additional operations for situations where you need more control over acquiring and managing a lock.

#### 1. `tryLock()` — Don't Wait Indefinitely

With `synchronized`, if another thread owns the monitor, the current thread waits.

With `ReentrantLock`, you can try to acquire the lock and immediately continue if it isn't available:

```java
ReentrantLock lock = new ReentrantLock();

if (lock.tryLock()) {
    try {
        // Critical section
        updateData();
    } finally {
        lock.unlock();
    }
} else {
    // Lock wasn't available
    handleBusyResource();
}
```

You can also specify a timeout:

```java
if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
    try {
        updateData();
    } finally {
        lock.unlock();
    }
} else {
    // Couldn't acquire lock within 500 ms
    handleTimeout();
}
```

This is useful when you don't want a thread to wait forever.

#### 2. Interruptible Lock Acquisition

`lockInterruptibly()` allows a waiting thread to respond to interruption:

```java
try {
    lock.lockInterruptibly();

    try {
        updateData();
    } finally {
        lock.unlock();
    }

} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

This can be useful when an application needs to cancel work while a thread is waiting for a lock.

#### 3. Fairness

A `ReentrantLock` can optionally use a fairness policy:

```java
ReentrantLock fairLock =
    new ReentrantLock(true);
```

The `true` requests a fair lock, meaning threads waiting for the lock are generally granted access in waiting order.

This can reduce the possibility of a thread repeatedly being bypassed, although fairness can have a performance cost.

### Quick Comparison

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Mutual exclusion | Yes | Yes |
| Automatic unlock | Yes | No |
| `tryLock()` | No | Yes |
| Timed lock acquisition | No | Yes |
| Interruptible acquisition | No | Yes |
| Fairness option | No | Yes |
| Simpler syntax | Yes | No |

**Interview rule of thumb:** Use `synchronized` when simple mutual exclusion is enough. Consider `ReentrantLock` when you specifically need features such as `tryLock()`, timed acquisition, interruptible acquisition, or configurable fairness.

---

## 10. CompletableFuture

Very useful for asynchronous programming.

```java
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> {
        return 10 + 20;
    });

future.thenAccept(result ->
    System.out.println("Result = " + result)
);
```

You can chain operations:

```java
CompletableFuture
    .supplyAsync(() -> getUser())
    .thenApply(user -> getOrders(user))
    .thenAccept(orders ->
        System.out.println(orders)
    );
```

Conceptually:

```text
getUser()
   ↓
getOrders()
   ↓
process orders
```

without blocking the main thread.

---

## 11. Running Tasks in Parallel

Suppose you need to call two independent services:

```text
Service A → 2 seconds
Service B → 3 seconds
```

Sequential:

```java
getA(); // 2 sec
getB(); // 3 sec
```

Total ≈ **5 seconds**

Parallel:

```java
CompletableFuture<A> a =
    CompletableFuture.supplyAsync(() -> getA());

CompletableFuture<B> b =
    CompletableFuture.supplyAsync(() -> getB());

CompletableFuture.allOf(a, b).join();
```

Total ≈ **3 seconds**.

This is a very common backend-system use case.

---

## 12. Deadlock

A deadlock occurs when threads wait for each other forever.

```text
Thread 1:
lock(A)
wait for B

Thread 2:
lock(B)
wait for A
```

```text
Thread 1                  Thread 2

LOCK A                    LOCK B
  ↓                         ↓
WAIT B  ←──────────────→  WAIT A
```

Neither can continue.

### Avoiding deadlocks

A common technique is **consistent lock ordering**:

```java
lock(A);
lock(B);
```

Always acquire locks in the same order.

---

## 13. Thread Lifecycle

Know this diagram:

```text
NEW
 ↓
RUNNABLE
 ↓
RUNNING
 ↓
BLOCKED / WAITING / TIMED_WAITING
 ↓
RUNNABLE
 ↓
TERMINATED
```

For example:

```java
Thread.sleep(1000);
```

puts the thread into:

```text
TIMED_WAITING
```

Whereas:

```java
thread.join();
```

causes the calling thread to wait for another thread.

---

## 14. join()

```java
Thread t1 = new Thread(() -> {
    System.out.println("Task completed");
});

t1.start();

t1.join();

System.out.println("Main continues");
```

The main thread waits until `t1` finishes.

```text
Main
 |
 | start t1
 ↓
waiting
 |
 | ← t1 finishes
 ↓
continue
```

---

## 15. Interview Cheat Sheet

| Concept | Purpose |
|---|---|
| `Thread` | Create/manage a thread |
| `Runnable` | Task without return value |
| `Callable` | Task with return value |
| `ExecutorService` | Manage thread pools |
| `Future` | Get asynchronous result |
| `synchronized` | Mutual exclusion + visibility |
| `volatile` | Visibility |
| `AtomicInteger` | Thread-safe atomic operations |
| `ReentrantLock` | Explicit locking/control |
| `CompletableFuture` | Async composition |
| `sleep()` | Pause current thread |
| `join()` | Wait for another thread |
| Deadlock | Threads waiting on each other |
| Race condition | Result depends on execution order |

---

## Most Important Mental Model

When you see a multithreading problem, ask:

```text
1. What data is shared?
        ↓
2. Can multiple threads modify it?
        ↓
3. Is the operation atomic?
        ↓
4. If not, how do I protect it?
        ↓
   synchronized / Lock / Atomic*
        ↓
5. Do I need visibility?
        ↓
      volatile
        ↓
6. Do I need controlled concurrency?
        ↓
   ExecutorService
        ↓
7. Do I need asynchronous composition?
        ↓
   CompletableFuture
```

For a **Google Senior SWE interview**, the next important step is learning `ExecutorService` + `synchronized` + `Lock` + `AtomicInteger` through several coding problems, because that's where multithreading concepts become practical.
