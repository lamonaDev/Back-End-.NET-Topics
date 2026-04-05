# SortedList, Dictionary & HashSet — Deep Dive

## 🧠 Overview

This guide dives deep into three critical C# collections — `SortedList<K,V>`, `Dictionary<K,V>`, and `HashSet<T>` — covering their internal data structures, hashing theory, collision handling, and the `GetHashCode`/`Equals` contract.

---

## 🟣 Part 1: SortedList\<TKey, TValue\>

### What Is It?

`SortedList<TKey, TValue>` is a key-value collection backed by **two parallel sorted arrays** — one for keys, one for values. Keys are **always maintained in ascending order**, and it uniquely supports **index-based positional access** (`Keys[0]`, `Values[2]`).

---

### 🌍 Real-World Analogy

Imagine a **phone book** — names (keys) are sorted alphabetically, and phone numbers (values) sit next to them. You can flip to a position by index, look up by name with binary search, but adding someone in the middle means shifting everyone else down the page.

---

### ⚙️ Internal Structure — Two Parallel Arrays

```
After: marks["Ahmed"]=60, marks["Bola"]=65, marks["Mohamed"]=50, marks["Salma"]=80

keys[]                 values[]
  0   "Ahmed"            0   60
  1   "Bola"             1   65
  2   "Mohamed"          2   50
  3   "Salma"            3   80

keys[i] always maps to values[i].
Binary search on keys[] → O(log n) lookup.
```

---

### 💻 Code Examples

```csharp
SortedList<string, int> marks = new()
{
    ["Mohamed"] = 50,
    ["Salma"]   = 80,
    ["Ahmed"]   = 60,
    ["Bola"]    = 65
};

// Iteration is ALWAYS in key-sorted order
foreach (var kvp in marks)
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
// → Ahmed:60, Bola:65, Mohamed:50, Salma:80

// ── Index access — UNIQUE to SortedList! ──────────────────────
Console.WriteLine(marks.Keys[0]);   // O(1) → "Ahmed"
Console.WriteLine(marks.Values[0]); // O(1) → 60

// ── Add / Remove ───────────────────────────────────────────────
marks.Add("Laila", 75);             // O(n) — binary search to find position + shift
marks.Remove("Ahmed");              // O(n) — find + shift remaining elements
marks["Ahmed"] = 95;                // Add OR update

// ── Safe Access ────────────────────────────────────────────────
// ❌ marks["Ahmed"] throws KeyNotFoundException if missing
if (marks.TryGetValue("Ahmed", out int val))  // O(log n) ← recommended
    Console.WriteLine(val);
else
    Console.WriteLine("Ahmed Not Found");

int pos = marks.IndexOfKey("Bola");  // O(log n) — returns position in sorted order
```

### Complexity

| Operation | Complexity | Reason |
|---|---|---|
| `Add(key, val)` | O(n) | Binary search O(log n) + array shift O(n) |
| `Remove(key)` | O(n) | Find + shift remaining |
| `this[key]` (get) | O(log n) | Binary search on keys array |
| `ContainsKey` | O(log n) | Binary search |
| `Keys[i]` | O(1) | Direct array index |
| `IndexOfKey` | O(log n) | Binary search |

---

### ⚙️ How Add() Works Internally

```csharp
// Suppose: keys = ["Ahmed","Bola","Salma"], _size = 3
// We call: marks.Add("Mohamed", 50);

// Step 1: Binary search → insertion index = 2 (between "Bola" and "Salma")
int i = Array.BinarySearch(keys, 0, _size, "Mohamed", comparer); // O(log n)
// If i >= 0 → duplicate key → throws ArgumentException

// Step 2: Ensure capacity (doubles like List<T> if needed)

// Step 3: SHIFT keys[] and values[] right to make room  ← O(n) BOTTLENECK
Array.Copy(keys,   i, keys,   i + 1, _size - i);
Array.Copy(values, i, values, i + 1, _size - i);

// Step 4: Insert at found position
keys[i]   = "Mohamed";
values[i] = 50;
_size++;
version++;
```

> ⚠️ **Why Add is O(n) even though search is O(log n):** Finding the insertion point takes O(log n) via binary search — but shifting all elements after the point takes O(n). The **shift is the bottleneck**.

---

### 💡 When to Use SortedList

```
✅ Need key-value pairs
✅ Need them always sorted
✅ Need positional access (Keys[i])
✅ Inserts are infrequent (it's heavy on insert)

→ Use SortedDictionary if inserts are frequent (tree-based, O(log n) insert)
→ Use Dictionary if you don't need sorted order (fastest, O(1) all ops)
```

---

## 🔵 Part 2: Dictionary\<TKey, TValue\>

### The Problem: Why We Need Hashing

Before hashing, finding an element meant searching through the collection.

