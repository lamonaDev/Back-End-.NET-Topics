# SortedList, Dictionary, and HashSet in C#

## Table of Contents
1. [Understanding Dictionary](#understanding-dictionary)
2. [Understanding HashSet](#understanding-hashset)
3. [Understanding SortedList and SortedDictionary](#understanding-sortedlist-and-sorteddictionary)
4. [Hashing and Collision Resolution](#hashing-and-collision-resolution)
5. [Collection Comparison](#collection-comparison)
6. [Real-World Use Cases](#real-world-use-cases)
7. [Memory and Performance](#memory-and-performance)
8. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Dictionary

### What is Dictionary?

`Dictionary<TKey, TValue>` is a **hash table** implementation providing O(1) average time complexity for insert, delete, and lookup operations. It maps keys to values using a hash function.

### Hash Table Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    DICTIONARY (HASH TABLE) STRUCTURE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Memory Layout:                                                │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Buckets (Array of Entry references)                      │    │
│   │  ┌─────┬─────┬─────┬─────┬─────┬─────┐                  │    │
│   │  │  0  │  1  │  2  │  3  │  4  │ ... │                  │    │
│   │  │  │  │  │  │  │  │     │     │      │                  │    │
│   │  └──┼──┴──┼──┴──┼──┘     └─────┘      │                  │    │
│   │     │     │     │                        │                  │    │
│   │     ↓     ↓     ↓                        │                  │    │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ │                  │    │
│   │  │ Entry   │ │ Entry   │ │ Entry   │ │                  │    │
│   │  │ Key: A  │ │ Key: B  │ │ Key: C  │ │                  │    │
│   │  │ Val: 1  │ │ Val: 2  │ │ Val: 3  │ │                  │    │
│   │  │ hash:A  │→│ hash:B  │→│ hash:C  │→│ null             │    │
│   │  └─────────┘ └─────────┘ └─────────┘ │                  │    │
│   │     Bucket 1      Bucket 2    Bucket 3 │                  │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Hash Function: hashCode % bucketCount = bucketIndex              │
│                                                                  │
│   Lookup Process:                                                │
│   1. Compute hash of key                                          │
│   2. Find bucket index                                            │
│   3. Traverse chain to find matching key                        │
│   4. Return value                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Dictionary Usage

```csharp
public class DictionaryDemo
{
    public static void Main()
    {
        // Create dictionary
        Dictionary<string, int> ages = new();
        
        // Add (O(1) average)
        ages["Alice"] = 30;
        ages.Add("Bob", 25);
        ages.TryAdd("Charlie", 35); // Returns false if key exists
        
        // Lookup (O(1) average)
        int aliceAge = ages["Alice"]; // 30
        bool hasBob = ages.ContainsKey("Bob"); // true
        bool hasDiana = ages.TryGetValue("Diana", out int dianaAge); // false
        
        // Update
        ages["Alice"] = 31; // Overwrite
        
        // Remove (O(1) average)
        ages.Remove("Bob");
        
        // Iterate
        foreach (var (name, age) in ages)
        {
            Console.WriteLine($"{name}: {age}");
        }
        
        // Count
        Console.WriteLine($"Count: {ages.Count}");
    }
}
```

---

## Understanding HashSet

### What is HashSet?

`HashSet<T>` is a **set data structure** based on a hash table. It stores unique elements with O(1) lookup, add, and remove operations.

### HashSet Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    HASHSET STRUCTURE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Similar to Dictionary but only stores keys:                   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Buckets                                                │    │
│   │  ┌─────┬─────┬─────┬─────┬─────┐                      │    │
│   │  │  0  │  1  │  2  │  3  │  4  │                      │    │
│   │  │  │  │  │  │  │  │     │     │                      │    │
│   │  └──┼──┴──┼──┴──┼──┘     └─────┘                      │    │
│   │     │     │     │                                      │    │
│   │     ↓     ↓     ↓                                      │    │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐                  │    │
│   │  │ "Alice" │→│ "Bob"   │→│ "Charlie"│→│ null         │    │
│   │  │ hash:A  │ │ hash:B  │ │ hash:C  │                  │    │
│   │  └─────────┘ └─────────┘ └─────────┘                  │    │
│   │                                                      │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Operations:                                                    │
│   ├─ Add(item): O(1) - add if not present                       │
│   ├─ Remove(item): O(1) - remove if present                     │
│   ├─ Contains(item): O(1) - check presence                      │
│   └─ UnionWith, IntersectWith, ExceptWith: set operations       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### HashSet Usage

```csharp
public class HashSetDemo
{
    public static void Main()
    {
        // Create HashSet
        HashSet<string> uniqueNames = new();
        
        // Add (O(1), returns false if already exists)
        uniqueNames.Add("Alice"); // true
        uniqueNames.Add("Bob");   // true
        uniqueNames.Add("Alice"); // false - duplicate
        
        // Contains (O(1))
        bool hasAlice = uniqueNames.Contains("Alice"); // true
        
        // Set Operations
        var set1 = new HashSet<int> { 1, 2, 3, 4 };
        var set2 = new HashSet<int> { 3, 4, 5, 6 };
        
        set1.UnionWith(set2);        // { 1, 2, 3, 4, 5, 6 }
        set1.IntersectWith(set2);    // { 3, 4 }
        set1.ExceptWith(set2);       // { 1, 2 }
        set1.SymmetricExceptWith(set2); // { 1, 2, 5, 6 }
        
        // Check subset/superset
        bool isSubset = set1.IsSubsetOf(set2);
        bool isSuperset = set1.IsSupersetOf(set2);
    }
}
```

---

## Understanding SortedList and SortedDictionary

### What are Sorted Collections?

`SortedList<TKey, TValue>` and `SortedDictionary<TKey, TValue>` maintain keys in sorted order. They use different internal structures:
- **SortedList**: Two arrays (keys and values), memory efficient
- **SortedDictionary**: Red-Black tree, better for frequent updates

### SortedList Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    SORTEDLIST STRUCTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Two parallel arrays kept in sorted order:                     │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Keys Array           │  Values Array                   │    │
│   │  ┌─────────────────┐  │  ┌─────────────────┐            │    │
│   │  │ "Alice"         │  │  │ 30              │            │    │
│   │  │ "Bob"           │  │  │ 25              │            │    │
│   │  │ "Charlie"       │  │  │ 35              │            │    │
│   │  │ "Diana"         │  │  │ 28              │            │    │
│   │  └─────────────────┘  │  └─────────────────┘            │    │
│   │        ↑                  ↑                              │    │
│   │        Sorted by key      Aligned values               │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Lookup: Binary Search O(log n)                                │
│   Insert: Find position, shift elements O(n)                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### SortedDictionary Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    SORTEDDICTIONARY (RED-BLACK TREE)             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Self-balancing binary search tree:                            │
│                                                                  │
│              ┌─────────┐                                          │
│              │ "Charlie"│ ← Root                                  │
│              │   35    │                                          │
│              └────┬────┘                                          │
│            ┌──────┴──────┐                                      │
│            ▼             ▼                                      │
│       ┌─────────┐   ┌─────────┐                                │
│       │ "Bob"   │   │ "Diana" │                                │
│       │   25    │   │   28    │                                │
│       └────┬────┘   └─────────┘                                │
│              ▼                                                   │
│       ┌─────────┐                                              │
│       │ "Alice" │                                              │
│       │   30    │                                              │
│       └─────────┘                                              │
│                                                                  │
│   Lookup: O(log n) - tree traversal                             │
│   Insert: O(log n) - insert + rebalance                         │
│                                                                  │
│   Red-Black Tree ensures balance:                                │
│   ├─ All paths have same number of black nodes                  │
│   ├─ No two red nodes adjacent                                  │
│   └─ Guarantees O(log n) height                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Comparison: SortedList vs SortedDictionary

```csharp
public class SortedCollectionsDemo
{
    public static void Compare()
    {
        // SortedList - better for mostly static data
        SortedList<string, int> sortedList = new()
        {
            { "Alice", 30 },
            { "Bob", 25 },
            { "Charlie", 35 }
        };
        
        // SortedDictionary - better for frequently modified data
        SortedDictionary<string, int> sortedDict = new()
        {
            { "Alice", 30 },
            { "Bob", 25 },
            { "Charlie", 35 }
        };
        
        // Fast index access in SortedList
        var firstKey = sortedList.Keys[0]; // "Alice"
        var firstValue = sortedList.Values[0]; // 30
        
        // Enumeration is always sorted
        foreach (var (key, value) in sortedList)
        {
            Console.WriteLine($"{key}: {value}");
        }
    }
}
```

---

## Hashing and Collision Resolution

### Hash Function

```
┌─────────────────────────────────────────────────────────────────┐
│                    HASH FUNCTION FLOW                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Key: "Alice"                                                   │
│      │                                                           │
│      ▼                                                           │
│   GetHashCode()                                                  │
│      │                                                           │
│      ▼                                                           │
│   Hash Code: 0x3A8F2C1D (example)                              │
│      │                                                           │
│      ▼                                                           │
│   bucketIndex = hashCode % bucketCount                           │
│      │                                                           │
│      ▼                                                           │
│   Index: 5 ← Store/Retrieve from bucket 5                       │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Collision: When two keys hash to same bucket           │    │
│   │                                                         │    │
│   │  "Alice" → hash: 0x3A8F2C1D → index: 5                   │    │
│   │  "Aaron" → hash: 0x3A8F3000 → index: 5                   │    │
│   │                                                         │    │
│   │  Resolution: Chaining (linked list in bucket)           │    │
│   │  Bucket 5: ["Alice"] → ["Aaron"] → null                 │    │
│   │                                                         │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Custom GetHashCode

```csharp
public class Person : IEquatable<Person>
{
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
    public int Age { get; set; }
    
    public override int GetHashCode()
    {
        // Combine hash codes of fields
        return HashCode.Combine(FirstName, LastName, Age);
    }
    
    public override bool Equals(object? obj) => 
        obj is Person p && Equals(p);
    
    public bool Equals(Person? other)
    {
        if (other is null) return false;
        return FirstName == other.FirstName &&
               LastName == other.LastName &&
               Age == other.Age;
    }
}
```

---

## Collection Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    COLLECTION COMPARISON                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Dictionary<TKey, TValue>:                                      │
│   ├─ Unordered                                                   │
│   ├─ O(1) lookup, insert, delete                                 │
│   ├─ Fastest general-purpose key-value storage                   │
│   └─ Use for: Caches, indexes, lookup tables                     │
│                                                                  │
│   HashSet<T>:                                                    │
│   ├─ Unordered unique collection                                 │
│   ├─ O(1) add, remove, contains                                  │
│   ├─ Set operations (union, intersect, etc.)                     │
│   └─ Use for: Deduplication, membership testing                  │
│                                                                  │
│   SortedList<TKey, TValue>:                                      │
│   ├─ Sorted by key                                               │
│   ├─ O(log n) lookup, O(n) insert                                │
│   ├─ Memory efficient, index access                              │
│   └─ Use for: Sorted data, mostly read-only                     │
│                                                                  │
│   SortedDictionary<TKey, TValue>:                                  │
│   ├─ Sorted by key                                               │
│   ├─ O(log n) lookup, insert, delete                             │
│   ├─ Better for frequent modifications                           │
│   └─ Use for: Dynamic sorted collections                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Use Cases

### Dictionary: In-Memory Cache

```csharp
public class InMemoryCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, CacheEntry<TValue>> _cache;
    private readonly TimeSpan _expiration;

    public InMemoryCache(TimeSpan expiration)
    {
        _expiration = expiration;
        _cache = new Dictionary<TKey, CacheEntry<TValue>>();
    }

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        if (_cache.TryGetValue(key, out var entry) && entry.Expiration > DateTime.UtcNow)
        {
            return entry.Value;
        }

        var value = factory(key);
        _cache[key] = new CacheEntry<TValue>
        {
            Value = value,
            Expiration = DateTime.UtcNow + _expiration
        };

        return value;
    }

    public void Remove(TKey key) => _cache.Remove(key);

    private class CacheEntry<T>
    {
        public required T Value { get; set; }
        public DateTime Expiration { get; set; }
    }
}
```

### HashSet: Duplicate Detection

```csharp
public class DuplicateDetector
{
    public static List<int> FindDuplicates(IEnumerable<int> numbers)
    {
        var seen = new HashSet<int>();
        var duplicates = new HashSet<int>();

        foreach (var number in numbers)
        {
            if (!seen.Add(number))
            {
                duplicates.Add(number);
            }
        }

        return duplicates.ToList();
    }

    public static bool HasDuplicates(IEnumerable<int> numbers)
    {
        var set = new HashSet<int>();
        foreach (var number in numbers)
        {
            if (!set.Add(number))
                return true;
        }
        return false;
    }
}
```

### SortedDictionary: Event Timeline

```csharp
public class EventTimeline
{
    private readonly SortedDictionary<DateTime, List<Event>> _events;

    public EventTimeline()
    {
        _events = new SortedDictionary<DateTime, List<Event>>();
    }

    public void AddEvent(Event evt)
    {
        if (!_events.TryGetValue(evt.Timestamp, out var list))
        {
            list = new List<Event>();
            _events[evt.Timestamp] = list;
        }
        list.Add(evt);
    }

    public IEnumerable<Event> GetEventsBetween(DateTime start, DateTime end)
    {
        foreach (var (timestamp, events) in _events)
        {
            if (timestamp < start) continue;
            if (timestamp > end) break;
            
            foreach (var evt in events)
                yield return evt;
        }
    }
}
```

---

## Memory and Performance

### Memory Layout Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY LAYOUT                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Dictionary:                                                   │
│   ├─ Array of Entry structs (key, value, hash)                │
│   ├─ Overhead: ~20% unused capacity                            │
│   ├─ Rehashing occurs when load factor > 0.72                  │
│   └─ Doubles capacity on resize                              │
│                                                                  │
│   HashSet:                                                      │
│   ├─ Similar to Dictionary (only stores hash + key)            │
│   ├─ More compact than Dictionary<T, bool>                    │
│   └─ Same resizing behavior                                     │
│                                                                  │
│   SortedList:                                                   │
│   ├─ Two arrays: keys[] and values[]                           │
│   ├─ Compact - no unused slots                                │
│   ├─ Resizes when full                                        │
│   └─ Insertion requires shifting                              │
│                                                                  │
│   SortedDictionary:                                             │
│   ├─ Tree nodes with references                               │
│   ├─ More overhead per entry (color, parent, child refs)     │
│   ├─ No resizing - grows by allocation                       │
│   └─ Better locality of reference during range queries       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Use Dictionary for fast lookups
var lookup = new Dictionary<int, string>();

// 2. Use HashSet for uniqueness
var unique = new HashSet<int>();

// 3. Pre-size when count is known
var dict = new Dictionary<int, string>(1000);
var set = new HashSet<int>(1000);

// 4. Implement GetHashCode properly
public override int GetHashCode() => HashCode.Combine(Field1, Field2);

// 5. Use TryGetValue for safe access
if (dict.TryGetValue(key, out var value)) { }

// 6. Use TryAdd for conditional insertion
if (dict.TryAdd(key, value)) { /* added */ }

// 7. Use proper Equals with GetHashCode
public override bool Equals(object? obj) => obj is MyType other && Id == other.Id;
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Mutable objects as keys
var dict = new Dictionary<List<int>, string>(); // ❌ Dangerous!

// PITFALL 2: Modifying key while in dictionary
var key = new { Id = 1 };
dict[key] = "value";
key.Id = 2; // ❌ Breaks hash table!

// PITFALL 3: Not overriding Equals and GetHashCode
public class Person { public int Id { get; set; } }
// Two Person objects with same Id won't be equal!

// PITFALL 4: Using reference equality for value types
var dict = new Dictionary<int[], string>();
dict[new[] { 1, 2 }] = "A";
dict[new[] { 1, 2 }]; // ❌ Different array, not found!

// PITFALL 5: Not handling missing keys
dict[key]; // ❌ KeyNotFoundException
// Use TryGetValue or ContainsKey

// PITFALL 6: Poor hash distribution
public override int GetHashCode() => 1; // ❌ Everything collides!
```

---

## Interview Questions

**Q: What's the difference between Dictionary and Hashtable?**> Dictionary is generic (type-safe, no boxing), Hashtable is non-generic. Dictionary is generally preferred in modern C#. Dictionary also maintains insertion order since .NET Core.

**Q: What's the time complexity of Dictionary operations?**> Average case: O(1) for insert, delete, and lookup. Worst case: O(n) if all keys collide. This assumes a good hash function with uniform distribution.

**Q: When should you use SortedDictionary over Dictionary?**> Use SortedDictionary when you need keys maintained in sorted order or need range queries. Dictionary is faster (O(1) vs O(log n)) but doesn't maintain order.

**Q: What's the difference between SortedList and SortedDictionary?**> SortedList uses two arrays (memory efficient, O(n) insert). SortedDictionary uses a Red-Black tree (O(log n) insert, better for frequent modifications). SortedList also supports index access.

**Q: Why must you override GetHashCode when overriding Equals?**> Dictionary and HashSet rely on hash codes to locate items. If two objects are equal (Equals returns true), they must have the same hash code. Violating this breaks hash-based collections.

**Q: What's a hash collision and how is it resolved?**> A collision occurs when two different keys produce the same hash code. .NET resolves collisions using chaining (each bucket can contain a linked list of entries with the same hash).
