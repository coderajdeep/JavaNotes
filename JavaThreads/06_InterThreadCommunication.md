[ChatGPT](https://chatgpt.com/share/6ab16325-1084-83ee-94fa-59b9276ecaf3)

# Java Inter-Thread Communication — Compact Notes

## 1. What is Inter-Thread Communication?

**Inter-Thread Communication (ITC)** allows threads to coordinate while working with shared resources.

Classic example:

```text
Producer ──► Shared Buffer ◄── Consumer
```

If the buffer is empty, the consumer should **wait** rather than continuously checking (busy waiting).

Java's traditional monitor-based ITC uses:

```java
wait()
notify()
notifyAll()
```

These methods belong to **`Object`**, because every Java object can act as a monitor.

---

# 2. Java Monitor — Mental Model

Conceptually, an object's monitor can be viewed as:

```text
             Object's Monitor
                   │
        ┌──────────┼──────────┐
        │          │          │
      Owner     Wait Set    Entry Set
        │          │          │
        B        A C D       E F
```

### Owner
Thread currently holding the monitor.

### Wait Set
Threads that called:

```java
lock.wait();
```

They are typically in:

```text
WAITING
```

### Entry Set / Contenders
Threads trying to acquire a monitor currently owned by another thread.

They are typically:

```text
BLOCKED
```

> "Entry Set" is useful conceptual terminology; the Java specification explicitly defines the object's **wait set**.

---

# 3. `wait()`

Basic usage:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
}
```

When a thread calls:

```java
lock.wait();
```

the conceptual sequence is:

```text
Thread owns lock
      ↓
wait()
      ↓
releases monitor
      ↓
enters Wait Set
      ↓
WAITING
```

### Important

`wait()`:

- Releases the **associated monitor**
- Puts the thread into `WAITING`
- Waits for notification, interruption, or a spurious wakeup
- Must be called while owning that object's monitor

Otherwise:

```text
IllegalMonitorStateException
```

---

# 4. `notify()`

```java
synchronized (lock) {
    lock.notify();
}
```

`notify()` selects **one** thread from the object's Wait Set.

Example:

```text
Before:

Owner: B

Wait Set:
    A
    C
    D
```

After `notify()` selects A:

```text
Owner: B

Wait Set:
    C
    D

A → trying to reacquire lock
```

A does **not** immediately execute.

B still owns the monitor.

Therefore:

```text
A → BLOCKED
B → still owns lock
C → WAITING
D → WAITING
```

When B releases the lock, A competes for it.

### Important

`notify()` **does not transfer the lock** to A.

It only makes A eligible to reacquire the monitor.

---

# 5. `notifyAll()`

```java
synchronized (lock) {
    lock.notifyAll();
}
```

Moves all waiting threads out of the Wait Set into competition for the monitor.

```text
Before:

Owner: B

Wait Set:
    A C D
```

After:

```text
Owner: B

Wait Set:
    empty

A C D → compete to reacquire lock
```

Only **one thread can own the monitor at a time**.

---

# 6. `notify()` vs `notifyAll()`

| `notify()` | `notifyAll()` |
|---|---|
| Wakes one waiting thread | Wakes all waiting threads |
| Other waiting threads remain waiting | All become eligible |
| Selected thread isn't guaranteed to get lock immediately | All compete for lock |
| Can be more efficient | Can cause more contention |
| Requires careful synchronization design | Often safer with multiple conditions |

---

# 7. Very Important: `notify()` Doesn't Release Lock

Consider:

```java
synchronized (lock) {
    lock.notify();

    // B still owns lock here
    System.out.println("Still holding lock");
}
```

The sequence is:

```text
B owns lock
   ↓
notify()
   ↓
A becomes eligible
   ↓
B STILL owns lock
   ↓
B continues
   ↓
B exits synchronized block
   ↓
B releases lock
   ↓
A/E/other contenders compete
```

---

# 8. What If Another Thread Is Already BLOCKED?

Suppose:

```text
Owner:
    B

Wait Set:
    A C D

BLOCKED:
    E
```

B calls:

```java
notify();   // selects A
```

Now:

```text
Owner:
    B

Wait Set:
    C D

Trying to acquire:
    A E
```

When B releases the lock:

```text
A ──┐
    ├── compete for monitor
E ──┘
```

**There is no guarantee that A wins.**

E, which was already `BLOCKED`, may acquire the lock before A.

Therefore:

> `notify()` does not give the notified thread priority over already-blocked threads.

---

# 9. Why `while`, Not `if`?

Always use:

```java
while (!condition) {
    wait();
}
```

rather than:

```java
if (!condition) {
    wait();
}
```

Reasons:

### 1. Spurious wakeups

A thread can return from `wait()` without the expected notification.

### 2. Another thread may consume/change the resource first

Example:

```text
Producer → notifyAll()

Consumer A ──┐
              ├── compete
Consumer B ──┘
```

A consumes the item first.

When B gets the lock:

```text
wait() returns
      ↓
B checks condition again
      ↓
condition is false
      ↓
B waits again
```

The `while` loop handles this correctly.

---

# 10. `sleep()` vs `wait()`

This is one of the most important distinctions.

### `sleep()`

```java
Thread.sleep(1000);
```

```text
RUNNABLE
   ↓
