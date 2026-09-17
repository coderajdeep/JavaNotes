[ChatGPT](https://chatgpt.com/share/6aab9fc9-d530-83ee-a89a-5192bef46c69)

[Youtube](https://youtu.be/1CJbB6SzjVw)

# Java HashMap — Internal Implementation

## 1. Internal Structure

`HashMap` internally uses:

**Array + Linked List + Red-Black Tree**

```text
HashMap
   |
   v
Node<K,V>[] table
   |
   +--> bucket 0
   +--> bucket 1
   +--> bucket 2 --> Node --> Node
   +--> bucket 3
   ...
```

Each node conceptually contains:

```java
class Node<K,V> {
    int hash;
    K key;
    V value;
    Node<K,V> next;
}
```

The `next` reference forms a linked list when multiple keys map to the same bucket.

---

## 2. Initial Table Size

When we create:

```java
HashMap<String, Integer> map = new HashMap<>();
```

the internal table is initially **not created**.

```text
table = null
```

This is **lazy initialization**.

On the first `put()`, the table is created.

For the default `HashMap`:

```text
Initial capacity = 16
```

So:

```text
Before first put:
table → null

After first put:
table → array of 16 buckets
```

---

## 3. How `put()` Finds a Bucket

For:

```java
map.put("Rajdeep", 100);
```

the general process is:

```text
key
 ↓
hashCode()
 ↓
hash spreading
 ↓
bucket index
 ↓
table[index]
```

---

## 4. Hash Spreading

Hash spreading mixes the **higher bits** of the hash code into the lower bits.

Conceptually:

```java
int hash = key.hashCode();
hash = hash ^ (hash >>> 16);
```

### Why?

Bucket selection mainly uses the lower bits:

```java
index = (n - 1) & hash;
```

So if the lower bits of different hash codes are similar, many keys may end up in the same bucket.

Hash spreading makes the lower bits depend partly on the higher bits, helping reduce collisions.

### Why `>>> 16`?

A Java `int` is 32 bits:

```text
AAAAAAAAAAAAAAAA BBBBBBBBBBBBBBBB
       upper             lower
```

```java
h >>> 16
```

moves the upper 16 bits into the lower 16-bit position and fills the left side with `0`:

```text
0000000000000000 AAAAAAAAAAAAAAAA
```

Java has two right-shift operators:

```text
>>   → signed right shift; preserves the sign bit
>>>  → unsigned right shift; fills with 0
```

`HashMap` uses `>>>` so the shift behaves consistently even when the hash is negative.

---

## 5. Calculating the Bucket Index

The bucket index is approximately:

```java
index = (n - 1) & hash;
```

where:

```text
n = table.length
```

For example, if:

```text
n = 16
```

then:

```text
n - 1 = 15
```

Therefore:

```java
index = hash & 15;
```

The result is always between:

```text
0 and 15
```

which corresponds to the 16 buckets.

---

## 6. Why `(n - 1) & hash`?

`HashMap` keeps its table capacity as a **power of 2**:

```text
16 → 32 → 64 → 128 → ...
```

If:

```text
n = 16
n - 1 = 15 = 00001111
```

then:

```java
hash & 15
```

effectively extracts the lower bits needed for the bucket index.

For a power-of-two `n`:

```java
hash & (n - 1)
```

is equivalent to the bucket calculation:

```java
hash % n
```

while being very efficient.

---

## 7. What if the Bucket Is Empty?

If:

```java
table[index] == null
```

a new node is placed there.

```text
table[5]
   |
   v
+----------------+
| key = Rajdeep  |
| value = 100    |
+----------------+
```

---

## 8. Collision

Different keys can produce the same bucket index.

Example:

```text
"A"       → bucket 5
"Rajdeep" → bucket 5
```

They are stored together:

```text
bucket 5
   |
   v
Node("A", 200)
   |
   v
Node("Rajdeep", 100)
```

Initially, collisions are handled using a **linked list**.

---

## 9. Linked List → Red-Black Tree

If a bucket becomes heavily populated, Java 8+ `HashMap` can convert the bucket's linked list into a **Red-Black Tree**.

Important thresholds:

```java
TREEIFY_THRESHOLD = 8
UNTREEIFY_THRESHOLD = 6
MIN_TREEIFY_CAPACITY = 64
```

The table may be resized instead of immediately treeifying when the table is still small.

Complexity:

```text
Linked List → O(n)
Red-Black Tree → O(log n)
```

---

## 10. How `get()` Works

For:

```java
map.get("Rajdeep");
```

the process is:

```text
"Rajdeep"
    ↓
hashCode()
    ↓
hash spreading
    ↓
calculate bucket index
    ↓
table[index]
    ↓
compare hash
    ↓
compare keys using equals()
    ↓
return value
```

HashMap therefore uses **both `hashCode()` and `equals()`** to identify a key.

---

## 11. `null` Key

`HashMap` allows **one `null` key**.

Conceptually:

```text
null
 ↓
hash = 0
 ↓
(16 - 1) & 0
 ↓
0
 ↓
bucket 0
```

### Important

**Bucket 0 is NOT reserved for the `null` key.**

A normal key can also produce:

```text
(n - 1) & hash == 0
```

and therefore go into bucket 0.

Example:

```text
bucket 0
   |
   v
Node(null, 100)
   |
   v
Node("A", 200)
```

This is simply a collision.

HashMap distinguishes them by checking the actual key:

```java
key == null
```

or:

```java
key.equals(node.key)
```

So:

> **`null` has hash 0; bucket 0 is not exclusively for `null`.**

---

## 12. Duplicate Keys

If:

```java
map.put("Rajdeep", 100);
map.put("Rajdeep", 500);
```

HashMap does not create two entries.

It finds the existing key using hash + `equals()` and updates the value:

```text
"Rajdeep" → 500
```

---

## 13. Resizing

Default load factor:

```text
0.75
```

For capacity `16`:

```text
threshold = 16 × 0.75
          = 12
```

When the map grows beyond the threshold, the table is resized:

```text
16 → 32 → 64 → 128 → ...
```

Increasing capacity reduces collisions and maintains efficient lookup.

During resizing, Java can efficiently determine whether an entry stays at its old index or moves by the old capacity.

---

## 14. `equals()` / `hashCode()` Contract

For keys:

```java
a.equals(b) == true
```

must imply:

```java
a.hashCode() == b.hashCode()
```

Otherwise, `HashMap` can behave unexpectedly.

Remember:

```text
hashCode()
    ↓
hash spreading
    ↓
find bucket
    ↓
hash comparison
    ↓
equals()
    ↓
key found
```

---

## Key Takeaways

1. **HashMap = array of buckets + linked lists + Red-Black Trees.**
2. The table is **lazily initialized**; initially `table == null`.
3. Default initial capacity is **16** when the table is first created.
4. HashMap applies **hash spreading** before calculating the bucket.
5. Hash spreading uses:

```java
h ^ (h >>> 16)
```

6. `>>>` is a **zero-filled right shift**; `>>` preserves the sign bit.
7. Bucket index:

```java
(n - 1) & hash
```

8. Capacity is kept as a **power of 2**, enabling efficient bucket calculation.
9. Multiple keys can have the same bucket → **collision**.
10. Heavy collisions can convert a linked list into a **Red-Black Tree**.
11. `null` key has hash `0`, but **bucket 0 is not reserved for null**.
12. HashMap uses **both `hashCode()` and `equals()`**.
13. Default load factor is **0.75**.
14. When the threshold is exceeded, HashMap **resizes**.
15. Average `get()` / `put()` complexity is **O(1)**.
16. Treeified buckets provide approximately **O(log n)** lookup.
