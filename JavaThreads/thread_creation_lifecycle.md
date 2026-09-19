[ChatGPT](https://chatgpt.com/share/6aae2441-d244-83ee-a4d7-1851f4665696)

# Java Threads — Creation, Lifecycle & Synchronization

## 1. Thread

A **thread** is an independent path of execution inside a process.

A Java application normally starts with a JVM process containing multiple JVM/runtime threads. The JVM creates the initial **main thread**, which executes:

```java
public static void main(String[] args)
```

Additional application threads can be created by the application.

---

## 2. Thread Creation

### Extending `Thread`

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running");
    }
}

MyThread t = new MyThread();
t.start();
```

`start()` creates/starts a new thread and eventually executes `run()`.

```java
t.run();    // normal method call; NO new thread
t.start();  // starts a new thread
```

### Implementing `Runnable` — commonly preferred

```java
Runnable task = () -> {
    System.out.println("Running");
};

Thread t = new Thread(task);
t.start();
```

Why prefer `Runnable`?

- Separates **task** from **execution mechanism**.
- Java supports single class inheritance; extending `Thread` consumes the only `extends` slot.
- The same task can be given to different threads.
- Works naturally with `ExecutorService` and thread pools.

Mental model:

```text
Runnable → WHAT work should be done?
Thread   → WHERE/HOW is the work executed?
```

### `Callable`

`Callable` is similar to `Runnable` but can:

- Return a value.
- Throw checked exceptions.

```java
Callable<Integer> task = () -> 10 + 20;
```

Usually submitted through `ExecutorService`.

---

# 3. Thread Lifecycle / States

Java's `Thread.State` has **6 states**:

```text
NEW
 │
 │ start()
 ▼
RUNNABLE
 │
 ├── lock unavailable ──► BLOCKED
 │                          │
 │                          └──► RUNNABLE
 │
 ├── wait/join ──────────► WAITING
 │                          │
 │                          └──► RUNNABLE
 │
 ├── sleep/timed wait ──► TIMED_WAITING
 │                          │
 │                          └──► RUNNABLE
 │
 └── run() completes ───► TERMINATED
```

### NEW

Thread object exists, but `start()` hasn't been called.

```java
Thread t = new Thread(task);
```

State: `NEW`

---

### RUNNABLE

After:

```java
t.start();
```

the thread enters `RUNNABLE`.

Java's `RUNNABLE` includes both:

- Ready to run.
- Actually running on the CPU.

Java does **not** have a separate `RUNNING` state.

---

### BLOCKED

Thread is waiting to acquire a **monitor lock**.

Example:

```java
synchronized (obj) {
    // critical section
}
```

If Thread A owns `obj`'s monitor and Thread B tries to enter another synchronized section using the same monitor:

```text
Thread A → owns monitor
Thread B → wants monitor → BLOCKED
```

---

### WAITING

Thread waits **indefinitely** for another thread/event.

Common examples:

```java
thread.join();
object.wait();
LockSupport.park();
```

Mental model:

> **"Wait until something happens."**

Example:

```java
worker.join();
```

means:

> Wait until `worker` finishes.

---

### TIMED_WAITING

Thread waits for a **limited amount of time**.

Examples:

```java
Thread.sleep(1000);
thread.join(1000);
object.wait(1000);
```

Mental model:

> **"Wait until something happens OR the timeout expires."**

Example:

```text
TIMED_WAITING
      │
      ├── event occurs ──► RUNNABLE
      │
      └── timeout ───────► RUNNABLE
```

---

### TERMINATED

The thread's `run()` method has finished.

```text
RUNNABLE → TERMINATED
```

A terminated thread cannot be started again.

```java
t.start();
t.start();  // IllegalThreadStateException
```

---

# 4. Monitor Lock

A **monitor** is Java's built-in synchronization mechanism associated with an object.

A `synchronized` block acquires the monitor of the specified object:

```java
synchronized (obj) {
    // protected code
}
```

Only one thread at a time can own that monitor.

If the monitor is already owned:

```text
Thread A
   ↓
owns obj's monitor

Thread B
   ↓
tries to acquire obj's monitor
   ↓
BLOCKED
```

### Synchronized instance method

```java
synchronized void increment() {
    count++;
}
```

is conceptually equivalent to:

```java
void increment() {
    synchronized (this) {
        count++;
    }
}
```

So a synchronized instance method automatically uses **`this` object's monitor**.

You don't pass `this` explicitly.

For:

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

each object has its own monitor:

```text
c1 → Monitor 1
c2 → Monitor 2
```

A thread holding `c1`'s monitor doesn't block another thread trying to acquire `c2`'s monitor.

---

# 5. WAITING vs TIMED_WAITING

The easiest distinction:

```text
WAITING
"I'll wait indefinitely until something happens."

TIMED_WAITING
"I'll wait, but only for a limited amount of time."
```

| Operation | State |
|---|---|
| `thread.join()` | WAITING |
| `thread.join(1000)` | TIMED_WAITING |
| `object.wait()` | WAITING |
| `object.wait(1000)` | TIMED_WAITING |
| `Thread.sleep(1000)` | TIMED_WAITING |
| `LockSupport.park()` | WAITING |
| `LockSupport.parkNanos(...)` | TIMED_WAITING |

### Key distinction

**WAITING** → no timeout.

**TIMED_WAITING** → timeout exists.

**BLOCKED** → waiting specifically to acquire a monitor lock.

---

## 6. Important `start()` vs `run()` Rule

Always remember:

```java
t.start();
```

→ starts a **new thread**.

```java
t.run();
```

→ simply calls `run()` like an ordinary method; **no new thread is created**.

---

## 7. Overall Mental Model

```text
                 Java Process
                      │
          ┌───────────┴───────────┐
          │                       │
      Main Thread            Other Threads
     (JVM-created)           (application)
          │
          ├── creates → Worker Thread
          └── creates → Worker Thread


Thread lifecycle:

NEW
 │
 │ start()
 ▼
RUNNABLE
 │
 ├── monitor unavailable → BLOCKED
 │
 ├── wait/join            → WAITING
 │
 ├── sleep/timed wait     → TIMED_WAITING
 │
 └── run() finishes       → TERMINATED
```
