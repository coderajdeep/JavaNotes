[ChatGPT](https://chatgpt.com/share/6aae4180-efb4-83e8-a096-de9021d95999)

# Java Thread Methods — Compact Note

## 1. `start()` vs `run()`

```java
Thread t = new Thread(() -> {
    // task
});

t.start();
```

- `start()` creates/schedules a **new thread**.
- JVM eventually invokes that thread's `run()`.
- Calling `t.run()` directly is just a normal method call and **does not create a new thread**.

```text
t.start()
   ↓
NEW → RUNNABLE → execution → TERMINATED
```

---

## 2. `sleep()`

```java
Thread.sleep(1000);
```

- Pauses the **currently executing thread**.
- `sleep()` is a static method of `Thread`.
- `1000` = milliseconds.
- State becomes `TIMED_WAITING`.
- Does **not release a monitor/lock**.

```text
RUNNABLE
   ↓ sleep()
TIMED_WAITING
   ↓ timeout
RUNNABLE
```

Important:

```java
Thread.sleep(1000);
```

means **current thread sleeps**, not an arbitrary `Thread` object.

---

## 3. `join()`

```java
worker.join();
```

Means:

> "Wait until `worker` finishes."

Example:

```java
worker.start();
worker.join();

System.out.println("Worker finished");
```

Main waits while worker runs.

```text
Worker: RUNNABLE ─────→ TERMINATED
                           ↑
                           │
Main:    WAITING ──────────┘
                           ↓
                        continues
```

Timed version:

```java
worker.join(2000);
```

Waits at most 2 seconds.

---

## 4. `yield()`

```java
Thread.yield();
```

Means:

> "I'm willing to give other runnable threads a chance."

It is only a **scheduler hint**.

- No guarantee another thread will run.
- Should not be used for synchronization or correctness.

---

# 5. `interrupt()`

```java
worker.interrupt();
```

Means:

> **Request interruption of `worker`.**

It does **not forcibly kill the thread**.

```text
Main
 |
 | worker.interrupt()
 ↓
Worker
 |
 | responds to interruption
```

If the worker is sleeping:

```java
try {
    Thread.sleep(10000);
} catch (InterruptedException e) {
    // handle interruption
}
```

`interrupt()` causes `sleep()` to throw `InterruptedException`.

```text
TIMED_WAITING
      ↓
 interrupt()
      ↓
InterruptedException
      ↓
catch block
```

If the thread is simply doing CPU work, `interrupt()` does not automatically stop it. The thread must cooperate.

---

# 6. Interrupt Flag

A thread has an **interrupt status flag**.

### `isInterrupted()`

```java
thread.isInterrupted();
```

- Checks the interrupt flag.
- **Does not clear it.**

### `Thread.interrupted()`

```java
Thread.interrupted();
```

- Checks the **current thread's** interrupt flag.
- **Clears the flag**.

### Difference

```text
isInterrupted()
    → check only

interrupted()
    → check + clear
```

---

# 7. Why Restore Interrupt Status?

Consider:

```java
try {
    Thread.sleep(10000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

When `sleep()` detects interruption:

```text
interrupt()
    ↓
InterruptedException
    ↓
interrupt status is cleared
```

Therefore, inside the catch block:

```java
Thread.currentThread().interrupt();
```

restores the interrupt flag.

```text
Before catch:
interrupt flag = true

InterruptedException:
interrupt flag = false

currentThread().interrupt():
interrupt flag = true
```

Why?

> To preserve the interruption signal so higher-level code can detect it.

---

# 8. `isAlive()`

```java
thread.isAlive();
```

Returns whether the thread has started and has not yet terminated.

```text
NEW          → false
RUNNABLE     → true
BLOCKED      → true
WAITING      → true
TIMED_WAITING→ true
TERMINATED   → false
```

---

# 9. `getState()`

```java
thread.getState();
```

Possible states:

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

Important:

`RUNNABLE` can mean either:

- currently executing, or
- ready/eligible to execute.

Java does not have a separate `RUNNING` state.

---

# 10. Thread Scheduling

After:

```java
worker.start();
```

both threads may be runnable:

```text
Main   → RUNNABLE
Worker → RUNNABLE
```

The scheduler decides which runs.

### Main can sleep before Worker runs

Possible:

```text
worker.start()

Main   → RUNNABLE
Worker → RUNNABLE

Main gets CPU
   ↓
Main → TIMED_WAITING

Worker gets CPU later
```

There is **no guaranteed execution order** after `start()`.

Also, on a multi-core CPU, both threads can execute simultaneously.

---

# 11. Main Thread Can Terminate Before Worker

Yes.

```text
Main   → TERMINATED
Worker → RUNNABLE
```

is completely possible.

Important distinction:

> **Main thread terminating does NOT necessarily mean JVM terminates.**

The JVM can continue while non-daemon worker threads are alive.

```text
JVM
├── Main   → TERMINATED
└── Worker → RUNNABLE
```

The JVM normally exits when there are no remaining **non-daemon threads**.

If you need Main to wait for Worker:

```java
worker.start();
worker.join();
```

---

# 12. `currentThread()`

```java
Thread.currentThread();
```

Returns the thread currently executing the code.

Example:

```java
System.out.println(Thread.currentThread().getName());
```

---

# 13. Thread Priority

```java
thread.setPriority(Thread.MAX_PRIORITY);
```

Priorities:

```text
MIN_PRIORITY  = 1
NORM_PRIORITY = 5
MAX_PRIORITY  = 10
```

Priority is a **scheduling hint**, not a guarantee.

Don't assume:

```text
Priority 10 → always executes before Priority 5
```

---

# 14. Thread Name

```java
thread.setName("Worker-1");
thread.getName();
```

Useful for debugging and logs.

```java
Thread.currentThread().getName();
```

---

# 15. Daemon Thread

```java
thread.setDaemon(true);
```

A daemon thread is a background thread that **does not keep the JVM alive by itself**.

Must be called before `start()`:

```java
thread.setDaemon(true);
thread.start();
```

---

# 16. `wait()` vs `sleep()` vs `join()`

| | `sleep()` | `wait()` | `join()` |
|---|---|---|---|
| Belongs to | `Thread` | `Object` | `Thread` |
| Purpose | Pause current thread | Wait for notification/condition | Wait for another thread |
| Releases monitor? | ❌ No | ✅ Yes | Not a monitor-release primitive |
| Requires synchronized? | ❌ | ✅ | ❌ |
| Can timeout? | ✅ | ✅ | ✅ |
| Can throw `InterruptedException`? | ✅ | ✅ | ✅ |

---

# 17. Most Important Mental Model

Remember these five:

```text
sleep()
→ "Pause me for some time."

join()
→ "Wait until that thread finishes."

wait()
→ "Wait for a condition/notification."

yield()
→ "I can give another runnable thread a chance."

interrupt()
→ "Please respond to this interruption request."
```

And:

```text
start()
→ Start a new thread.

isAlive()
→ Is this thread still alive?

getState()
→ What state is this thread in?

currentThread()
→ Which thread is executing right now?

isInterrupted()
→ Check interrupt flag without clearing it.

interrupted()
→ Check + clear current thread's interrupt flag.
```

## Interview Takeaways

1. `start()` creates/schedules a new thread; `run()` does not.
2. `sleep()` pauses the current thread and **doesn't release locks**.
3. `join()` makes one thread wait for another to finish.
4. `interrupt()` is a **cooperative interruption request**, not a forceful kill.
5. `InterruptedException` clears the interrupt status; restore it when appropriate with:
   ```java
   Thread.currentThread().interrupt();
   ```
6. `yield()` is only a scheduler hint.
7. Main can terminate before worker; JVM may still remain alive because of non-daemon threads.
8. After `start()`, don't assume which thread runs first.
9. `RUNNABLE` means eligible to run or actually running; Java has no separate `RUNNING` state.
10. `wait()` releases the monitor; `sleep()` does not.
