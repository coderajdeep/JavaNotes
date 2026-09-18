[ChatGPT](https://chatgpt.com/share/6aad6bce-d928-83e8-873d-3546e27bd62c)

# JVM Internals

The **JVM (Java Virtual Machine)** is the runtime engine that loads Java bytecode, manages memory, and executes the program.

```text
Java Source (.java)
       │
       ▼
     javac
       │
       ▼
Bytecode (.class)
       │
       ▼
      JVM
       │
 ┌─────┼──────────────────────┐
 │     │                      │
Class  Runtime Data       Execution
Loader   Areas             Engine
       │                      │
       │                 ┌────┼─────┐
       │                 │    │     │
       │             Interpreter JIT  GC
       │
       ▼
      CPU
```

---

## 1. Class Loader Subsystem

Responsible for loading `.class` files into the JVM.

### Class Loader hierarchy

```text
Bootstrap ClassLoader
        ↑
Platform ClassLoader
        ↑
Application ClassLoader
```

### Examples

| Class Loader | Examples |
|---|---|
| **Bootstrap** | `String`, `Object`, `System`, `ArrayList` |
| **Platform** | Java platform modules such as `java.sql` |
| **Application** | Classes from your application/classpath |

Example:

```java
String s = "Hello";       // Bootstrap
java.sql.Connection c;    // Platform
MyService service;        // Application
```

`String.class.getClassLoader()` returns `null`, which represents the Bootstrap ClassLoader.

### Parent Delegation

A class loader normally asks its parent to load a class first.

```text
Application
     ↓
Platform
     ↓
Bootstrap
```

This helps prevent application classes from replacing core Java classes and avoids duplicate loading.

---

# 2. Class Loading Lifecycle

When the JVM loads a class:

```text
Loading
   ↓
Linking
   ├── Verification
   ├── Preparation
   └── Resolution
   ↓
Initialization
```

### Loading
Class Loader finds the `.class` bytecode and loads it into the JVM.

### Verification
JVM checks whether the bytecode is valid and follows JVM rules.

### Preparation
Memory is allocated for static fields and they receive **default values**.

```java
static int count = 10;
```

During preparation:

```text
count = 0
```

### Resolution
Symbolic references in the class are resolved to actual classes, methods, fields, etc.

Resolution may happen lazily.

### Initialization
Static initialization is executed.

```java
static int count = 10;

static {
    System.out.println("Initialized");
}
```

Now:

```text
count = 10
static block executes
```

The compiler generates a special `<clinit>` method for class initialization when required.

**Important:** Loaded ≠ initialized.

---

# 3. Runtime Data Areas

The JVM runtime memory is broadly divided into **shared** and **per-thread** areas.

```text
Runtime Data Areas
│
├── Shared
│   ├── Heap
│   └── Method Area
│
└── Per Thread
    ├── Java Stack
    ├── PC Register
    └── Native Method Stack
```

---

## 4. Heap — Shared

The **Heap** stores Java objects.

```java
Person p = new Person();
```

Conceptually:

```text
Stack
  │
  │ p
  ▼
Heap
┌───────────────┐
│ Person object │
└───────────────┘
```

The Heap is **shared by all JVM threads**.

The Garbage Collector manages this memory.

---

## 5. Method Area — Shared

The JVM specification defines the **Method Area** as the logical area for class-level information.

It contains information such as:

- Class metadata
- Method information
- Field information
- Runtime constant pool
- Other class-related information

### HotSpot implementation

In HotSpot JVM, much of this metadata is stored in **Metaspace**.

```text
JVM Specification
       │
       ▼
  Method Area
       │
       ▼
HotSpot implementation
       │
       ▼
   Metaspace
```

**Metaspace is shared**, not per-thread.

```text
Thread 1 ──┐
Thread 2 ──┼──→ Shared Metaspace
Thread 3 ──┘
```

Metaspace uses **native memory**, rather than being part of the normal Java Heap.

---

# 6. Java Stack — Per Thread

Every JVM thread has its **own Java Stack**.

Each method call creates a **Stack Frame**.

Example:

```java
main() {
    calculate();
}

calculate() {
    int x = 10;
}
```

Conceptually:

```text
Thread
  │
  ▼
Java Stack
┌─────────────────┐
│ calculate()     │ ← current frame
│ x = 10          │
├─────────────────┤
│ main()          │
└─────────────────┘
```

When `calculate()` returns, its frame is removed.

Each frame contains information needed for that method execution, such as:

- Local variables
- Operand stack
- Reference to the runtime constant pool
- Return information

Deep recursion can result in:

```text
StackOverflowError
```

---

# 7. PC Register — Per Thread

Each JVM thread has its own **Program Counter (PC) Register**.

It keeps track of the bytecode instruction currently being executed / to be executed by that thread.

```text
Thread 1 → PC → instruction 25
Thread 2 → PC → instruction 73
```

This allows each thread to independently keep track of its execution position.

---

# 8. Native Method Stack — Per Thread

Used when JVM executes **native methods**, typically through JNI.

```text
Java Code
    │
    ▼
   JNI
    │
    ▼
Native Code (C/C++)
```

Each thread has its own native method stack.

---

# 9. Execution Engine

The Execution Engine executes the bytecode.

Main components:

```text
Execution Engine
│
├── Interpreter
├── JIT Compiler
└── Garbage Collector
```

---

## 10. Interpreter

The Interpreter executes bytecode instructions directly.

```text
Bytecode
   ↓
Interpreter
   ↓
CPU
```

It allows the JVM to start executing code without compiling the entire application first.

---

# 11. JIT Compiler

**JIT = Just-In-Time Compiler**

The JVM identifies frequently executed **hot code** and compiles it into native machine code.

```text
Bytecode
   ↓
JIT Compiler
   ↓
Native Machine Code
   ↓
CPU
```

Therefore, modern JVM execution is not simply "Java is interpreted."

A simplified model is:

```text
                    Bytecode
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        Interpreter             JIT
             │                   │
             ▼                   ▼
        Execute bytecode    Native code
                                 │
                                 ▼
                                CPU
```

JIT compilation can also perform runtime optimizations based on observed program behavior.

---

# 12. Garbage Collector

The GC automatically reclaims memory occupied by objects that are no longer reachable.

```text
Heap

Object A → reachable
Object B → unreachable
Object C → reachable
Object D → unreachable

             ↓
           GC
             ↓
   Memory of B and D reclaimed
```

Common HotSpot collectors include:

- G1
- ZGC
- Shenandoah
- Serial
- Parallel

The exact GC behavior depends on the selected collector.

---

# 13. JNI and Native Libraries

Java can interact with native code using **JNI (Java Native Interface)**.

```text
Java Application
       │
       ▼
      JNI
       │
       ▼
Native Library
       │
       ▼
Operating System
```

---

# 14. Shared vs Per-Thread — Important Interview Table

| Runtime Area | Shared? | Main Purpose |
|---|---|---|
| **Heap** | ✅ Shared | Java objects |
| **Method Area** | ✅ Shared | Class-level information |
| **Metaspace** | ✅ Shared | HotSpot's implementation for much of class metadata |
| **Java Stack** | ❌ Per thread | Method calls / stack frames |
| **PC Register** | ❌ Per thread | Current bytecode execution position |
| **Native Method Stack** | ❌ Per thread | Native method execution |

### Easy memory trick

```text
SHARED
├── Heap
└── Method Area / Metaspace

PER THREAD
├── Stack
├── PC
└── Native Stack
```

---

# 15. Complete JVM Picture

```text
                         JVM
                          │
          ┌───────────────┴────────────────┐
          │                                │
    Class Loader                     Runtime Data Areas
          │                                │
          │                     ┌──────────┴──────────┐
          │                     │                     │
          │                  Shared               Per Thread
          │                     │                     │
          │               ┌─────┴─────┐       ┌──────┼──────┐
          │               │           │       │      │      │
          │             Heap    Method Area  Stack   PC   Native
          │                         │                       Stack
          │                     Metaspace
          │
          ▼
     Loaded Classes
          │
          ▼
    Execution Engine
          │
     ┌────┼─────┐
     │    │     │
 Interpreter JIT  GC
     │    │
     └────┴───────► Native Machine Code
                         │
                         ▼
                        CPU
```

---

# Key Interview Points

1. **JVM executes bytecode**, not Java source code.
2. **Class Loader loads `.class` files** into the JVM.
3. Class loading lifecycle:

```text
Loading → Linking → Initialization
             │
       ┌─────┼─────┐
 Verification Preparation Resolution
```

4. **Heap is shared** among threads.
5. **Java Stack, PC Register and Native Method Stack are per-thread.**
6. **Method Area is a JVM specification concept.**
7. **Metaspace is a HotSpot implementation detail** and is shared.
8. Metaspace uses **native memory**, not the Java Heap.
9. **Interpreter executes bytecode.**
10. **JIT compiles frequently executed code into native machine code.**
11. **GC manages/reclaims unreachable objects in the Heap.**
12. **Loaded does not mean initialized.**
13. Parent Delegation is used by the class-loader hierarchy.
14. The JVM is **not just an interpreter**; modern JVMs combine interpretation and JIT compilation.
