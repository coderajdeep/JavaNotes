[ChatGPT](https://chatgpt.com/share/6aaf5d33-d3f0-83ee-adb9-0e8ef6065b2c)

# Java Multithreading — 1-Page Interview Revision Sheet

## 1. Race Condition

**Definition:**  
A race condition occurs when the final result depends on the order/timing in which multiple threads execute.

### Simple example

```java
class Counter {
    int count = 0;

    void increment() {
        count++;
    }
}
```

If T1 and T2 both execute `increment()`:

```text
count = 0

T1 → reads 0
T2 → reads 0
T1 → writes 1
T2 → writes 1

Final = 1
Expected = 2
```

The update from one thread is lost.

**Remember:**  
`count++` looks like one statement but conceptually involves:

```text
READ → INCREMENT → WRITE
```

---

## 2. Critical Section

**Definition:**  
The part of code that accesses/modifies a shared resource and needs protection.

```java
void increment() {
    count++;       // Critical Section
}
```

Think:

```text
Shared Resource
      ↓
Critical Section
      ↓
Needs synchronization/protection
```

---

## 3. Shared Resource

A resource accessed by multiple threads.

Examples:

```java
int count;
List<Integer> list;
BankAccount account;
```

Example:

```text
T1 ──┐
     ├──→ count
T2 ──┘
```

Here `count` is the shared resource.

---

# 4. Atomicity

**Definition:**  
An atomic operation happens as one indivisible operation. It cannot be partially completed/interleaved in the middle.

### Atomic

```java
int x = 10;
```

Conceptually, it happens as one operation.

### Non-atomic

```java
count++;
```

Conceptually:

```text
1. Read count
2. Increment
3. Write count
```

Another thread can interfere between these steps.

### Interview point

```text
Atomicity problem → compound operation can be interrupted/interleaved
```

---

# 5. Check-Then-Act Problem

Pattern:

```text
CHECK → ACTION
```

Example:

```java
if (balance >= 500) {
    balance -= 500;
}
```

Two threads can both check the old balance before either updates it.

```text
Initial balance = ₹1000

T1: balance >= 500 → YES
T2: balance >= 600 → YES

T2: withdraw ₹600 → balance = ₹400
T1: withdraw ₹500 → balance = -₹100
```

The **check and action need to be protected as one operation**.

---

# 6. Visibility Problem

**Definition:**  
One thread updates a variable, but another thread does not see the latest value.

Example:

```java
boolean flag = false;
```

T1:

```java
flag = true;
```

T2:

```java
while (!flag) {
    // keep waiting
}
```

Conceptually, T2 may continue seeing the old value:

```text
T1 → flag = true

T2 → still sees false
```

This is a **visibility problem**.

---

# 7. `volatile`

`volatile` is primarily used to provide **visibility** for a variable between threads.

```java
volatile boolean flag = false;
```

Example:

```java
class Demo {
    static volatile boolean flag = false;

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            flag = true;
        });

        Thread t2 = new Thread(() -> {
            while (!flag) {
                // wait
            }

            System.out.println("Finished");
        });

        t1.start();
        t2.start();
    }
}
```

The important idea:

```text
T1 → writes flag = true
             ↓
        visible to T2
```

### VERY IMPORTANT

`volatile` does **NOT** make compound operations atomic.

```java
volatile int count = 0;

count++;     // Still NOT atomic
```

Because:

```text
READ → INCREMENT → WRITE
```

So:

```text
volatile → visibility
volatile ≠ atomicity
```

---

# 8. Ordering Problem

**Definition:**  
The compiler/JVM/CPU can reorder operations for optimization, as long as the required single-threaded behavior is preserved.

Example:

```java
int x = 0;
boolean flag = false;
```

T1:

```java
x = 10;
flag = true;
```

T2:

```java
if (flag) {
    System.out.println(x);
}
```

We logically expect:

```text
x = 10
flag = true
→ print 10
```

Without proper synchronization, another thread may observe operations in an unexpected order.

The important interview idea is:

```text
Program order ≠ necessarily the order observed
by another thread without proper synchronization.
```

---

# 9. Thread Interference

When multiple threads interfere with each other's operations on shared state.

It can cause:

```text
Thread Interference
       ↓
 ┌─────┼─────────┐
 ↓     ↓         ↓
Race  Visibility Ordering
       ↓
Data Inconsistency
```

---

# 10. `synchronized`

`synchronized` is used to protect a critical section using a **lock/monitor**.

Example:

```java
class Counter {
    int count = 0;

    synchronized void increment() {
        count++;
    }
}
```

Now:

```text
T1 → acquires lock → increment() → releases lock
T2 → waits
T2 → acquires lock → increment() → releases lock
```

Therefore, only one thread at a time enters the synchronized critical section.

---

# 11. `volatile` vs `synchronized`

| | `volatile` | `synchronized` |
|---|---|---|
| Main purpose | Visibility | Mutual exclusion + visibility |
| Lock? | No | Yes |
| One thread at a time? | No | Yes, for the protected section |
| Makes `count++` atomic? | ❌ No | ✅ Yes, when the increment is synchronized |
| Useful for simple flags? | ✅ Yes | ✅ Yes |
| Protects critical section? | ❌ No | ✅ Yes |

### Easy way to remember

```text
volatile
   ↓
"I need everyone to see the latest value."

synchronized
   ↓
"I need only one thread at a time
to execute this critical section."
```

---

# 12. Most Important Interview Distinctions

### Race Condition

```text
Result depends on thread timing/order.
```

### Atomicity

```text
Operation cannot be divided/interleaved midway.
```

### Visibility

```text
One thread's update is not seen by another thread.
```

### Ordering

```text
Another thread may observe operations
in an unexpected order.
```

### Critical Section

```text
Code that accesses/modifies shared state.
```

### Shared Resource

```text
Data/resource accessed by multiple threads.
```

---

# 13. One Mental Model

Suppose:

```java
count++;
```

Two threads execute it.

Ask these questions:

```text
Is count shared?
        ↓
      YES
        ↓
Is count++ a compound operation?
        ↓
      YES
        ↓
Can threads interfere?
        ↓
      YES
        ↓
Race condition possible
        ↓
Protect the critical section
        ↓
synchronized / AtomicInteger
```

If instead you have:

```java
volatile boolean running = true;
```

and one thread changes:

```java
running = false;
```

while another repeatedly checks:

```java
while (running) {
}
```

the primary concern is:

```text
Visibility
    ↓
volatile
```

---

## Final Memory Trick

```text
Race Condition
→ WRONG RESULT

Atomicity
→ COMPLETE OR NOT AT ALL

Visibility
→ CAN OTHER THREAD SEE MY UPDATE?

Ordering
→ IN WHAT ORDER ARE OPERATIONS OBSERVED?

Critical Section
→ CODE THAT NEEDS PROTECTION

Shared Resource
→ DATA BEING SHARED

volatile
→ VISIBILITY

synchronized
→ LOCK + MUTUAL EXCLUSION + VISIBILITY
```

**One-line interview answer:**  
> Multithreading problems mainly arise when multiple threads access shared mutable state. Race conditions and atomicity problems concern unsafe compound operations, visibility concerns seeing the latest value, and ordering concerns the order in which operations become visible. `volatile` mainly addresses visibility, while `synchronized` provides mutual exclusion and the necessary memory-visibility guarantees.