sleep()
   ↓
TIMED_WAITING
   ↓
lock is STILL held
   ↓
timeout
   ↓
RUNNABLE
```

`sleep()` **does not release any monitor locks**.

Example:

```java
synchronized (lock) {
    Thread.sleep(1000);
}
```

The thread sleeps while **still holding `lock`**.

Other threads trying to acquire `lock` may become `BLOCKED`.

---

### `wait()`

```java
synchronized (lock) {
    lock.wait();
}
```

```text
RUNNABLE
   ↓
wait()
   ↓
WAITING
   ↓
releases associated monitor
   ↓
notification/interruption
   ↓
tries to reacquire monitor
   ↓
BLOCKED if unavailable
   ↓
acquires lock
   ↓
wait() returns
```

---

# 11. `wait()` vs `sleep()` vs `join()`

| | `wait()` | `sleep()` | `join()` |
|---|---|---|---|
| Defined in | `Object` | `Thread` | `Thread` |
| Releases monitor? | **Yes** | **No** | No |
| Requires synchronized monitor? | **Yes** | No | No |
| Typical state | `WAITING` | `TIMED_WAITING` | `WAITING` |
| Purpose | Inter-thread coordination | Pause execution | Wait for another thread to finish |
| Woken by `notify()`? | Yes | No | No |

---

# 12. `wait(timeout)`

```java
lock.wait(5000);
```

The thread can resume because of:

```text
1. notify()
2. notifyAll()
3. interrupt()
4. timeout
```

Even after being notified or timing out, it must **reacquire the monitor** before `wait()` returns.

---

# 13. Interrupting `wait()`

If a thread is waiting:

```java
lock.wait();
```

and another thread calls:

```java
waitingThread.interrupt();
```

then:

```text
WAITING
   ↓
Interrupted
   ↓
InterruptedException
```

Typical handling:

```java
try {
    synchronized (lock) {
        lock.wait();
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

# 14. Object Used for Communication Must Be the Same

This works:

```java
synchronized (lock) {
    lock.wait();
}
```

and:

```java
synchronized (lock) {
    lock.notify();
}
```

But this does **not** communicate:

```java
Object lock1 = new Object();
Object lock2 = new Object();
```

Thread A:

```java
synchronized (lock1) {
    lock1.wait();
}
```

Thread B:

```java
synchronized (lock2) {
    lock2.notify();
}
```

Because:

```text
A waits on lock1
B notifies lock2

Different monitors → no communication
```

---

# 15. Synchronized Methods and `wait/notify`

Instance synchronized method:

```java
public synchronized void method() {
    wait();
}
```

is associated with:

```java
synchronized (this) {
    this.wait();
}
```

Therefore:

```text
Instance synchronized
        ↓
Monitor = this
```

Static synchronized method:

```java
public static synchronized void method() {
    // ...
}
```

uses:

```text
Monitor = ClassName.class
```

So:

```java
static synchronized void method() {
    wait();
}
```

is conceptually associated with:

```java
synchronized (MyClass.class) {
    MyClass.class.wait();
}
```

---

# 16. Producer-Consumer Pattern

Canonical pattern:

### Consumer

```java
synchronized (lock) {
    while (!available) {
        lock.wait();
    }

    // consume resource
}
```

### Producer

```java
synchronized (lock) {
    // produce resource

    available = true;

    lock.notifyAll();
}
```

Mental model:

```text
Consumer
   │
condition false
   ↓
wait()
   ↓
releases lock
   ↓
WAITING
   │
   │
Producer
   │
changes shared state
   ↓
notifyAll()
   ↓
Consumer becomes eligible
   ↓
reacquires lock
   ↓
checks condition again
   ↓
consumes
```

---

# 17. Most Important Thread-State Transition

For a thread calling `wait()`:

```text
        owns monitor
             │
             ▼
         RUNNABLE
             │
           wait()
             │
             ▼
          WAITING
             │
     notify / notifyAll
             │
             ▼
    tries to reacquire monitor
             │
       ┌─────┴─────┐
       │           │
    available    unavailable
       │           │
       ▼           ▼
    RUNNABLE     BLOCKED
                     │
               acquires monitor
                     │
                     ▼
                  RUNNABLE
                     │
                     ▼
               wait() returns
```

---

# 18. Final Mental Model

Remember these four statements:

> **`synchronized` → acquire/release the monitor.**

> **`wait()` → release the monitor and wait.**

> **`notify()` → wake one waiting thread, but don't release the monitor.**

> **`notifyAll()` → wake all waiting threads, but don't release the monitor.**

And the most important rule:

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // use shared resource
}
```

paired with:

```java
synchronized (lock) {
    // change shared state
    condition = true;

    lock.notifyAll();
}
```

### One-line interview answer

**Inter-thread communication in Java allows threads to coordinate through a shared object's monitor using `wait()`, `notify()`, and `notifyAll()`. `wait()` releases the monitor and enters the wait set, while `notify/notifyAll()` make waiting threads eligible to reacquire the monitor; the notified thread does not automatically get the lock.**
