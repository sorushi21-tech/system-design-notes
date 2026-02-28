# Java Multithreading & Concurrency

---

## Table of Contents

1. [What is Multithreading?](#1-what-is-multithreading)
2. [Creating and Running Threads](#2-creating-and-running-threads)
3. [Thread Lifecycle — All Thread States](#3-thread-lifecycle--all-thread-states)
4. [Thread Safety — The Core Problem](#4-thread-safety--the-core-problem)
5. [The `synchronized` Keyword](#5-the-synchronized-keyword)
6. [`volatile` and Atomic Classes](#6-volatile-and-atomic-classes)
7. [Advanced Locking — `java.util.concurrent.locks`](#7-advanced-locking--javautilconcurrentlocks)
8. [ExecutorService & Thread Pools](#8-executorservice--thread-pools)
9. [CompletableFuture — Async Programming](#9-completablefuture--async-programming)
10. [Synchronization Utilities](#10-synchronization-utilities)
11. [Concurrent Collections](#11-concurrent-collections)
12. [Classic Concurrency Problems](#12-classic-concurrency-problems)
13. [`wait()`, `notify()`, `notifyAll()`](#13-wait-notify-notifyall)
14. [ThreadLocal](#14-threadlocal)
---

## 1. What is Multithreading?

### 1.1 The Basic Concept

A program, by default, executes one instruction at a time — from top to bottom. This single execution path is called a **thread**. A program with only one such path is a **single-threaded** program.

**Multithreading** is the ability of a program to run multiple threads concurrently. Each thread follows its own execution path independently. 
- The operating system (OS) switches between threads rapidly, giving the illusion of parallel execution. 
- On a multi-core CPU, threads can genuinely run in parallel — one thread per core.
---

### 1.2 Process vs Thread

**Process:** When an application is launched (e.g., Chrome, IntelliJ), the OS creates a process. Each process gets its own **isolated memory space**. Two processes cannot directly read each other's memory.

**Thread:** A thread lives **inside** a process. Multiple threads within the same process share the **heap memory** (where objects are stored), but each thread has its own **call stack** (local variables and method calls).

| Feature       | Process                                   | Thread                                                |
|---------------|-------------------------------------------|-------------------------------------------------------|
| Memory        | Own separate memory space                 | Shares heap memory with sibling threads               |
| Creation cost | Heavy — OS allocates memory and resources | Lightweight — needs only a stack and PC register      |
| Communication | Complex — needs IPC (sockets, pipes)      | Easy — threads share heap objects directly            |
| Crash impact  | One process crash does not affect others  | One thread crash can crash the entire process         |
| Example       | Chrome tabs run as separate processes     | Each HTTP request in Tomcat runs on a separate thread |

> 🎯 ** Question: Why does Java use threads instead of processes for concurrency?**
>
> - Threads share the same heap memory within a process, so inter-thread communication is fast and simple — threads can read and write shared objects directly. 
> - Creating a thread is also far cheaper than creating a new process. 
> - In web servers like Tomcat, each incoming HTTP request is assigned a thread from a thread pool, not a new process.

---

### 1.3 Why is Multithreading Needed?


1. **Better utilization of multi-core CPUs** 
   - Modern machines have 4, 8, or even 64 CPU cores. 
   - A single-threaded application uses only one core. 
   - Multithreading distributes work across all available cores, improving throughput significantly.

2. **Avoiding blocking on I/O** 
   - When a thread calls a database or makes an HTTP request, it waits for the response (e.g., 200 ms). During that wait, the thread is idle. 
   - With multithreading, other threads continue processing while one thread waits. This is critical for high-throughput servers.

---

## 2. Creating and Running Threads

Java provides four ways to define and run a thread. Each approach has an appropriate use case.

---

### 2.1 Way 1 — Extend the `Thread` Class

- A class extends `Thread` and overrides the `run()` method. The `run()` method contains the code the new thread will execute. 
- The thread is started by calling `start()`, **not** `run()` directly.

```java
class PrintTask extends Thread {
    private String message;

    PrintTask(String message) {
        this.message = message;
    }

    @Override
    public void run() {
        // Code inside run() executes on the new thread
        for (int i = 0; i < 3; i++) {
            System.out.println(message + " — " + Thread.currentThread().getName());
        }
    }
}

// Starting two threads
PrintTask t1 = new PrintTask("Hello");
PrintTask t2 = new PrintTask("World");
t1.start();   // Creates a new OS thread; that thread calls run()
t2.start();   // Another new thread — both run concurrently
```

> **Calling `run()` Instead of `start()`**
>
> - Calling `t1.run()` directly does **not** create a new thread. 
>   - The code executes on the **current thread** — just like a regular method call. 
> - The `start()` method is what triggers OS-level thread creation.
>   - `start()` can be called **only once** per `Thread` object. Calling it a second time throws `IllegalThreadStateException`.

---

### 2.2 Way 2 — Implement `Runnable` (Returns void and can't throw exception)

- Implementing `Runnable` is preferred over extending `Thread` because Java does not support multiple inheritance. If a class already extends another class, it cannot also extend `Thread`. 
- `Runnable` separates the task logic from the thread mechanism cleanly.

```java
// Option A: Named class
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Task running on: " + Thread.currentThread().getName());
    }
}
new Thread(new MyTask()).start();

// Option B: Lambda expression (Java 8+)
// Runnable is a functional interface, so lambdas work directly
Runnable task = () -> System.out.println("Lambda task running!");
new Thread(task).start();

// Option C: Inline lambda
new Thread(() -> System.out.println("Inline task")).start();
```

---

### 2.3 Way 3 — Implement `Callable` (Returns a Result)

- `Runnable`'s `run()` method returns `void` and cannot throw checked exceptions. 
- `Callable<T>` addresses both limitations — it returns a value of type `T` and can throw a checked exception.

```java
import java.util.concurrent.*;

Callable<Integer> sumTask = () -> {
    int sum = 0;
    for (int i = 1; i <= 100; i++) sum += i;
    return sum;   // Returns 5050
};

// FutureTask wraps a Callable so it can be given to a Thread
FutureTask<Integer> futureTask = new FutureTask<>(sumTask);
new Thread(futureTask).start();

// get() blocks the calling thread until the result is available
Integer result = futureTask.get();
System.out.println("Sum = " + result);  // Sum = 5050
```

---

### 2.4 Way 4 — Use `ExecutorService` (Best Practice for Production)

- In production code, raw `Thread` objects are rarely created directly. 
- `ExecutorService` manages a pool of threads — creating, reusing, and destroying them automatically.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<Integer> future = executor.submit(() -> heavyCalculation());
int result = future.get();   // Blocks until result is ready
executor.shutdown();
```

---

### 2.5 Summary

| Approach             | When to Use                                                                          |
|----------------------|--------------------------------------------------------------------------------------|
| Extend `Thread`      | Simple examples. Limited by single inheritance — not suitable for real applications. |
| Implement `Runnable` | Task with no return value. Preferred for fire-and-forget tasks.                      |
| Implement `Callable` | Task that must return a value or throw a checked exception.                          |
| `ExecutorService`    | **Always use in production.** Handles lifecycle, pooling, and results automatically. |

---

## 3. Thread Lifecycle — All Thread States

Every Java thread goes through a well-defined set of states during its lifetime. The `Thread.State` enum defines these states.

![img.png](img.png)

```
                         ┌──────────────────────────────────────┐
                         │   (notified / timeout / lock free)   │
  NEW ──start()──► RUNNABLE ◄────────────────────────────────────┘
                     │
          ┌──────────┼──────────────────────────┐
          │          │                          │
          ▼          ▼                          ▼
       BLOCKED    WAITING               TIMED_WAITING
   (acquiring    (wait / join /        (sleep(n) / wait(n)
      a lock)     park — no timeout)    / join(n))
          │          │                          │
          └──────────┴──────────────────────────┘
                     │
               run() ends
                     │
                     ▼
               TERMINATED
```

| State             | Description                                                                                                                                                                                                          |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **NEW**           | - Thread object created with `new Thread()`. `start()` not yet called. <br/> - No OS thread exists yet.                                                                                                              |
| **RUNNABLE**      | - `start()` has been called. <br/> - The thread is either actively executing on a CPU core or ready and waiting for the OS scheduler to assign CPU time.<br/> - Both cases appear identical from Java.               |
| **BLOCKED**       | - The thread is trying to enter a `synchronized` block, but the monitor lock is held by another thread.<br/> - The thread waits until the lock is released.                                                          |
| **WAITING**       | - The thread has voluntarily paused indefinitely.<br/> - Caused by `Object.wait()`, `Thread.join()` (no timeout), or `LockSupport.park()`.<br/> - Wakes only when explicitly notified or the joined thread finishes. |
| **TIMED_WAITING** | - Similar to WAITING but with a maximum timeout.<br/> - Caused by `Thread.sleep(n)`, `Object.wait(n)`, `Thread.join(n)`.<br/> - The thread wakes automatically when the timeout expires.                             |
| **TERMINATED**    | - The `run()` method has returned normally or an uncaught exception caused the thread to stop.<br/> - The thread object still exists in memory but can never be restarted.                                           |

> 🎯 **Question: What is the difference between BLOCKED and WAITING?**
>
> **BLOCKED:** The thread is **actively trying to acquire a lock**. It becomes BLOCKED automatically when it reaches a `synchronized` block whose monitor is held by another thread. The thread did not choose to stop — it is being prevented.
>
> **WAITING:** The thread has **voluntarily given up the CPU** by calling `wait()`, `join()`, or `park()`. It will only resume when another thread calls `notify()`, `notifyAll()`, or the joined thread finishes.

---

### 3.1 Daemon Threads

```java
Thread t = new Thread(() -> backgroundWork());
t.setDaemon(true);   // Must be called BEFORE start()
t.start();
```

- **Daemon threads** are background service threads (e.g., the Garbage Collector, JIT compiler). 
- When all non-daemon threads finish, the JVM shuts down — even if daemon threads are still running.
- 
- **Non-daemon threads** (user threads) keep the JVM alive until they complete.
- `setDaemon(true)` must be called **before** `start()`. Calling it after throws `IllegalThreadStateException`.

---

## 4. Thread Safety

### 4.1 What Does Thread Safety Mean?

A class or piece of code is **thread-safe** if it behaves correctly when accessed by multiple threads simultaneously, without requiring additional synchronization from the caller.

Thread safety is violated by three fundamental problems.

---

### 4.2 Problem 1 — Race Condition

- A **race condition** occurs when two or more threads access shared data concurrently, and the final result depends on the order in which the threads happen to be scheduled. 
- The outcome is non-deterministic.

The most common example is `count++`. Although it appears to be a single operation, it compiles into **three separate bytecode instructions**:

```
Step 1: READ   — load the current value of count into a CPU register
Step 2: ADD    — add 1 to the register value
Step 3: WRITE  — store the result back to count in memory
```

If two threads interleave between these steps, the final value is incorrect:

```
Thread A                            Thread B
Step 1: READ count  → 0
Step 2: ADD 1       → 1
                                    Step 1: READ count  → 0   (stale read!)
Step 3: WRITE 1 to count
                                    Step 2: ADD 1       → 1
                                    Step 3: WRITE 1 to count

Final value: count = 1   (expected: 2)
```

```java
// Unsafe — race condition on count
public class UnsafeCounter {
    private int count = 0;

    public void increment() {
        count++;   // NOT atomic — compiles to three separate bytecode instructions
    }
}
// With 1000 threads each calling increment() once,
// the final count may be 850, 920, or any value below 1000.
```

---

### 4.3 Problem 2 — Visibility Problem

- Modern CPUs have multiple levels of cache (L1, L2, L3). 
- When a thread writes a variable, the updated value may remain in the CPU's cache and **not be flushed to main memory immediately**. 
- Another thread running on a different core may read a **stale value** from its own cache.

```java
// Thread 1 sets running = false, but Thread 2 may never observe the change
class Worker {
    private boolean running = true;   // No visibility guarantee

    public void stop() {
        running = false;   // May only be written to Thread 1's CPU cache
    }

    public void run() {
        while (running) {   // Thread 2 reads from its own cache — may loop forever
            doWork();
        }
    }
}

// Fix: use the volatile keyword
private volatile boolean running = true;
// volatile guarantees all reads and writes go through main memory
```

---

### 4.4 Problem 3 — Atomicity Problem

- An **atomic operation** is one that completes entirely as a single, indivisible unit — it cannot be interrupted midway by another thread. 
- The `count++` operation is not atomic. Reading or writing a 64-bit `long` or `double` on a 32-bit JVM is also not atomic — it is split into two 32-bit operations.

---

> 💡 **Summary of the Three Thread Safety Problems**
>
> | Problem            | Root Cause                                              | Fix                                       |
> |--------------------|---------------------------------------------------------|-------------------------------------------|
> | **Race Condition** | Multi-step operations interleave between threads        | `synchronized`, `AtomicInteger`, or locks |
> | **Visibility**     | CPU caching causes threads to see stale values          | `volatile` keyword or `synchronized`      |
> | **Atomicity**      | A multi-step operation is interrupted by another thread | `synchronized` or Atomic classes          |

---

## 5. The `synchronized` Keyword

- `synchronized` is the most straightforward mechanism for achieving thread safety. 
- It uses a **monitor lock** (also called an intrinsic lock). 
- Only one thread may hold a given monitor lock at a time. 
- Any other thread that attempts to enter a `synchronized` block protected by that same lock will **block** until the lock is released.

> **Analogy:** A `synchronized` block is like a single-occupancy room with one key. Only one person can be inside at a time. Others wait outside until the current occupant leaves and the key is available.

---

### 5.1 Synchronized Method

```java
public class SafeCounter {
    private int count = 0;

    // Lock object is 'this' — the SafeCounter instance
    // Only one thread can execute this method on the same instance at a time
    public synchronized void increment() {
        count++;   // Now safe — no other thread can interleave here
    }

    public synchronized int getCount() {
        return count;
    }
}
```

For a `static synchronized` method, the lock is on the `Class` object (e.g., `SafeCounter.class`), not on any instance.

---

### 5.2 Synchronized Block (Preferred — Finer Granularity)

- Locking the entire method is wasteful when only a small part accesses shared state. 
- A synchronized block locks **only the critical section**, allowing the rest of the method to run without contention.

```java
public class BankAccount {
    private double balance;
    private final Object lock = new Object();  // Dedicated lock object

    public void withdraw(double amount) {
        // Non-critical work — no lock needed here
        logWithdrawalAttempt(amount);

        synchronized (lock) {   // Only this section is protected
            if (balance >= amount) {
                balance -= amount;
            } else {
                throw new IllegalStateException("Insufficient funds");
            }
        }

        // Non-critical work again — no lock needed
        sendNotification(amount);
    }
}
```

> 🎯 **Question: What is the lock object in `synchronized`?**
>
> - For a **synchronized instance method**: the lock is `this` (the current object instance).
> - For a **synchronized static method**: the lock is the `Class` object (e.g., `MyClass.class`).
> - For a **synchronized block**: the lock is whatever object is specified inside the parentheses — any non-null object.
>
> **Important:** Two threads synchronizing on **different objects** do not block each other. For synchronization to work correctly, all threads must lock on the **same object**.

---

### 5.3 What `synchronized` Guarantees

`synchronized` provides **two guarantees together**:

1. **Mutual Exclusion (Atomicity):** Only one thread at a time executes the synchronized block.
2. **Visibility:** When a thread exits a synchronized block, all its writes are flushed to main memory. When another thread enters a synchronized block on the same lock, it sees all those flushed writes.

---

## 6. `volatile` and Atomic Classes

### 6.1 The `volatile` Keyword

- The `volatile` keyword instructs the JVM to always **read from and write to main memory** for that variable, bypassing CPU caches entirely. This solves the **visibility problem**.

- However, `volatile` does **not** provide atomicity. It is not a substitute for `synchronized` when compound operations (like `count++`) are involved.

```java
// Correct use of volatile — a simple stop-flag
public class TaskRunner {
    private volatile boolean shouldStop = false;

    public void stop() {
        shouldStop = true;   // Written directly to main memory immediately
    }

    public void run() {
        while (!shouldStop) {   // Always reads the latest value from main memory
            processNextItem();
        }
        System.out.println("Stopped cleanly.");
    }
}

// Incorrect use — volatile does NOT make compound operations atomic
volatile int count = 0;
count++;   // Still three operations (read, add, write) — still a race condition
           // Use AtomicInteger instead
```

| Feature          | `synchronized`                      | `volatile`                                      |
|------------------|-------------------------------------|-------------------------------------------------|
| Mutual exclusion | YES — only one thread at a time     | NO                                              |
| Visibility       | YES — on lock acquire and release   | YES — on every individual read and write        |
| Atomicity        | YES — entire block is atomic        | NO — only individual reads/writes are atomic    |
| Performance      | Slower — lock acquisition overhead  | Faster — no lock, just a memory barrier         |
| Typical use      | Compound operations on shared state | Simple flags and single-variable status signals |

---

### 6.2 Atomic Classes — Lock-Free Thread Safety

- The `java.util.concurrent.atomic` package provides classes such as `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, and `AtomicReference<T>`. 
- These classes are thread-safe and typically **faster than `synchronized`** because they use hardware-level **CAS (Compare-And-Swap)** instructions — no OS-level locking is required.

**How CAS works:** 
- The CPU atomically checks whether the current memory value matches an expected value. 
- If it matches, the new value is written in. 
- If not (another thread changed it first), the operation retries automatically. 
- This is called **optimistic locking** — it assumes contention is rare.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class SafeCounter {
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();   // Atomic — uses hardware CAS; no OS lock required
    }

    public int getCount() {
        return count.get();
    }
}

// Common AtomicInteger operations
AtomicInteger ai = new AtomicInteger(10);

ai.get();                     // Read the current value
ai.set(20);                   // Write a new value
ai.incrementAndGet();         // Atomic ++i — returns new value
ai.getAndIncrement();         // Atomic i++ — returns old value
ai.addAndGet(5);              // Atomic += 5 — returns new value
ai.compareAndSet(20, 99);     // If current == 20, set to 99. Returns true/false.
ai.getAndUpdate(x -> x * 2); // Atomically apply a function; return old value

// LongAdder — better than AtomicLong for high-contention counters
// Maintains separate counter cells per CPU to minimise contention
LongAdder adder = new LongAdder();
adder.increment();
long total = adder.sum();   // Aggregates all internal cells
```

---

## 7. Advanced Locking — `java.util.concurrent.locks`

### 7.1 Limitations of `synchronized`

The `synchronized` keyword addresses many use cases, but has several constraints:

- A thread cannot **attempt** to acquire a lock — it either succeeds or blocks indefinitely.
- A blocked thread **cannot be interrupted**.
- There is no way to set a **timeout** on lock acquisition.
- There is no built-in support for **fairness** (FIFO ordering of waiting threads).
- Only **one condition** (wait/notify) is available per lock object.

`ReentrantLock` addresses all of these.

---

### 7.2 `ReentrantLock`

- The name **Reentrant** means the same thread can acquire the lock multiple times without deadlocking itself — the same behavior as `synchronized`. 
- The key requirement is that `unlock()` must be called the same number of times as `lock()`.

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class SafeTransfer {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    // Basic usage — ALWAYS release the lock in a finally block
    public void transfer(double amount) {
        lock.lock();
        try {
            balance -= amount;
        } finally {
            lock.unlock();   // Guaranteed to execute even if an exception is thrown
        }
    }

    // tryLock() — attempt to acquire the lock without blocking indefinitely
    public boolean tryTransfer(double amount) {
        if (lock.tryLock()) {   // Returns true if acquired, false if not available
            try {
                balance -= amount;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;   // Lock was busy; the attempt is abandoned
    }

    // tryLock with timeout — wait at most 1 second, then give up
    public boolean transferWithTimeout(double amount) throws InterruptedException {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                balance -= amount;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }

    // lockInterruptibly() — can be cancelled if thread is interrupted while waiting
    public void interruptibleTransfer(double amount) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            balance -= amount;
        } finally {
            lock.unlock();
        }
    }
}

// Fair lock — threads acquire the lock in the order they requested it (FIFO)
// Prevents starvation but reduces overall throughput
ReentrantLock fairLock = new ReentrantLock(true);
```

> **Critical Rule — Always Release in `finally`**
>
> If `lock.lock()` is called and an exception occurs before `lock.unlock()`, the lock is **never released**. 
> 
> All threads waiting for that lock are blocked indefinitely — a deadlock. The `finally` block guarantees the lock is released regardless of exceptions.


---

### 7.3 `ReadWriteLock` — Optimised for Read-Heavy Scenarios

- In many systems, shared data is read far more often than it is written — a configuration cache, for example, may be read thousands of times between updates. 
- A regular lock makes all reads mutually exclusive, even though concurrent reads are perfectly safe.

`ReadWriteLock` provides two separate locks:
- **Read lock:** Multiple threads may hold it simultaneously, as long as no thread holds the write lock.
- **Write lock:** Exclusive — only one thread may hold it, and only when no read lock is held.

```java
import java.util.concurrent.locks.*;

public class ThreadSafeCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple reader threads can execute this simultaneously
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // Only one writer at a time; all readers are blocked while writing
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

---


### 7.5 Lock Types — Comparison

| Lock Type               | Best For                                           | Key Characteristic                         |
|-------------------------|----------------------------------------------------|--------------------------------------------|
| `synchronized`          | Simple critical sections                           | Auto-released; minimal boilerplate         |
| `ReentrantLock`         | Needs tryLock, timeout, interruptible, or fairness | Must manually call `unlock()` in `finally` |
| `ReadWriteLock`         | Read-heavy, write-rare shared data                 | Concurrent reads + exclusive writes        |
| `StampedLock` (Java 8+) | Optimistic reads for maximum performance           | Most flexible; most complex                |

---

## 8. ExecutorService & Thread Pools

### 8.1 The Problem with Creating Raw Threads

- Creating an OS thread is an expensive operation — it involves memory allocation for the thread stack (typically 512 KB–1 MB), OS kernel calls, and scheduling setup (approximately 0.5–1 ms per thread).

- If an application creates a new thread for every incoming request and receives 10,000 requests per second, the JVM would attempt to maintain 10,000 active threads — almost certainly leading to `OutOfMemoryError`.

**Thread pools** address this by maintaining a fixed set of reusable threads. When a task arrives, it is assigned to an idle thread. When the task completes, the thread returns to the pool and awaits the next task.

---

### 8.2 How `ExecutorService` Works Internally

```
Task submitted               Task Queue               Thread Pool              Result
executor.submit(task) ──► [task1][task2][task3] ──►  [Thread-1] ──► runs ──► Future<T>
                          (BlockingQueue)             [Thread-2]
                                                      [Thread-3]
                                                      [Thread-4]

When queue is full AND all threads are busy → Rejection Policy is applied
```

---

### 8.3 Executor Factory Methods

```java
// Fixed thread pool — exactly N threads always alive
// New tasks queue when all threads are busy
ExecutorService fixed = Executors.newFixedThreadPool(4);

// Cached thread pool — creates threads on demand; reuses idle threads (60s keepalive)
// Warning: can create an unbounded number of threads under heavy sustained load
ExecutorService cached = Executors.newCachedThreadPool();

// Single-thread executor — exactly one thread; tasks execute sequentially in submission order
// If the thread dies due to an exception, a replacement is created automatically
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled thread pool — for delayed and periodic task execution
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Run once after a 5-second delay
scheduler.schedule(() -> sendReminder(), 5, TimeUnit.SECONDS);

// Run every 10 seconds — interval measured from the START of each execution
scheduler.scheduleAtFixedRate(heartbeat, 0, 10, TimeUnit.SECONDS);

// Run 10 seconds AFTER the previous execution ENDS
scheduler.scheduleWithFixedDelay(cleanup, 0, 10, TimeUnit.SECONDS);
```

> 🎯 **Question: Difference between `scheduleAtFixedRate` and `scheduleWithFixedDelay`?**
>
> - **`scheduleAtFixedRate`:** The next execution is scheduled at a fixed interval **from the start of the previous execution**. If the task runs longer than the interval, the next run begins immediately after the previous ends — no overlap, but also no gap.
> - **`scheduleWithFixedDelay`:** The next execution begins after a fixed delay **from the end of the previous execution**. This guarantees a minimum rest period between consecutive runs regardless of task duration.

---

### 8.4 `ThreadPoolExecutor` — Full Control (Production Standard)

- The factory methods are convenient but carry a hidden risk: `newFixedThreadPool` and `newSingleThreadExecutor` use an **unbounded `LinkedBlockingQueue`** (capacity `Integer.MAX_VALUE`). 
- Under heavy load, millions of tasks can accumulate in the queue, causing `OutOfMemoryError`.

- In production, a `ThreadPoolExecutor` should always be configured explicitly with a **bounded queue**.

```java
import java.util.concurrent.*;

ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                                    // corePoolSize    — always-alive threads
    8,                                    // maximumPoolSize — maximum threads under load
    60L, TimeUnit.SECONDS,               // keepAliveTime   — idle non-core threads exit after this
    new LinkedBlockingQueue<>(500),      // workQueue       — bounded; prevents OOM
    new ThreadFactory() {                 // threadFactory   — custom names aid debugging
        private int count = 0;
        public Thread newThread(Runnable r) {
            return new Thread(r, "worker-" + (++count));
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()   // rejectedExecutionHandler
);
```

**Task submission flow — step by step:**

```
New task submitted
       │
       ▼
 Active threads < corePoolSize?
       │ YES → Create new core thread → Execute task
       │ NO
       ▼
 Queue has space?
       │ YES → Add to queue → Wait for idle thread
       │ NO
       ▼
 Active threads < maximumPoolSize?
       │ YES → Create new non-core thread → Execute task
       │ NO
       ▼
 Apply RejectedExecutionHandler
```

| Rejection Policy          | Behaviour When Pool and Queue Are Both Full                                                                    |
|---------------------------|----------------------------------------------------------------------------------------------------------------|
| `AbortPolicy` *(default)* | Throws `RejectedExecutionException`. Caller must handle it.                                                    |
| `CallerRunsPolicy`        | The calling thread runs the task itself. Naturally slows the producer — creates backpressure. **Recommended.** |
| `DiscardPolicy`           | Silently discards the task. Acceptable only when task loss is tolerable.                                       |
| `DiscardOldestPolicy`     | Discards the oldest queued task and retries the new submission.                                                |

> 💡 **Thread Pool Sizing Guidelines**
>
> - **CPU-bound tasks:** `corePoolSize = Runtime.getRuntime().availableProcessors()`
> - **I/O-bound tasks:** `corePoolSize = CPU cores × (1 + average_wait_time / average_compute_time)`
> - Always use a **bounded queue** in production to prevent `OutOfMemoryError` under sustained load.

---

### 8.5 Submitting Tasks and Retrieving Results

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// execute() — fire-and-forget; no return value; uncaught exceptions are silently lost
executor.execute(() -> logMessage("event"));

// submit(Runnable) — returns Future<?>; get() returns null on completion
Future<?> f1 = executor.submit(() -> sendEmail());
f1.get();   // Blocks until the task completes

// submit(Callable<T>) — returns Future<T> with a real result
Future<String> f2 = executor.submit(() -> fetchFromDatabase(id));
String data  = f2.get();                           // Blocks until done
String data2 = f2.get(3, TimeUnit.SECONDS);        // Blocks at most 3 s; throws TimeoutException
// If the task threw an exception, f2.get() throws ExecutionException wrapping the original

// invokeAll() — submit a collection; blocks until ALL tasks complete
List<Callable<Integer>> tasks = List.of(() -> process("A"), () -> process("B"));
List<Future<Integer>> results = executor.invokeAll(tasks);

// invokeAny() — submits a collection; returns result of FIRST completed task; cancels rest
Integer fastest = executor.invokeAny(tasks);
```

---

### 8.6 Graceful Shutdown

```java
// shutdown() — stops accepting new tasks; submitted tasks complete normally
executor.shutdown();

// Best practice — graceful shutdown with timeout
executor.shutdown();
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        // Tasks did not finish within the timeout — force termination
        List<Runnable> notStarted = executor.shutdownNow();
        System.out.println(notStarted.size() + " tasks were abandoned");
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();   // Restore the interrupt status flag
}
```

> 🎯 **Question: Difference between `shutdown()` and `shutdownNow()`?**
>
> - **`shutdown()`:** The executor stops accepting new tasks. Tasks already in the queue and tasks currently running complete normally.
> - **`shutdownNow()`:** Attempts to stop all executing tasks by calling `Thread.interrupt()` on worker threads. Returns a list of queued tasks that were never started. Does not guarantee that running tasks will stop — it depends on whether the task checks the interrupt flag.

---

## 9. CompletableFuture — Async Programming

### 9.1 Problem with the Old `Future` Interface

- The `Future` interface (Java 5) has a fundamental limitation: `future.get()` **always blocks** the calling thread until the result is ready. 
  - There is no mechanism to say "when this result is ready, automatically perform the next step" without blocking.

- `CompletableFuture` (Java 8) solves this by supporting **non-blocking, chainable, callback-driven pipelines**. 
  - Instead of blocking and waiting, stages are declared upfront, and each stage executes automatically when the previous one completes.
---

### 9.2 Pipeline Overview

```
supplyAsync(() -> fetchData())          Stage 1: Starts asynchronously — returns CF<Data>
    .thenApply(data -> process(data))   Stage 2: Transforms the result — returns CF<Result>
    .thenAccept(r -> save(r))           Stage 3: Consumes the result — returns CF<Void>

Error handling  : .exceptionally(ex -> fallback)
Always-run      : .handle((result, ex) -> ...)   |   .whenComplete((result, ex) -> ...)
Combining       : .thenCombine(cf2, (a, b) -> merge)
Wait for all    : CompletableFuture.allOf(cf1, cf2, cf3)
Wait for first  : CompletableFuture.anyOf(cf1, cf2, cf3)
```

---

### 9.3 Creating a `CompletableFuture`

```java
import java.util.concurrent.CompletableFuture;

// runAsync — runs a Runnable asynchronously; returns CF<Void>
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() ->
    System.out.println("Running on: " + Thread.currentThread().getName())
);

// supplyAsync — runs a Supplier asynchronously; returns CF<T>
// By default uses ForkJoinPool.commonPool() — a shared pool across the JVM
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> fetchUser(id));

// Recommended in production: supply a dedicated executor
// The common pool is shared; a slow task can starve other unrelated pipelines
ExecutorService pool = Executors.newFixedThreadPool(10);
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(
    () -> fetchUser(id), pool
);

// Immediately completed — useful in tests or as precomputed fallback values
CompletableFuture<String> done = CompletableFuture.completedFuture("cached-result");
```

---

### 9.4 Chaining Stages

```java
CompletableFuture<Void> pipeline = CompletableFuture

    // Stage 1: fetch userId asynchronously
    .supplyAsync(() -> getUserId())                    // Returns CF<String>

    // thenApply — synchronous transform: T → U; runs on the completing thread
    .thenApply(userId -> fetchUser(userId))            // Returns CF<User>

    // thenApplyAsync — asynchronous transform: T → U; runs on a new thread from pool
    .thenApplyAsync(user -> enrichUser(user), pool)    // Returns CF<EnrichedUser>

    // thenCompose — flatMap: T → CF<U>
    // Used when the transformation itself is an async operation returning a CF
    .thenCompose(user -> loadPermissionsAsync(user.getId()))  // Returns CF<UserWithPerms>

    // thenAccept — consumes the result; no return value
    .thenAccept(user -> saveToCache(user))             // Returns CF<Void>

    // thenRun — no input, no output; side-effect after the previous stage completes
    .thenRun(() -> System.out.println("Pipeline complete."));
```
---

### 9.5 Combining Multiple Futures

```java
// thenCombine — combines two independent futures when BOTH complete
CompletableFuture<String> userFuture  = CompletableFuture.supplyAsync(() -> getUser());
CompletableFuture<String> orderFuture = CompletableFuture.supplyAsync(() -> getOrder());

CompletableFuture<String> combined = userFuture.thenCombine(
    orderFuture,
    (user, order) -> "User: " + user + ", Order: " + order
);
// Both run in parallel; combined executes when BOTH complete

// allOf — waits for ALL futures to complete; returns CF<Void>
List<CompletableFuture<String>> futures = List.of(
    CompletableFuture.supplyAsync(() -> callServiceA()),
    CompletableFuture.supplyAsync(() -> callServiceB()),
    CompletableFuture.supplyAsync(() -> callServiceC())
);
CompletableFuture<List<String>> allResults =
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList()));

// anyOf — completes as soon as the FIRST future finishes
CompletableFuture<Object> fastest = CompletableFuture.anyOf(
    callReplica1Async(), callReplica2Async(), callReplica3Async()
);
String result = (String) fastest.join();  // Returns the first available result
```

---

### 9.6 Error Handling

```java
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> {
        if (dbDown) throw new RuntimeException("Database unavailable");
        return fetchData();
    })

    // exceptionally — runs ONLY on exception; provides a fallback value
    .exceptionally(ex -> {
        logger.error("Caught: " + ex.getMessage());
        return "fallback-value";  // Pipeline continues with this value
    })

    // handle — runs ALWAYS (success or failure); can inspect both result and exception
    .handle((result, ex) -> {
        if (ex != null) return "Error: " + ex.getMessage();
        return result.toUpperCase();
    })

    // whenComplete — runs ALWAYS; for side-effects only; does NOT change the result
    .whenComplete((result, ex) -> {
        if (ex != null) logger.error("Failed", ex);
        else logger.info("Succeeded: " + result);
    });
```
---

### 9.7 `CompletableFuture` Method Reference

| Method                 | What It Does                                                 |
|------------------------|--------------------------------------------------------------|
| `supplyAsync(fn)`      | Starts an async task that returns a value                    |
| `runAsync(fn)`         | Starts an async task with no return value                    |
| `thenApply(fn)`        | Transforms the result synchronously (like `map`)             |
| `thenCompose(fn)`      | Chains another async operation (like `flatMap`)              |
| `thenAccept(fn)`       | Consumes the result; returns `CF<Void>`                      |
| `thenRun(fn)`          | Runs a side-effect after completion; no input or output      |
| `thenCombine(cf2, fn)` | Waits for this and another CF; combines both results         |
| `allOf(cfs...)`        | Waits for ALL futures to complete                            |
| `anyOf(cfs...)`        | Completes when the FIRST future completes                    |
| `exceptionally(fn)`    | Handles exception; returns a fallback value                  |
| `handle(fn)`           | Handles success or failure; can change the result            |
| `whenComplete(fn)`     | Side-effects on completion; cannot change the result         |
| `join()`               | Blocks until complete; throws unchecked exception on failure |
| `complete(val)`        | Manually completes the CF with a given value                 |

---

## 10. Synchronization Utilities

### 10.1 `CountDownLatch` — Wait for N Tasks to Complete

`CountDownLatch` is a one-time gate. It is initialised with a count. Each call to `countDown()` decrements the count by one. Any thread calling `await()` is blocked until the count reaches zero. Once at zero, the latch **cannot be reset** — it is single-use.

```java
import java.util.concurrent.CountDownLatch;

// Example: wait for 3 services to start before accepting traffic
CountDownLatch latch = new CountDownLatch(3);

executor.submit(() -> { startServiceA(); latch.countDown(); });   // count: 3 → 2
executor.submit(() -> { startServiceB(); latch.countDown(); });   // count: 2 → 1
executor.submit(() -> { startServiceC(); latch.countDown(); });   // count: 1 → 0

latch.await();   // Blocks until count reaches zero
System.out.println("All services ready. Accepting requests.");

// With timeout — do not wait more than 30 seconds
boolean ready = latch.await(30, TimeUnit.SECONDS);
if (!ready) System.out.println("Timeout — not all services started.");
```

---

### 10.2 `CyclicBarrier` — Synchronize N Threads at a Checkpoint

`CyclicBarrier` is a reusable barrier. A fixed number of threads (called **parties**) must all call `await()` before any of them can proceed. 
Once all parties arrive, an optional barrier action runs, and then all threads are released simultaneously. The barrier resets automatically for the next cycle.

```java
import java.util.concurrent.CyclicBarrier;

CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    // Runs once all 3 threads have arrived; executes before any thread proceeds
    System.out.println("All threads completed this phase. Advancing to next.");
});

Runnable worker = () -> {
    try {
        loadData();
        barrier.await();    // Wait for all 3 threads to finish loading

        processData();
        barrier.await();    // Barrier resets automatically — wait again

        saveResults();
    } catch (Exception e) {
        Thread.currentThread().interrupt();
    }
};

for (int i = 0; i < 3; i++) executor.submit(worker);
```

| Feature | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Reusable | No — single use | Yes — resets after each cycle |
| Who counts down | Any thread can call `countDown()` | Only the participating threads call `await()` |
| Who waits | Observer threads wait; workers count down | All N threads wait for each other |
| Barrier action | Not supported | Optional — runs when all N parties arrive |
| Typical use | Wait for N tasks to finish | Synchronise N threads at multiple phase boundaries |

---

### 10.3 `Semaphore` — Limit Concurrent Access to a Resource

A `Semaphore` manages a fixed set of **permits**. A thread must acquire a permit before accessing the resource. When finished, it releases the permit so another waiting thread may acquire it. If no permits are available, the requesting thread blocks.

```java
import java.util.concurrent.Semaphore;

// Limit to 5 concurrent database connections
Semaphore dbPool = new Semaphore(5);

public String queryDatabase(String sql) throws InterruptedException {
    dbPool.acquire();   // Blocks if all 5 permits are in use
    try {
        return runQuery(sql);   // At most 5 threads execute here simultaneously
    } finally {
        dbPool.release();   // Always release in finally — even on exception
    }
}

// Non-blocking attempt
if (dbPool.tryAcquire()) {
    try { return runQuery(sql); } finally { dbPool.release(); }
} else {
    return "Service at capacity. Please retry.";
}

// With timeout
if (dbPool.tryAcquire(2, TimeUnit.SECONDS)) { ... }

// Monitoring
dbPool.availablePermits();   // Permits currently free
dbPool.getQueueLength();     // Threads currently waiting for a permit
```

**Common use cases:** Rate limiting API calls, connection pool management, throttling access to an external service, binary semaphore (1 permit) acting as a mutex that can be released by a different thread than the one that acquired it.

---

## 11. Concurrent Collections

### 11.1 Why Not Use `synchronized` Collections?

`Collections.synchronizedMap(new HashMap<>())` wraps a `HashMap` with a single mutex. Every read and every write locks the **entire map**. Even multiple threads performing read-only operations must take turns — despite the fact that concurrent reads are completely safe. This is a significant performance bottleneck under high concurrency.

Additionally, even with `synchronizedMap`, iteration over the map is **not thread-safe** — the calling code must synchronise externally around the iteration loop.

The `java.util.concurrent` collections are purpose-built for concurrent access. They use fine-grained locking or lock-free algorithms to allow much higher throughput.

---

### 11.2 `ConcurrentHashMap`

The standard thread-safe map for concurrent applications. In Java 8+, read operations (`get`) are completely **lock-free**. Write operations use CAS and bucket-level locking — only the affected segment is locked, not the entire map.

```java
ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// Basic thread-safe operations
wordCount.put("hello", 1);
Integer count = wordCount.get("hello");

// putIfAbsent — atomic; only inserts if the key does not already exist
wordCount.putIfAbsent("world", 0);

// compute — atomically performs a read-modify-write on a key
wordCount.compute("hello", (key, old) -> old == null ? 1 : old + 1);

// computeIfAbsent — inserts a value only if the key is absent; ideal for lazy-loading
wordCount.computeIfAbsent("newWord", key -> expensiveLoad(key));

// merge — combines delta with existing value, or inserts delta if key is absent
wordCount.merge("hello", 1, Integer::sum);
```

> 🎯 **Question: `ConcurrentHashMap` vs `Hashtable` vs `synchronizedMap`?**
>
> |                  | `Hashtable`                   | `synchronizedMap`                      | `ConcurrentHashMap`                                      |
> |------------------|-------------------------------|----------------------------------------|----------------------------------------------------------|
> | Locking strategy | Full-map lock on every method | Full-map lock on every operation       | Lock-free reads; bucket-level writes                     |
> | Concurrent reads | No — serialised               | No — serialised                        | Yes — fully concurrent                                   |
> | Iteration        | Not thread-safe               | Not thread-safe (must sync externally) | Weakly consistent — no `ConcurrentModificationException` |
> | Null keys/values | Not allowed                   | Allowed                                | Not allowed                                              |
> | Recommendation   | Legacy — avoid                | Adequate for low concurrency           | **Preferred in all production code**                     |

---

### 11.3 `CopyOnWriteArrayList`

- Every modification to a `CopyOnWriteArrayList` creates a **complete copy** of the underlying array with the change applied, then atomically replaces the reference. 
- Read operations always see a consistent snapshot and require no locking whatsoever.

```java
CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

listeners.add("listenerA");      // Creates a new copy of the array; swaps reference
listeners.remove("listenerA");   // Same — fresh copy is created

// Iteration is always safe — iterates over an immutable snapshot
// No ConcurrentModificationException, even if another thread modifies the list
for (String listener : listeners) {
    notify(listener);
}
```

- **Best for:** Lists that are iterated frequently but modified rarely (e.g., event listeners, observer patterns).
- **Not suitable for:** Large lists with frequent writes — each write copies the entire array, making modifications O(n).

---

### 11.4 `BlockingQueue` — Producer-Consumer Pattern

- `BlockingQueue` provides thread-safe, blocking enqueue and dequeue operations. 
  - `put()` blocks if the queue is full; 
  - `take()` blocks if the queue is empty. 
  - No manual `wait()`/`notify()` is required — the blocking behavior is built in.

```
Producer                     BlockingQueue                 Consumer
generateTask() ─► put()  ►  [t1][t2][t3][t4]  ◄─ take() ◄─ processTask()
                  BLOCKS                          BLOCKS
                  if full                         if empty
               (backpressure)                  (waits for work)
```


| Implementation                  | Characteristics                                                                     |
|---------------------------------|-------------------------------------------------------------------------------------|
| `LinkedBlockingQueue(capacity)` | Linked nodes; separate head/tail locks — best general-purpose choice                |
| `ArrayBlockingQueue(capacity)`  | Array-backed; single lock; optional FIFO fairness                                   |
| `PriorityBlockingQueue`         | Unbounded; elements dequeued by natural priority; `put()` never blocks              |
| `SynchronousQueue`              | Zero capacity; each `put()` blocks until a corresponding `take()` is called         |
| `DelayQueue`                    | Elements become available only after a specified delay — suited for scheduled tasks |

---

## 12. Classic Concurrency Problems

### 12.1 Deadlock — Circular Lock Dependency

- **Deadlock** occurs when two or more threads are permanently blocked, each waiting for a lock held by another thread in the cycle. 
- No thread can proceed, and the application appears to freeze silently.

**Deadlock requires all four conditions simultaneously (breaking any one prevents it):**

| Condition        | Description                                                 |
|------------------|-------------------------------------------------------------|
| Mutual exclusion | Only one thread can hold a lock at a time                   |
| Hold and wait    | A thread holds one lock while waiting to acquire another    |
| No preemption    | Locks cannot be forcibly taken from a thread                |
| Circular wait    | Thread A waits for B, B waits for A — a circular dependency |

**Prevention strategies:**
- Strategy 1: Lock ordering — always acquire locks in the same global order
- Strategy 2: tryLock with timeout

---

### 12.2 Livelock — Active but Making No Progress --> both threads keep yielding in response to each other

- A **livelock** occurs when threads are not blocked but are continuously changing state in response to each other, without making any actual progress. 
- Unlike deadlock, the threads are active — but their activity accomplishes nothing.

---

### 12.3 Starvation — A Thread Never Gets CPU Time

- **Starvation** occurs when a thread is perpetually denied access to a shared resource because higher-priority threads or unfair scheduling consistently win. 
- The thread is alive and eligible to run but never actually gets scheduled.

---

## 13. `wait()`, `notify()`, `notifyAll()`

### 13.1 Overview

- Every Java object has a built-in **monitor**. 
- The methods `wait()`, `notify()`, and `notifyAll()` operate on this monitor and enable threads to pause and resume in coordination.

These methods **must be called inside a `synchronized` block** on the same object. Calling them outside `synchronized` throws `IllegalMonitorStateException`.

**Behaviour summary:**

- **`wait()`** 
  - Atomically releases the monitor lock and suspends the calling thread. 
  - The thread moves to WAITING (or TIMED_WAITING if a timeout is given). 
  - The lock is released atomically, so no other thread can interleave between the condition check and the wait.
- **`notify()`** 
  - Wakes up **one** arbitrarily chosen thread waiting on this monitor. 
  - That thread then competes to reacquire the lock.
- **`notifyAll()`** 
  - Wakes up **all** threads waiting on this monitor. 
  - All threads compete for the lock; only one succeeds. 
  - The rest return to WAITING.

---

### 13.2 Producer-Consumer with `wait()`/`notifyAll()`

```java
public class BoundedBuffer {
    private final Queue<Integer> buffer = new LinkedList<>();
    private final int MAX = 5;

    // Producer
    public synchronized void produce(int item) throws InterruptedException {
        // ALWAYS use while, not if — spurious wakeups can occur
        while (buffer.size() == MAX) {
            wait();   // Releases the lock; suspends until notified
                      // On waking, re-acquires lock and re-checks the condition
        }
        buffer.add(item);
        System.out.println("Produced: " + item);
        notifyAll();   // Wakes all waiting threads (producers and consumers)
    }

    // Consumer
    public synchronized int consume() throws InterruptedException {
        while (buffer.isEmpty()) {
            wait();   // Wait until a producer adds an item
        }
        int item = buffer.poll();
        System.out.println("Consumed: " + item);
        notifyAll();   // Wake waiting producers
        return item;
    }
}
```

> ⚠️ **Always Use `while`, Never `if`, Around `wait()`**
>
> **Spurious wakeups** are permitted by the Java specification — a thread can wake from `wait()` without any thread calling `notify()`. This is uncommon but real.
>
> Using `if (condition) wait()` means the thread resumes without re-checking the condition, potentially proceeding with stale or invalid state.
>
> Using `while (condition) { wait(); }` guarantees the condition is re-evaluated after every wakeup — whether genuine or spurious.

> 🎯 **Question: `notify()` vs `notifyAll()`?**
>
> - **`notify()`** wakes one arbitrarily chosen waiting thread. If the wrong type wakes (e.g., a producer wakes when the buffer is still full), it re-checks its `while` condition, finds it still blocked, and goes back to `wait()` — but the correct thread (a consumer) was never notified. This is the **missed signal** problem.
> - **`notifyAll()`** wakes all waiting threads. Each independently re-checks its own `while` condition. The thread whose condition is satisfied proceeds; all others return to `wait()`. This is safer and generally preferred.

---

## 14. ThreadLocal

### 14.1 What is `ThreadLocal`?

- `ThreadLocal<T>` provides each thread with its own independent copy of a variable. 
- Thread A's value is completely isolated from Thread B's value. 
- No synchronization is needed because there is no sharing between threads.

Each value is stored inside the **thread itself** (in a `ThreadLocalMap` field on the `Thread` object), not in the `ThreadLocal` instance.

---

### 14.2 Usage Examples

```java
// SimpleDateFormat is not thread-safe.
// Creating one instance per call is expensive.
// ThreadLocal gives each thread its own instance — created lazily on first access.
public class DateFormatter {
    private static final ThreadLocal<SimpleDateFormat> formatter =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public static String format(Date date) {
        return formatter.get().format(date);  // Each thread uses its own instance
    }
}

// Per-request context in a web application
public class RequestContext {
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

    // Called at the start of every request (e.g., in a servlet filter)
    public static void setUser(User user) { currentUser.set(user); }

    // Called from any application layer without passing the user explicitly
    public static User getUser() { return currentUser.get(); }

    // Called at the end of every request — CRITICAL
    public static void clear() { currentUser.remove(); }
}
```

---

### 14.3 Memory Leak Risk in Thread Pools

> **Memory Leak — `remove()` Must Always Be Called**
>
> Thread pool threads are **reused** across requests. If a `ThreadLocal` value is set during Request A and `remove()` is never called, the next request processed by that same thread will inherit Request A's value.
>
> Two consequences:
> 1. **Data leakage:** User A's session data may be visible during User B's request — a security vulnerability.
> 2. **Memory leak:** The old object is never garbage collected because the thread holds a strong reference through its `ThreadLocalMap`.
>
> The fix: always call `threadLocal.remove()` in a `finally` block at the end of every request or task.

```java
// Correct usage pattern in a thread pool environment
try {
    RequestContext.setUser(resolveUser(request));
    handleRequest(request);
} finally {
    RequestContext.clear();   // Executes unconditionally — removes the ThreadLocal value
}
```

---

### Q1: Explain the Java Memory Model (JMM). Why is it important?

**Answer:**

- The Java Memory Model defines the rules governing how threads interact through shared memory. 
- Without these rules, a write performed by Thread A may not be observable by Thread B due to CPU-level caching and instruction reordering by both the CPU and the JVM compiler.

The JMM defines **happens-before relationships** — a formal guarantee that if operation A happens-before operation B, then all memory writes made by A are visible to B before B reads them.

**Key happens-before rules:**

| Relationship                | Guarantee                                                               |
|-----------------------------|-------------------------------------------------------------------------|
| `synchronized` exit → enter | All writes before unlock are visible after the same lock is acquired    |
| `volatile` write → read     | A volatile write is visible to any subsequent read of the same variable |
| `Thread.start()`            | All code before `start()` is visible to the newly started thread        |
| `Thread.join()`             | All code in thread T is visible after `t.join()` returns                |
| Static initialiser          | Fully completes before any thread accesses the class                    |

Without proper synchronization, the JMM permits threads to see inconsistent or stale data — leading to bugs that are extremely difficult to reproduce because they depend on CPU timing and JVM-level optimisations.

---

### Q2: What is the difference between `Callable` and `Runnable`?

**Answer:**

| Feature            | `Runnable`                            | `Callable<T>`                            |
|--------------------|---------------------------------------|------------------------------------------|
| Method signature   | `void run()`                          | `T call() throws Exception`              |
| Return value       | None                                  | Returns a value of type `T`              |
| Checked exceptions | Cannot be thrown                      | Can be thrown                            |
| Usage              | `Thread`, `ExecutorService.execute()` | `ExecutorService.submit()` → `Future<T>` |

- `Runnable` is appropriate for fire-and-forget tasks that produce no result. 
- `Callable` is used when the task must return a computed value or may throw a checked exception. 
  - `ExecutorService.submit(Callable)` wraps the result in a `Future<T>` for later retrieval.

---

### Q3: What is a deadlock? How is it detected and prevented?

**Answer:**

Deadlock is a situation where two or more threads are permanently blocked, each waiting for a resource held by another thread in the same cycle. No thread can proceed.

**Four necessary conditions (all must hold simultaneously):**
1. Mutual exclusion — resource can be held by only one thread at a time
2. Hold and wait — a thread holds a resource while waiting for another
3. No preemption — resources cannot be forcibly taken from a thread
4. Circular wait — a circular chain of threads, each waiting for the next

**Detection:** 
- `jstack <PID>` prints a thread dump that explicitly reports deadlocks. 
- `ThreadMXBean.findDeadlockedThreads()` enables programmatic detection.

**Prevention strategies:**
- **Lock ordering:** Establish a global order for acquiring locks and always follow it across all threads.
- **`tryLock` with timeout:** Use `ReentrantLock.tryLock(timeout)` — if the lock is not available within the timeout, release held locks and retry.
- **Minimise lock scope:** Hold locks for the shortest possible duration; avoid acquiring multiple locks simultaneously.

---

### Q4: What is the difference between `Future` and `CompletableFuture`?

**Answer:**

| Feature           | `Future` (Java 5)                            | `CompletableFuture` (Java 8)                   |
|-------------------|----------------------------------------------|------------------------------------------------|
| Blocking          | `get()` always blocks                        | Non-blocking — supports callbacks and chaining |
| Chaining          | Not supported                                | `thenApply`, `thenCompose`, etc.               |
| Error handling    | Only through `ExecutionException` on `get()` | `exceptionally`, `handle` inline in pipeline   |
| Combining futures | Not supported                                | `allOf`, `anyOf`, `thenCombine`                |
| Manual completion | Not possible                                 | `complete(value)`                              |

`CompletableFuture` enables building non-blocking asynchronous pipelines where each stage executes automatically when the previous one completes — without any thread blocking to wait.

---

### Q5: How would a thread-safe counter for 1000 concurrent threads be implemented?

**Answer:**

```java
// 1. synchronized method — correct, but creates a single-lock bottleneck
public synchronized void increment() { count++; }

// 2. synchronized block — same bottleneck at a finer scope
synchronized (this) { count++; }

// 3. AtomicInteger — lock-free CAS; significantly faster than synchronized
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

- `AtomicInteger` is appropriate when both increment and read operations are needed frequently. 

---

### Q6: What are `notify()` and `notifyAll()`, and when should each be used?

**Answer:**

Both wake threads that are blocked in `wait()` on an object's monitor. Both must be called from within a `synchronized` block on that same object.

- **`notify()`** wakes one arbitrarily chosen waiting thread. Efficient when all waiting threads are interchangeable — any one of them can safely proceed. If threads serve different roles (e.g., producers and consumers sharing the same lock), `notify()` may wake the wrong type, causing a missed signal.

- **`notifyAll()`** wakes all waiting threads. Each reacquires the lock in turn and re-checks its `while` condition. Only threads whose condition is satisfied proceed; the rest return to `wait()`. Safer and generally preferred.

`notify()` should only be used when it is certain that all waiting threads are identical in purpose and any one can handle being woken.

---

### Q7: What is `ThreadLocal`, and what risks come with it?

**Answer:**

`ThreadLocal<T>` provides thread-isolated storage. Each thread gets its own independent copy of the stored variable. No synchronisation is needed because threads do not share the value.

**Common use cases:**
- Per-request user or transaction context in web applications
- Thread-safe reuse of non-thread-safe objects (e.g., `SimpleDateFormat`)
- Per-thread database connection or session

**Risks:**

1. **Memory leak in thread pools:** Thread pool threads are reused. If `remove()` is not called after each request, the previous value remains on the thread indefinitely and is never garbage collected, causing the heap to grow over time.

2. **Data leakage between requests:** The next request on a reused thread may inherit the previous request's `ThreadLocal` value — a potential security issue.

**Mitigation:** Always call `threadLocal.remove()` in a `finally` block at the end of every request or task lifecycle.

---

### Q8: What happens when `ThreadPoolExecutor`'s queue is full and all threads are busy?

**Answer:**

When the queue is full and the active thread count equals `maximumPoolSize`, the configured `RejectedExecutionHandler` is invoked:

- **`AbortPolicy` (default):** Throws `RejectedExecutionException`. The caller must catch and handle it.
- **`CallerRunsPolicy`:** The submitting thread executes the task itself. This naturally slows the producer — it acts as backpressure. Generally recommended.
- **`DiscardPolicy`:** The task is silently dropped. No exception. Acceptable only when task loss is tolerable.
- **`DiscardOldestPolicy`:** The oldest task in the queue is discarded, and the new task is retried.

A custom handler can be implemented by implementing the `RejectedExecutionHandler` interface — for example, to log the rejection and route the task to a secondary overflow queue.

---


| Tool                           | Use When                                                         |
|--------------------------------|------------------------------------------------------------------|
| `synchronized`                 | Simple critical section with no special lock requirements        |
| `volatile`                     | Single boolean flag or status variable read by multiple threads  |
| `AtomicInteger` / `AtomicLong` | Atomic operations on a single numeric variable                   |
| `LongAdder`                    | Very high-contention increment-only counter                      |
| `ReentrantLock`                | Need `tryLock`, timeout, interruptible acquire, or fairness      |
| `ReadWriteLock`                | Shared data that is read frequently but written rarely           |
| `ConcurrentHashMap`            | Thread-safe map with concurrent reads and writes                 |
| `CopyOnWriteArrayList`         | List iterated frequently but modified rarely                     |
| `BlockingQueue`                | Producer-Consumer pattern with automatic blocking on full/empty  |
| `CountDownLatch`               | Wait for N tasks to complete before proceeding (one-time gate)   |
| `CyclicBarrier`                | N threads must meet at a checkpoint before continuing (reusable) |
| `Semaphore`                    | Limit concurrent access to a resource to at most N threads       |
| `ThreadLocal`                  | Per-thread isolated variable (e.g., per-request context)         |
| `CompletableFuture`            | Non-blocking async pipeline with callbacks and stage composition |
| Immutable objects              | Shared state that never changes — no synchronisation required    |

---

### `ThreadPoolExecutor` — Task Flow

```
New task submitted
       │
       ▼
 Active threads < corePoolSize?
   YES → Create new core thread → Execute task
   NO  ↓
       ▼
 Queue has space?
   YES → Add task to queue → Wait for an idle thread
   NO  ↓
       ▼
 Active threads < maximumPoolSize?
   YES → Create new non-core thread → Execute task
   NO  ↓
       ▼
 Apply RejectedExecutionHandler
 (Abort / CallerRuns / Discard / DiscardOldest / Custom)
```

---

### `CompletableFuture`

| Method                 | Category    | Notes                                           |
|------------------------|-------------|-------------------------------------------------|
| `supplyAsync(fn)`      | Create      | Async task with a return value                  |
| `runAsync(fn)`         | Create      | Async task with no return value                 |
| `thenApply(fn)`        | Transform   | Synchronous T → U                               |
| `thenApplyAsync(fn)`   | Transform   | Asynchronous T → U on a new thread              |
| `thenCompose(fn)`      | Transform   | T → CF\<U\> — flatMap equivalent                |
| `thenAccept(fn)`       | Consume     | No return value; returns `CF<Void>`             |
| `thenRun(fn)`          | Side-effect | No input, no output                             |
| `thenCombine(cf2, fn)` | Combine     | Wait for both; combine results                  |
| `allOf(cfs...)`        | Combine     | Wait for all to complete                        |
| `anyOf(cfs...)`        | Combine     | Complete on first completion                    |
| `exceptionally(fn)`    | Error       | Handle exception; return fallback value         |
| `handle(fn)`           | Error       | Handle success or failure; can transform result |
| `whenComplete(fn)`     | Error       | Side-effect only; cannot change result          |
| `complete(val)`        | Manual      | Force completion with a specified value         |
| `join()`               | Get result  | Blocks; throws unchecked exception on failure   |

---

### Golden Rules of Concurrency

> 1. **Identify shared mutable state.** Immutable objects and thread-local state require no synchronisation.
> 2. **Synchronise all accesses to shared mutable state consistently.** If one thread writes, all reads and writes must be synchronised on the same lock.
> 3. **Keep synchronised regions small.** Lock only the critical section, not the entire method.
> 4. **Use bounded queues in production thread pools.** Unbounded queues lead to `OutOfMemoryError` under sustained load.
> 5. **Always call `ThreadLocal.remove()` in thread pools.** Failing to do so causes data leakage between request lifecycles and long-term memory leaks.

---