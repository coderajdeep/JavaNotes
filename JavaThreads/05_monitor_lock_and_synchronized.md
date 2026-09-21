[ChatGPT](https://chatgpt.com/share/6aaf76a4-2640-83e9-9a2c-b0027e0cfa2b)

# Java Monitor Locks & `synchronized`

## 1. Why Synchronization?

When multiple threads access **shared mutable data**, operations can interleave and cause:

- **Race conditions**
- **Data inconsistency**
- **Visibility problems**

Example:

```java
count++;
```

is conceptually:

```text
read → add 1 → write
```

Two threads can read the same value and overwrite each other's updates.

**Synchronization provides:**
1. **Mutual exclusion** — one thread at a time.
2. **Memory visibility** — changes made before releasing a monitor become visible to a thread that subsequently acquires the same monitor.

---

## 2. Monitor Lock

Every Java object can be associated with a **monitor**.

Think of it as:

```text
Object
  ↓
Monitor / Lock
```

Only **one thread can own a particular monitor at a time**.

```java
synchronized (obj) {
    // critical section
}
```

Here, `obj` is the lock object.

### Key rule

> **Same lock → mutual exclusion.**  
> **Different locks → no automatic mutual exclusion.**

---

## 3. How `synchronized` Works

Conceptually:

```text
Thread
  ↓
Try to acquire monitor
  ↓
 ┌───────────────┐
 │ Lock free?    │
 └───────────────┘
    ↓         ↓
   Yes        No
    ↓          ↓
Acquire      BLOCKED
    ↓
Execute critical section
    ↓
Release monitor
```

If another thread tries to acquire the same monitor while it is owned, that thread becomes **`BLOCKED`**.

---

## 4. Synchronized Instance Method

```java
class Counter {

    synchronized void increment() {
        count++;
    }
}
```

A synchronized **instance method locks `this`**.

Conceptually equivalent to:

```java
void increment() {
    synchronized (this) {
        count++;
    }
}
```

Therefore:

```text
obj.methodA()
obj.methodB()
```

cannot execute their synchronized portions simultaneously if both use the same object's monitor.

---

## 5. Synchronized Block

```java
void increment() {

    // non-critical work

    synchronized (this) {
        count++;
    }

    // more work
}
```

Only the block is protected.

### Method vs Block

| | Synchronized Method | Synchronized Block |
|---|---|---|
| Scope | Entire method | Selected section |
| Instance lock | `this` | Whatever object is specified |
| Control | Less precise | More precise |
| Typical benefit | Simple synchronization | Smaller critical section |

---

## 6. Object-Level Lock

Instance synchronization uses an **object's monitor**.

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

There are two different locks:

```text
c1 → Monitor A
c2 → Monitor B
```

Therefore:

```text
Thread A → c1.increment()
Thread B → c2.increment()
```

can execute concurrently.

**Important:** `synchronized` does NOT mean only one thread in the entire JVM. It means only one thread can own a **particular monitor** at a time.

---

## 7. Static Synchronized Method

```java
class Counter {

    static synchronized void increment() {
        count++;
    }
}
```

A static synchronized method locks the **class object**:

```java
Counter.class
```

Conceptually:

```java
static void increment() {
    synchronized (Counter.class) {
        count++;
    }
}
```

### Remember

```text
synchronized instance method
        ↓
      this

static synchronized method
        ↓
   ClassName.class
```

---

## 8. Instance vs Static Lock

```java
synchronized void instanceMethod() { }

static synchronized void staticMethod() { }
```

Locks:

```text
instanceMethod() → this
staticMethod()   → ClassName.class
```

These are **different monitors**.

Therefore, an instance synchronized method and a static synchronized method can execute concurrently.

---

## 9. Custom Lock

Instead of locking `this`, use a dedicated object:

```java
class Counter {

    private final Object lock = new Object();

    void increment() {
        synchronized (lock) {
            count++;
        }
    }
}
```

Here:

```text
lock object
    ↓
its monitor
    ↓
critical section
```

### Why custom locks?

- Keeps the synchronization mechanism private.
- Avoids exposing `this` as the lock.
- Allows different resources to use different locks.

Example:

```java
private final Object accountLock = new Object();
private final Object transactionLock = new Object();
```

Different locks allow greater concurrency.

---

## 10. Critical Section

A **critical section** is code that accesses shared mutable state and must be protected from concurrent modification.

Example:

```java
synchronized (lock) {

    if (balance >= amount) {
        balance -= amount;
    }
}
```

Protect the **whole logical operation**, not just one statement.

For example:

```text
check balance
     +
modify balance
```

should be performed atomically with respect to competing threads.

---

## 11. What Happens When Another Thread Enters?

Suppose Thread A owns:

```java
synchronized (lock) {
    // critical section
}
```

Thread B tries the same:

```text
Thread A
   ↓
acquires lock
   ↓
critical section

Thread B
   ↓
tries same lock
   ↓
BLOCKED
   ↓
waits for lock

Thread A
   ↓
releases lock
   ↓
Thread B can acquire it
```

---

## 12. `BLOCKED` vs `WAITING`

### `BLOCKED`

Trying to acquire a monitor currently owned by another thread:

```java
synchronized (lock) {
}
```

### `WAITING`

Explicitly waiting, for example:

```java
lock.wait();
```

So:

```text
Trying to acquire monitor → BLOCKED
Calling wait()            → WAITING
```

---

## 13. Race Condition Prevention

Without synchronization:

```text
count = 0

Thread A → read 0
Thread B → read 0
Thread A → write 1
Thread B → write 1

Final = 1 ❌
```

With synchronization:

```text
Thread A → acquire lock
         → read
         → increment
         → write
         → release

Thread B → acquire lock
         → read updated value
         → increment
         → write
         → release

Final = 2 ✅
```

Synchronization makes the critical operation mutually exclusive.

---

# Most Important Rules

### Rule 1

```java
synchronized void method()
```

→ locks `this`.

### Rule 2

```java
static synchronized void method()
```

→ locks `ClassName.class`.

### Rule 3

```java
synchronized (obj)
```

→ locks `obj`.

### Rule 4

> Same monitor → threads block each other.

### Rule 5

> Different monitors → they can execute concurrently.

### Rule 6

> `synchronized` protects a **critical section**, not merely a variable.

### Rule 7

> Always ask: **"What object is being locked?"**

---

## One-Line Mental Model

```text
synchronized
     ↓
Acquire object's monitor
     ↓
Only one owner at a time
     ↓
Others trying same monitor → BLOCKED
     ↓
Critical section executes safely
     ↓
Monitor released
```

### Interview Shortcut

Whenever you see:

```java
synchronized (X) {
    ...
}
```

immediately identify **X**.

**X = the lock.**

Then ask:

> "Which other code synchronizes on this same object?"

Those critical sections are mutually exclusive.