```csharp
// Linear Search — O(n)
// 1,000,000 students → worst case: 1,000,000 comparisons
for (int i = 0; i < names.Length; i++)
    if (names[i] == "Salma") return i;

// Binary Search — O(log n) (requires sorted data)
// 1,000,000 elements → ~20 comparisons
// BUT: requires sorting first O(n log n), and inserts re-sort
Array.BinarySearch(sortedNames, "Salma");
```

### 💡 The Core Idea: O(1) — Constant Time

```csharp
// Instead of searching, CALCULATE where the element is stored:
int hashCode = "Salma".GetHashCode();       // → some big integer, e.g. 482947
int index    = Math.Abs(hashCode) % tableSize; // → e.g. 482947 % 7 = 2
table[index] = 90; // store
var mark = table[index]; // retrieve in O(1) — no loop!
```

---

### 🔥 Collision Strategies

A **collision** occurs when two different keys produce the same array index. This is inevitable — the question is how to handle it.

#### Strategy A: Open Addressing — all entries stored IN the array

```
A1 — Linear Probing:   index = (hash + i) % n      where i = 1, 2, 3...
     Probes sequentially. Simple but creates clustering.

A2 — Quadratic Probing: index = (hash + i²) % n
     Probes jump further each step. Reduces clustering.

A3 — Double Hashing:   index = (hash1 + i × hash2) % n   ← BEST open addressing
     Step size varies per key → minimal clustering.
```

| Strategy | Formula | Clustering | Notes |
|---|---|---|---|
| Linear Probing | `(h + i) % n` | Primary (bad) | Simple, cache-friendly, bunches |
| Quadratic Probing | `(h + i²) % n` | Secondary | Better spread |
| Double Hashing | `(h1 + i×h2) % n` | Minimal | Best distribution |

#### Strategy B: Separate Chaining — each bucket holds a list

```
bucket 0:  null
bucket 1:  null
bucket 2:  [Mona→300] → [Ahmed→100] → null  ← both hash to 2
bucket 3:  [Omar→200] → null
bucket 4:  null

To find "Ahmed": hash → bucket 2 → scan chain: Mona? No → Ahmed? ✅
```

> **.NET's `Dictionary<K,V>` uses array-based chaining** (entries stored in an `Entry[]` struct array, chained via integer `next` index fields — not heap-allocated linked list nodes). Result: O(1) average for all operations.

---

### 💻 Basic Dictionary Usage

```csharp
Dictionary<int, string> students = new()
{
    [101] = "Ali",
    [102] = "Mohamed",
    [103] = "Salma"
};

// ── Three ways to Add — know the difference! ──────────────────
students[104]          = "Laila"; // ✅ Add OR Update — never throws
students.Add(104, "Mona");        // ⚠️ Add ONLY — throws ArgumentException if exists!
bool ok = students.TryAdd(101, "X"); // ✅ Add ONLY — returns false if exists, no throw

// ── Safe Access Patterns ───────────────────────────────────────
// ❌ Dangerous — throws KeyNotFoundException if missing
string name = students[999];

// ✅ Pattern 1: ContainsKey (two lookups)
if (students.ContainsKey(999))
    Console.WriteLine(students[999]);

// ✅ Pattern 2: TryGetValue — BEST. One lookup. Safe. Outputs value.
if (students.TryGetValue(999, out string? n))
    Console.WriteLine($"Found: {n}");
else
    Console.WriteLine("Not found");

// ✅ Pattern 3: GetValueOrDefault — returns null/default if missing
string? result = students.GetValueOrDefault(999);
```

### Complexity

| Operation | Complexity |
|---|---|
| `this[key]` get/set | O(1) avg |
| `Add` / `TryAdd` | O(1) avg |
| `Remove(key)` | O(1) avg |
| `ContainsKey` | O(1) avg |
| `ContainsValue` | O(n) — full scan! |
| `TryGetValue` | O(1) avg |

---

### 💻 Iterating

```csharp
Dictionary<string, int> ages = new()
{
    ["Ahmed"]   = 25,
    ["Mohamed"] = 30,
    ["Salma"]   = 30
};

// Deconstruct KeyValuePair
foreach (var (name, age) in ages)
    Console.WriteLine($"{name}: {age}");

// Keys only / Values only
foreach (string key   in ages.Keys)   Console.Write($"{key} ");
foreach (int   value  in ages.Values) Console.Write($"{value} ");

bool has30 = ages.ContainsValue(30); // O(n) — linear scan of values
ages.Remove("Mohamed");              // O(1) average
ages.Clear();                        // O(n) — nulls all entries
```

---

### ⚙️ How Dictionary Works Internally

```csharp
// Two core arrays:
int[]   _buckets; // routing table — stores 1-based index into _entries (0 = empty bucket)
Entry[] _entries; // all data lives here

struct Entry {
    uint hashCode;  // pre-computed hash
    int  next;      // chain link: index of next entry in bucket (-1 = end)
    TKey   key;
    TValue value;
}
```

