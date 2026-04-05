# List\<T\> — Complete Deep Dive

## 🧠 What Is List\<T\>?

`List<T>` is a **generic, dynamic-size collection** backed by an internal array. It's the most commonly used collection in C# — the go-to replacement for fixed arrays when you need items to grow, shrink, or be searched and sorted.

---

## 🌍 Real-World Analogy

An **array** is a parking lot with painted spots — 10 spots, that's it, forever. A **List** is an expandable garage that adds floors automatically. Use arrays when you know exactly how many items you'll ever have. Use List for everything else.

---

## 📦 Arrays vs Collections

| Aspect | Array | `List<T>` |
|---|---|---|
| Size | Fixed at declaration | Dynamic — grows/shrinks |
| Index access | ✅ O(1) | ✅ O(1) |
| Add element | ❌ Must recreate | ✅ `Add()` — O(1) amortized |
| Remove element | ❌ Must recreate | ✅ `Remove()` — O(n) |
| Memory | Exact — no waste | May over-allocate (capacity buffer) |
| Rich API | ❌ Very limited | ✅ Sort, Find, AddRange, etc. |

```csharp
// Array — fixed size forever
int[] numbers = new int[5];
numbers[5] = 30; // ❌ IndexOutOfRangeException!
// Need more space? Manual painful work:
int[] bigger = new int[10];
Array.Copy(numbers, bigger, numbers.Length);

// List — dynamic, rich API
List<int> numbers = new();
numbers.Add(10);
numbers.Add(20);
numbers.Add(30); // ✅ grows automatically
numbers.Remove(20); // ✅ shrinks too
numbers.Insert(0, 5); // ✅ insert anywhere
```

---

## ❌ Generic vs Non-Generic Collections

Always use `System.Collections.Generic` — never `System.Collections` (legacy).

```csharp
// ❌ Non-generic ArrayList — type-unsafe, causes boxing
ArrayList list = new();
list.Add(10);
list.Add("hello"); // ⚠️ No compile error — mixed types!
int n = (int)list[1]; // 💥 InvalidCastException at runtime
// int 10 was BOXED to object on Add() — heap pressure!

// ✅ Generic List<T> — compile-time safety, no boxing
List<int> list = new();
list.Add(10);
list.Add(20); // ✅ Only int allowed
// list.Add("hello"); // ❌ Compile-time error — caught immediately
int n = list[1]; // ✅ No cast needed
```

---

## 🔗 Interface Hierarchy

Understanding which interface to use as a parameter type makes your code more flexible:

```
IEnumerable<T>
  └── foreach / GetEnumerator()
      │
      ▼
ICollection<T>
  └── + Count, Add, Remove, Contains, Clear
      │
      ▼
IList<T>
  └── + this[index], Insert, RemoveAt, IndexOf
```

> **Practical rule:** Accept the **narrowest** interface your method actually needs. A method that only iterates should accept `IEnumerable<T>` — it'll work with List, array, HashSet, Queue, or anything.

---

## 💻 Creating a List\<T\> — 6 Ways

```csharp
List<int> a = new();             // empty — no heap alloc until first Add()
List<int> b = new() { 1, 2, 3 }; // collection initializer (C# 3.0+)
List<int> c = [1, 2, 3];         // collection expression (C# 12)
List<int> d = new(100);          // pre-allocated capacity — ZERO resizes
List<int> e = someArray.ToList(); // from an existing array
List<int> f = new(existingList);  // copy constructor — copies all elements
```

| Constructor | Allocation | When to use |
|---|---|---|
| `new()` | None until first `Add()` | Default, size unknown |
| `new(capacity)` | Pre-allocates `capacity` slots | Know the count upfront — eliminates resizes |
| `new(IEnumerable)` | Reads `.Count` first if available — exact allocation | Copying a known collection |

---

## 💻 Basic Operations

```csharp
List<string> names = new() { "Ahmed", "Sara", "Ali" };

// ── Adding ─────────────────────────────────────────────────────────
names.Add("Omar");              // O(1) amortized — appends to end
names.AddRange(["X", "Y"]);    // O(k) — adds k elements, at most one resize
names.Insert(0, "First");       // O(n) ⚠️ shifts EVERY element right!

// ── Removing ───────────────────────────────────────────────────────
names.Remove("Sara");           // O(n) — scan to find + shift remaining left
names.RemoveAt(0);              // O(n) — no scan, but still shifts
names.RemoveAll(x => x.Length > 3); // O(n) — single-pass, most efficient bulk removal

// ── Accessing ──────────────────────────────────────────────────────
string first = names[0];        // O(1) — direct array index
string last  = names[^1];       // O(1) — C# 8 index-from-end syntax
int count    = names.Count;     // O(1) — reads _size field directly

// ── Membership ─────────────────────────────────────────────────────
bool has  = names.Contains("Ali"); // O(n) — linear scan
int  idx  = names.IndexOf("Ali");  // O(n) — returns index or -1
```

