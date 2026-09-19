[ChatGPT](https://chatgpt.com/share/6aae24c8-bb10-83ee-947b-195856c6bf0b)

## Java: Process & Thread — Short Note

### 1. Process

A **process** is a running instance of a program.

When you execute:

```bash
java Main
```

the OS creates a **Java process**, which contains a JVM.

```text
Java Process / JVM
├── Heap
├── Metaspace
└── Threads
```

A process has its own address space and resources and is relatively isolated from other processes.

---

### 2. Thread

A **thread** is an independent **path of execution inside a process**.

A single Java process can contain multiple threads:

```text
Java Process / JVM
│
├── main thread
├── Thread-1
├── Thread-2
├── GC threads
└── JIT threads
```

Threads within the same process share resources such as the **heap**, but each thread has its own **stack**.

```text
Java Process
│
├── Shared Heap
│
├── Thread 1 → Own Stack
├── Thread 2 → Own Stack
└── Thread 3 → Own Stack
```

### 3. Main Thread

The **JVM automatically creates the main thread**.

It executes:

```java
public static void main(String[] args)
```

So:

```text
main thread
     ↓
executes main()
```

It is **not a thread explicitly created by your application**.

If you write:

```java
Thread t = new Thread(() -> {
    // task
});

t.start();
```

then your application creates another thread.

### 4. Thread = Lightweight Process?

It is common to call a thread a **"lightweight process"**, because it represents an independent execution flow but shares resources with other threads in the same process.

However, technically:

> **Process = resource container**  
> **Thread = execution flow within that container**

**Interview takeaway:** A process can contain multiple threads, and threads belonging to the same process share its memory/resources while maintaining their own execution state and stack.