### Step-by-Step: `Add("Ali", 100)`

```
1. hashCode = GetHashCode("Ali")          → e.g. 17
2. bucketIndex = 17 % _buckets.Length     → e.g. 17 % 5 = 2
3. _buckets[2] == 0 → empty bucket, no collision
4. Write entry:  _entries[0] = { hash=17, next=-1, key="Ali", value=100 }
5. _buckets[2] = 1  (1-based: entry at index 0)

_buckets = [0, 0, 1, 0, 0]
_entries[0] = { hash=17, next=-1, "Ali" → 100 }
```

### Step-by-Step: `Add("Mona", 300)` — Collision!

```
1. "Mona" → hashCode=22 → bucket 22%5=2 → collision! _buckets[2] ≠ 0

2. Write new entry at index 2:
   _entries[2] = { hash=22, next=0 ← old head, key="Mona", value=300 }

3. Update bucket head: _buckets[2] = 3 (entry at index 2 is new head)

Bucket 2 chain: _entries[2]("Mona") → _entries[0]("Ali") → end(-1)
```

### Step-by-Step: Lookup `dict["Ali"]`

```
1. hash("Ali") = 17
2. bucket = 17 % 5 = 2
3. chain head = _buckets[2] - 1 = 2 → _entries[2]
4. _entries[2].key = "Mona" ≠ "Ali" → follow next = 0 → _entries[0]
5. _entries[0].key = "Ali" ✅ → return _entries[0].value = 100
```

---

### ⚠️ The Mutable Key Trap

```csharp
var std01 = new Student(10, "Salma", 50);
Dictionary<Student, int> marks = new();
marks[std01] = 500;

Console.WriteLine(marks.ContainsKey(std01)); // true

std01.Name = "Laila"; // ← MUTATE the key object!
// GetHashCode now returns a DIFFERENT value (Name is in Combine(Id, Name))
// Dictionary looks in the WRONG bucket → entry is permanently lost!
Console.WriteLine(marks.ContainsKey(std01)); // false 💥

// ✅ Fix: use immutable record as key
public record StudentRecord(int Id, string Name, int Age);
// record auto-generates value-based GetHashCode + Equals
// rec.Name = "Laila" → ❌ won't compile — init-only properties
```

---

### 🔑 Why You Need Both GetHashCode AND Equals

```
Dictionary uses two-phase lookup:

Phase 1 — GetHashCode → Find the BUCKET
  Hash codes are NOT unique. Many keys → same bucket.
  GetHashCode is just a fast filter. Not the final answer.

Phase 2 — Equals → Confirm the exact KEY
  Walk the bucket's chain, calling Equals() on each entry.
  This is the definitive check.

THE GOLDEN RULE:
  If two objects are Equals() → they MUST return the same GetHashCode().
  Violation = Dictionary is permanently broken for those keys.
```

```csharp
// Class with GetHashCode but NO Equals override
var s1 = new Student(10, "Salma", 20);
var s2 = new Student(10, "Salma", 20);

s1.GetHashCode() == s2.GetHashCode(); // true (same Id+Name)
s1.Equals(s2);                        // FALSE — inherits object.Equals → reference equality
// Dictionary treats s1 and s2 as DIFFERENT keys!

// Record — auto-generates BOTH correctly
var r1 = new StudentRecord(10, "Salma", 20);
var r2 = new StudentRecord(10, "Salma", 20);

r1.GetHashCode() == r2.GetHashCode(); // true
r1.Equals(r2);                        // TRUE — value equality
// Dictionary correctly treats r1 and r2 as the SAME key ✅
```

| Scenario | GetHashCode | Equals | Dictionary Result |
|---|---|---|---|
| Same object | Same | `true` | Same key — value overwritten |
| Different objects, same data (class, no override) | May differ | `false` (reference) | Treated as different keys |
| Different objects, same data (record) | Same | `true` | Same key — value overwritten ✅ |
| Mutated key after insert | Changed | May not match | Entry permanently lost 💥 |

---

## 🟢 Part 3: HashSet\<T\>

### What Is It?

`HashSet<T>` is a **hash table that stores only keys, no values**. Every element is unique. Provides O(1) Add, Remove, and Contains. Built for fast membership testing and mathematical set operations.

---

### 🌍 Real-World Analogy

A **guest list** at an event. You don't store extra info about each guest — just whether they're on the list or not. Checking membership is instant (O(1)). Duplicates are automatically ignored.

---

### ⚙️ Internals

```csharp
// HashSet shares the same _buckets + _entries design as Dictionary.
// The ONLY difference: no TValue in the Entry struct.

// Dictionary Entry:
struct Entry { uint hashCode; int next; TKey key; TValue value; }

// HashSet Entry:
struct Entry { uint hashCode; int next; T value; } // just the element — no separate value
```