### Complexity Quick Reference

| Operation | Complexity | Reason |
|---|---|---|
| `Add()` | O(1) amortized | Writes to next slot; occasional O(n) resize |
| `Insert(0, x)` | O(n) ⚠️ | Shifts every element right |
| `Remove(x)` | O(n) | Scan + shift |
| `RemoveAt(i)` | O(n) | Shift remaining |
| `this[i]` | O(1) | Direct array index |
| `Count` | O(1) | Reads `_size` field |
| `Contains(x)` | O(n) | Linear scan |

> ⚠️ `Insert(0, x)` on a list of 50,000 items causes 50,000 moves. If you need frequent insertions at arbitrary positions, consider `LinkedList<T>`.

---

## 💻 Searching

```csharp
List<int> nums = [5, 2, 8, 1, 9, 3, 7, 4, 6];

// Find — first element matching predicate, or default(T)
int first = nums.Find(n => n > 5);         // O(n) → 8 (stops at first match)

// FindAll — new List with ALL matches
List<int> big = nums.FindAll(n => n > 5);  // O(n) → [8, 9, 7, 6]

// FindLast — scans from the END
int last = nums.FindLast(n => n > 5);      // O(n) → 6

// Exists — short-circuits on first match, returns bool
bool any = nums.Exists(n => n > 8);        // O(n) best-case O(1) → true

// IndexOf / LastIndexOf
int idx = nums.IndexOf(8);                 // O(n) → 2
int lastIdx = nums.LastIndexOf(3);         // O(n) — scans from end

// BinarySearch — ONLY works on a SORTED list!
nums.Sort();
int pos = nums.BinarySearch(7);            // O(log n) → 6
int neg = nums.BinarySearch(10);           // negative = not found; ~neg = insertion point
```

> 💡 `Exists()` short-circuits — stops at the first match. Use it over `FindAll().Count > 0`.
> ✅ `BinarySearch` requires `Sort()` first; gives O(log n) lookup.

---

## 💻 Sorting

```csharp
List<int> nums = [5, 2, 8, 1, 9, 3];

// Default sort — uses IComparable<T> on the element type
nums.Sort();                             // O(n log n) → [1, 2, 3, 5, 8, 9]

// Reverse — flips in-place (NOT sort descending)
nums.Reverse();                          // O(n) → [9, 8, 5, 3, 2, 1]

// Sort with lambda — descending
nums.Sort((a, b) => b.CompareTo(a));     // O(n log n) descending

// Sort a partial range — indices 1..3 only
nums.Sort(1, 3, null);                   // O(k log k) partial sort
```

> ⚠️ `Sort()` modifies the original. To preserve: `var sorted = new List<int>(original); sorted.Sort();`

---

## 💻 Bulk Operations

```csharp
List<int> nums = [1, 2, 3, 4, 5, 6];

// AddRange — at most ONE resize
nums.AddRange([10, 11, 12]);              // O(k)

// RemoveAll — single-pass two-pointer algorithm (far better than loop!)
nums.RemoveAll(x => x < 3);              // O(n) → [3, 4, 5, 6, 10, 11, 12]
int removed = nums.RemoveAll(x => x > 10); // returns count removed

// ForEach — iterate with Action
nums.ForEach(x => Console.Write($"{x} ")); // O(n)

// Clear — removes all elements; Capacity stays the same
nums.Clear();
Console.WriteLine(nums.Count);    // 0
Console.WriteLine(nums.Capacity); // unchanged (e.g., 8)
```

```csharp
// ❌ O(n²) — Remove() in a loop
foreach (var x in nums.ToList())
    if (x < 3) nums.Remove(x);   // each Remove is O(n) → total O(n²)

// ✅ O(n) — RemoveAll
nums.RemoveAll(x => x < 3);       // single-pass two-pointer, always O(n)
```

---

## 💻 Conversions

