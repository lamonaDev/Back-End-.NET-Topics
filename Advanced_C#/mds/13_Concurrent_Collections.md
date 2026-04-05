# Concurrent Collections: ConcurrentDictionary, ConcurrentBag, ConcurrentQueue

## 🧠 What Are They?

The `System.Collections.Concurrent` namespace provides **thread-safe collection types** that can be used by multiple threads simultaneously — without the need for external `lock` statements.

Regular collections like `Dictionary<K,V>`, `List<T>`, and `Queue<T>` are **NOT thread-safe**. Reading and writing them from multiple threads causes data corruption and crashes.

---

## 🌍 Real-World Analogy

| Collection | Analogy |
|---|---|
| **ConcurrentDictionary** | A shared office whiteboard with **labeled sticky notes** — multiple people can add/remove/update notes at the same time without chaos. |
| **ConcurrentBag** | A **shared grab bag** of items — anyone can toss in or grab an item, order doesn't matter. |
| **ConcurrentQueue** | A **checkout line** at a store — first person in, first person served. Thread-safe FIFO. |

---

## ⚙️ Why Regular Collections Break Under Threads

```csharp
var dict = new Dictionary<int, string>(); // NOT thread-safe

// Thread 1                         Thread 2
dict["key"] = "A";                  dict["key"] = "B";  
// Both try to resize/rehash simultaneously → CRASH or corruption
```

The concurrent versions solve this with **fine-grained internal locking** and **lock-free algorithms** — much more efficient than wrapping everything in a single `lock {}`.

---

## 🔵 1. ConcurrentDictionary\<TKey, TValue\>

A thread-safe key-value store. The most commonly used concurrent collection.

```csharp
using System.Collections.Concurrent;
using System.Threading.Tasks;

var scores = new ConcurrentDictionary<string, int>();

// Thread-safe add or update
scores.TryAdd("Alice", 100);
scores.TryAdd("Bob", 150);

// AddOrUpdate: if key exists, run update function; else add new value
scores.AddOrUpdate(
    key: "Alice",
    addValue: 10,
    updateValueFactory: (key, oldValue) => oldValue + 50
);
// Alice: 100 → 150

// GetOrAdd: get existing or add new (atomic)
int carlScore = scores.GetOrAdd("Carl", 200);

// TryGetValue, TryRemove, TryUpdate
if (scores.TryGetValue("Bob", out int bobScore))
    Console.WriteLine($"Bob's score: {bobScore}");

scores.TryRemove("Carl", out _);
```

### Concurrent word frequency counter (real-world example):

```csharp
var wordCount = new ConcurrentDictionary<string, int>(StringComparer.OrdinalIgnoreCase);

string[] words = { "apple", "banana", "apple", "cherry", "banana", "apple" };

Parallel.ForEach(words, word =>
{
    wordCount.AddOrUpdate(word, 1, (_, count) => count + 1);
});

foreach (var (word, count) in wordCount)
    Console.WriteLine($"{word}: {count}");
// apple: 3, banana: 2, cherry: 1
```

---

## 🟡 2. ConcurrentBag\<T\>

An **unordered** thread-safe collection. Optimized for scenarios where each thread both adds and removes from the same bag — like a work-stealing pool.

```csharp
var results = new ConcurrentBag<string>();

// Multiple threads adding results concurrently
Parallel.For(0, 5, i =>
{
    string result = $"Result from thread {i}";
    results.Add(result);
});

// Order is NOT guaranteed
while (results.TryTake(out string? item))
{
    Console.WriteLine(item);
}
```

### ⚙️ Memory Model — Thread-Local Storage

```
Thread 1       Thread 2       Thread 3
  [bag1]         [bag2]         [bag3]    ← each thread has local list
     │              │              │
     └──────────────┴──────────────┘
               ConcurrentBag
          (steals from others when empty)
```

> `ConcurrentBag` uses **thread-local storage** for each thread's items. A thread adds to and takes from its local list — no contention. If a thread's local list is empty, it **steals** from another thread's list. This makes it exceptionally fast for producer=consumer same-thread scenarios.

---

## 🟢 3. ConcurrentQueue\<T\>

A **FIFO** (First-In, First-Out) thread-safe queue. Perfect for producer-consumer pipelines.

```csharp
var queue = new ConcurrentQueue<string>();

// Producer threads enqueue
Task producer1 = Task.Run(() =>
{
    for (int i = 0; i < 5; i++)
    {
        queue.Enqueue($"P1-Item-{i}");
    }
});

Task producer2 = Task.Run(() =>
{
    for (int i = 0; i < 5; i++)
    {
        queue.Enqueue($"P2-Item-{i}");
    }
});

await Task.WhenAll(producer1, producer2);

// Consumer dequeues
while (queue.TryDequeue(out string? item))
{
    Console.WriteLine($"Processing: {item}");
}
```

### Peek without removing:

```csharp
if (queue.TryPeek(out string? next))
    Console.WriteLine($"Next item (without removing): {next}");
```

---

## 📊 Full Comparison Table

| Feature | `ConcurrentDictionary` | `ConcurrentBag` | `ConcurrentQueue` |
|---|---|---|---|
| Structure | Key-Value | Unordered set | FIFO queue |
| Thread-safe | ✅ | ✅ | ✅ |
| Order preserved | N/A (keyed) | ❌ | ✅ FIFO |
| Best for | Shared state / caches | Parallel result collection | Producer-consumer |
| Async-native | ❌ | ❌ | ❌ (use Channel for async) |
| Performance | Fine-grained locks | Thread-local + stealing | Lock-free segments |

---

## 🆚 ConcurrentDictionary vs Dictionary (with lock)

```csharp
// Option A: Dictionary + manual lock
private readonly Dictionary<string, int> _dict = new();
private readonly object _lock = new();

void Set(string key, int val)
{
    lock (_lock) { _dict[key] = val; } // entire dictionary locked
}

// Option B: ConcurrentDictionary — fine-grained locking internally
private readonly ConcurrentDictionary<string, int> _dict = new();

void Set(string key, int val)
{
    _dict[key] = val; // Only the affected bucket is locked
}
```

> `ConcurrentDictionary` uses **striped locking** (locks per bucket group), allowing concurrent reads and writes to different keys simultaneously.

---

## 📌 Summary

> Use `ConcurrentDictionary` for shared key-value state. Use `ConcurrentBag` when order doesn't matter and threads both produce and consume. Use `ConcurrentQueue` for ordered producer-consumer scenarios. For async pipelines, prefer `System.Threading.Channels` over `ConcurrentQueue`.