---

### 💻 Basic Operations

```csharp
HashSet<int> numbers = [1, 2, 3, 2, 1]; // silently deduplicates → {1, 2, 3}

// Add — returns bool: true if added, false if duplicate
bool added = numbers.Add(4); // true — new item
bool again = numbers.Add(4); // false — duplicate, silently ignored

// Contains — O(1)! The #1 reason to use HashSet over List
bool has3 = numbers.Contains(3); // O(1) ← same cost as Dictionary lookup

// Remove — returns true if found and removed
numbers.Remove(2); // O(1)

// ── Deduplicate a List ─────────────────────────────────────────
List<int> withDupes = [1, 1, 2, 2, 3];
List<int> unique = withDupes.ToHashSet().ToList(); // → [1, 2, 3]

// ── Fast membership test (replaces slow List.Contains) ─────────
HashSet<string> validRoles = ["Admin", "Editor", "Viewer"];
if (validRoles.Contains(userRole)) // O(1) — vs List.Contains O(n)!
    Console.WriteLine("Access granted");
```

| Operation | Complexity |
|---|---|
| `Add(x)` | O(1) avg |
| `Remove(x)` | O(1) avg |
| `Contains(x)` | O(1) avg ← key advantage |
| `Count` | O(1) |
| Set operations | O(n) |

---

### 💻 Set Math Operations

```csharp
HashSet<int> a = [1, 2, 3, 4];
HashSet<int> b = [3, 4, 5, 6];

// ⚠️ All these modify the set IN PLACE — copy first to preserve original!
var copy = new HashSet<int>(a);

copy.UnionWith(b);             // O(n) → {1,2,3,4,5,6}  — everything from both (A ∪ B)
copy.IntersectWith(b);         // O(n) → {3,4}           — only shared elements (A ∩ B)
copy.ExceptWith(b);            // O(n) → {1,2}           — in A but NOT in B (A − B)
copy.SymmetricExceptWith(b);   // O(n) → {1,2,5,6}       — in either but NOT both (A △ B)
```

### Relationship Checks

```csharp
HashSet<int> small = [3, 4];

small.IsSubsetOf(a);            // true — every element of small is in a
small.IsProperSubsetOf(a);      // true — subset AND a has MORE elements
a.IsSupersetOf(small);          // true — a contains everything in small
a.Overlaps(b);                  // true — at least one common element
a.SetEquals(b);                 // false — not exactly the same elements
```

> ⚠️ `UnionWith`, `IntersectWith`, `ExceptWith`, and `SymmetricExceptWith` all **modify the original set**. Copy first: `new HashSet<T>(original)`.

---

## 📊 Final Comparison: Dictionary vs SortedList vs HashSet vs List

| Aspect | `Dictionary<K,V>` | `SortedList<K,V>` | `SortedDictionary<K,V>` | `HashSet<T>` | `List<T>` |
|---|---|---|---|---|---|
| Structure | Hash table | Two sorted arrays | Red-Black tree | Hash table | Dynamic array |
| Key Lookup | O(1) avg | O(log n) | O(log n) | O(1) avg | O(n) |
| Insert | O(1) avg | O(n) ← shift | O(log n) | O(1) avg | O(1) amortized |
| Delete | O(1) avg | O(n) ← shift | O(log n) | O(1) avg | O(n) |
| Ordered Iteration | ❌ | ✅ sorted | ✅ sorted | ❌ | ✅ insertion order |
| Index Access (pos) | ❌ | ✅ `Keys[i]` | ❌ | ❌ | ✅ `list[i]` |
| Duplicates | Keys unique | Keys unique | Keys unique | ❌ unique | ✅ allowed |
| Set operations | ❌ | ❌ | ❌ | ✅ | ❌ |
| Best for | Fast lookup | Sorted + positional | Sorted + inserts | Unique + membership | General ordered |

### When to Choose

```
Dictionary<K,V>       → 95% of key-value needs. Fastest all-around.
SortedList<K,V>       → Need sorted + Keys[i] positional access + few inserts.
SortedDictionary<K,V> → Need sorted + frequent inserts (no positional access).
HashSet<T>            → Uniqueness, O(1) membership, set math.
List<T>               → Ordered, indexed, duplicates allowed, general purpose.
```

---

## 📌 Summary

> `SortedList` uses two parallel sorted arrays — O(log n) lookup but O(n) insert due to shifting. `Dictionary` uses array-based separate chaining for O(1) average on all operations — but requires correct `GetHashCode`/`Equals` implementations and **immutable keys**. `HashSet` is Dictionary without values — perfect for uniqueness and O(1) membership tests. Never use mutable objects as dictionary or hashset keys.
