[ChatGPT](https://chatgpt.com/share/6ab63ec5-90f0-83ee-b864-42d01599fc94)

# Java `ThreadLocal`

## 1. What is `ThreadLocal`?

`ThreadLocal<T>` provides **thread-local storage**.

> Each thread gets its **own independent value**.

Even if multiple threads use the same `ThreadLocal` object, each thread sees its own value.

```text
ThreadLocal
     |
     +── Thread-1 → "REQ-101"
     +── Thread-2 → "REQ-102"
     +── Thread-3 → "REQ-103"
```

---

## 2. Why use `ThreadLocal`?

Use it when data:

- belongs to a particular thread
- should not be shared between threads
- needs to be accessible across multiple methods without passing it as a parameter

Common examples:

- Request ID / correlation ID
- Logging context
- Transaction context
- User/request-specific information

---

## 3. Important Methods

### `set()`

Stores a value for the current thread.

```java
threadLocal.set(value);
```

### `get()`

Retrieves the value belonging to the current thread.

```java
threadLocal.get();
```

### `remove()`

Removes the current thread's value.

```java
threadLocal.remove();
```

**Always consider calling `remove()` when using thread pools.**

---

## 4. How does it work internally?

Conceptually, each `Thread` has a `ThreadLocalMap`.

```text
Thread-1
   |
   +-- ThreadLocalMap
          |
          +-- ThreadLocal A → "REQ-101"
          +-- ThreadLocal B → 100


Thread-2
   |
   +-- ThreadLocalMap
          |
          +-- ThreadLocal A → "REQ-102"
          +-- ThreadLocal B → 200
```

So:

```java
threadLocal.get()
```

essentially means:

> "Give me the value associated with this `ThreadLocal` for the current thread."

---

# 5. Full Example — Request ID

A common backend use case is storing a request/correlation ID.

Without `ThreadLocal`, we might have to pass the request ID through every method:

```text
handleRequest(requestId)
       ↓
serviceA(requestId)
       ↓
serviceB(requestId)
       ↓
repository(requestId)
```

With `ThreadLocal`, the request ID can be accessed by methods running on the same thread.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadLocalRequestIdExample {

    private static final ThreadLocal<String> requestId =
            new ThreadLocal<>();

    static void handleRequest(String id) {
        requestId.set(id);

        try {
            serviceA();
        } finally {
            // Very important when using thread pools
            requestId.remove();
        }
    }

    static void serviceA() {
        serviceB();
    }

    static void serviceB() {
        System.out.println(
                Thread.currentThread().getName()
                        + " -> Request ID: "
                        + requestId.get()
        );
    }

    public static void main(String[] args) {

        ExecutorService executor =
                Executors.newFixedThreadPool(2);

        executor.submit(() -> handleRequest("REQ-101"));
        executor.submit(() -> handleRequest("REQ-102"));
        executor.submit(() -> handleRequest("REQ-103"));
        executor.submit(() -> handleRequest("REQ-104"));

        executor.shutdown();
    }
}
```

Possible output:

```text
pool-1-thread-1 -> Request ID: REQ-101
pool-1-thread-2 -> Request ID: REQ-102
pool-1-thread-1 -> Request ID: REQ-103
pool-1-thread-2 -> Request ID: REQ-104
```

Notice that the same thread can process multiple requests:

```text
Thread-1 → REQ-101
Thread-1 → REQ-103

Thread-2 → REQ-102
Thread-2 → REQ-104
```

That's why this is important:

```java
finally {
    requestId.remove();
}
```

Without `remove()`, a pooled thread could retain the previous request's data.

---

## 6. `ThreadLocal` vs `synchronized`

### `synchronized`

Used when threads need to safely access **shared data**.

```text
Thread-1 ──┐
Thread-2 ──┼──> Shared Data
Thread-3 ──┘
```

### `ThreadLocal`

Used when each thread should have **separate data**.

```text
Thread-1 → Data A

Thread-2 → Data B

Thread-3 → Data C
```

So:

> `synchronized` → **shared data + controlled access**

> `ThreadLocal` → **separate data per thread**

---

## 7. `ThreadLocal` vs `AtomicInteger`

### `AtomicInteger`

One value is shared by all threads:

```text
Thread-1 ──┐
Thread-2 ──┼──> AtomicInteger
Thread-3 ──┘
```

### `ThreadLocal<Integer>`

Each thread has its own value:

```text
Thread-1 → 10
Thread-2 → 20
Thread-3 → 30
```

Therefore:

> `AtomicInteger` → **one shared value, safely modified**

> `ThreadLocal` → **one independent value per thread**

---

## 8. Key Interview Points

- `ThreadLocal` provides **thread-local storage**.
- Each thread has its **own value**.
- The `ThreadLocal` object itself can be shared.
- Internally, values are associated with the current thread's `ThreadLocalMap`.
- `set()` → set current thread's value.
- `get()` → get current thread's value.
- `remove()` → remove current thread's value.
- Particularly useful for **request/correlation IDs and contextual data**.
- Be careful with **thread pools** because threads are reused.
- Use `remove()` in a `finally` block to prevent stale context from leaking into another task.

### One-line definition

> **`ThreadLocal` allows each thread to maintain its own independent value without sharing that value with other threads.**