```csharp
List<int> nums = [1, 2, 3, 4, 5];

int[]       arr   = nums.ToArray();                         // O(n) — new array
List<int>   copy  = arr.ToList();                           // O(n) — copy
List<string> strs = nums.ConvertAll(x => x.ToString());    // O(n) → ["1","2","3","4","5"]
List<double> half = nums.ConvertAll(x => x / 2.0);         // O(n) → [0.5,1.0,...]
List<int>   slice = nums.GetRange(1, 3);                    // O(k) → [2,3,4] (shallow copy)

// Deduplicate via HashSet
List<int> dupes  = [1, 1, 2, 2, 3];
List<int> unique = dupes.ToHashSet().ToList();              // O(n) → [1, 2, 3]
```

> 💡 `ConvertAll` pre-allocates the result list to the correct size immediately — slightly more efficient than LINQ's `Select().ToList()` for `List<T>` sources.

---

## ⚙️ How List\<T\> Works Internally

### The Three Core Fields

```
private T[]  _items;    // actual backing array — Capacity = _items.Length
private int  _size;     // ← what .Count returns (elements actually added)
private int  _version;  // mutation counter — foreach throws if changed mid-loop
```

### Capacity Doubling Strategy

```
new List<int>()         → _items = s_emptyArray (no allocation!)
                          Count: 0  |  Capacity: 0

After first Add(1)      →  [1][ ][ ][ ]
                          Count: 1  |  Capacity: 4  ← DefaultCapacity = 4

After Add(1..4) — FULL  →  [1][2][3][4]
                          Count: 4  |  Capacity: 4  ← FULL → next Add triggers resize!

After Add(5)            →  [1][2][3][4][5][ ][ ][ ]
                          Count: 5  |  Capacity: 8  ← doubled! O(n) copy happens once
```

> 💡 **Why is `Add()` O(1) amortized?**
> Most calls just write to the next slot — O(1). Resizes happen at counts 4, 8, 16, 32... so across n total adds, total work = 2n. Divided over n calls = **O(1) per add on average**.

### `.Count` vs `.Count()` vs `.Length`

```csharp
List<int> list = [1, 2, 3];

list.Count    // ✅ O(1) — reads _size directly. ALWAYS use this.
list.Capacity // ✅ O(1) — reads _items.Length (internal array size)
// list.Length → ❌ COMPILE ERROR — List<T> has no .Length

list.Count()  // ⚠️ LINQ method — O(1) on List (detects ICollection),
              //    but O(n) on plain IEnumerable! Prefer the PROPERTY.
```

---

## 💻 Real-World: List\<Student\>

```csharp
public class Student : IComparable<Student>
{
    public int Id { get; init; }
    public string? Name { get; set; }
    public int Age { get; set; }

    public Student(int id, string name, int age) => (Id, Name, Age) = (id, name, age);
    public int CompareTo(Student? other) => Id.CompareTo(other?.Id); // sorts by Id
}

List<Student> students =
[
    new(1, "Ahmed",   20),
    new(2, "Sara",    22),
    new(3, "Mohamed", 21),
    new(4, "Laila",   19),
    new(5, "Bola",    23),
];

// ── Searching ──────────────────────────────────────────────────
Student? found   = students.Find(s => s.Age > 20);           // Sara (first match)
List<Student> older = students.FindAll(s => s.Age > 20);      // Sara, Mohamed, Bola
bool hasMinors   = students.Exists(s => s.Age < 18);          // false

// ── Sorting ────────────────────────────────────────────────────
students.Sort();                                               // by Id (uses CompareTo)
students.Sort((a, b) => a.Age.CompareTo(b.Age));              // by Age ascending
students.Sort((a, b) => b.Age.CompareTo(a.Age));              // by Age descending
students.Sort((a, b) => string.Compare(a.Name, b.Name,        // alphabetical by Name
    StringComparison.Ordinal));

// ── Bulk Ops ───────────────────────────────────────────────────
students.RemoveAll(s => s.Age < 21);                          // O(n) single-pass
students.ForEach(s => Console.WriteLine(s.Name));             // O(n)
List<string> names = students.ConvertAll(s => s.Name!);       // O(n) — pre-allocated
students.Add(new(6, "Omar", 24));                             // O(1) amortized
bool allAdults = !students.Exists(s => s.Age < 18);           // O(n)
```

---

## 📌 Summary

> `List<T>` is a **dynamic array** with a doubling resize strategy. Use `.Count` (not `.Count()` or `.Length`). Prefer `RemoveAll()` over `Remove()` in a loop. Pre-allocate with `new(capacity)` when the count is known. Use `BinarySearch()` after `Sort()` for O(log n) lookup. For reference types, `Clear()` nulls each slot to allow GC collection.
