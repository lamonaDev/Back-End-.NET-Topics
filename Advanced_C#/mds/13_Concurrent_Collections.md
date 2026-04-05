# Concurrent Collections in C#

## Table of Contents
1. [Understanding Concurrent Collections](#understanding-concurrent-collections)
2. [ConcurrentDictionary](#concurrentdictionary)
3. [ConcurrentQueue](#concurrentqueue)
4. [ConcurrentBag](#concurrentbag)
5. [ConcurrentStack](#concurrentstack)
6. [BlockingCollection](#blockingcollection)
7. [Memory and Performance](#memory-and-performance)
8. [Real-World Use Cases](#real-world-use-cases)
9. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Concurrent Collections

### What are Concurrent Collections?

Concurrent collections are thread-safe data structures in `System.Collections.Concurrent` designed for multi-threaded scenarios. Unlike regular collections wrapped with `lock`, they use **lock-free or fine-grained locking** mechanisms for better scalability under high concurrency.

### Why Not Just Use lock?

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOCK VS CONCURRENT COLLECTIONS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Dictionary + lock:                                            │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  lock (dict) {                                           │    │
│   │      // Only ONE thread can operate at a time            │    │
│   │      dict.Add(key, value);                              │    │
│   │  }                                                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│   Single bottleneck - all threads serialized                     │
│                                                                  │
│   ConcurrentDictionary:                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Buckets: [0] [1] [2] [3] [4] ... [31]                   │    │
│   │            │   │   │   │   │                             │    │
│   │           🔒  🔒  🔒  🔒  🔒  ← Fine-grained locks         │    │
│   │            │   │   │   │   │                             │    │
│   │  Thread A→│   │← Thread B                                 │    │
│   │  Thread C→│   │← Thread D                                 │    │
│   └─────────────────────────────────────────────────────────┘    │
│   Multiple threads can operate on different buckets             │
│                                                                  │
│   Lock-free operations (Interlocked) on hot paths              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Concurrent Collections Overview

| Collection | Thread-Safe | Lock-Free | Best For |
|------------|-------------|-----------|----------|
| ConcurrentDictionary | ✅ Yes | Partial | Key-value storage |
| ConcurrentQueue | ✅ Yes | Yes | FIFO work distribution |
| ConcurrentBag | ✅ Yes | Yes | Unordered collection |
| ConcurrentStack | ✅ Yes | Yes | LIFO scenarios |
| BlockingCollection | ✅ Yes | N/A (wrapper) | Producer-consumer with blocking |

---

## ConcurrentDictionary

### What is ConcurrentDictionary?

`ConcurrentDictionary<TKey, TValue>` is a thread-safe key-value store using **fine-grained locking** on hash buckets and **lock-free reads** on hot paths.

### Memory Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONCURRENTDICTIONARY STRUCTURE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Buckets (array of volatile buckets):                          │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Bucket 0 │ Bucket 1 │ Bucket 2 │ ... │ Bucket N         │    │
│   │    🔒       🔒          🔒                🔒             │    │
│   └─────────────────────────────────────────────────────────┘    │
│        │                                                          │
│   ┌────┴────────┐                                                │
│   │  Bucket 0   │  ← Lock per bucket                            │
│   │  ┌────────┐ │                                                │
│   │  │ Node 1 │─┼─>┌────────┐                                   │
│   │  │ K1, V1 │ │   │ Node 2 │─> ...                            │
│   │  └────────┘ │   │ K2, V2 │                                   │
│   │             │   └────────┘                                   │
│   └─────────────┘                                                │
│                                                                  │
│   Key Hash → Bucket Index → Acquire Lock → Modify Chain           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Methods

```csharp
// Thread-safe operations - no manual locking needed!
var cache = new ConcurrentDictionary<int, string>();

// Add or update
cache.TryAdd(1, "value");           // Add only if key doesn't exist
cache.TryUpdate(1, "new", "old");   // Update only if current == "old"

// Get or add (atomic)
var value = cache.GetOrAdd(1, key => ComputeValue(key));

// Add or update (atomic)
var value = cache.AddOrUpdate(1, 
    addValueFactory: key => ComputeValue(key),
    updateValueFactory: (key, old) => UpdateValue(key, old));

// Remove
cache.TryRemove(1, out var removed);

// Read (lock-free on hot paths)
if (cache.TryGetValue(1, out var val)) { }
```

### Complete Example

```csharp
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, CacheItem<TValue>> _cache;
    private readonly TimeSpan _expiration;

    public ThreadSafeCache(TimeSpan expiration)
    {
        _expiration = expiration;
        _cache = new ConcurrentDictionary<TKey, CacheItem<TValue>>();
    }

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> valueFactory)
    {
        return _cache.GetOrAdd(key, k => new CacheItem<TValue>
        {
            Value = valueFactory(k),
            Timestamp = DateTime.UtcNow
        }).Value;
    }

    public bool TryUpdate(TKey key, TValue newValue)
    {
        if (_cache.TryGetValue(key, out var existing))
        {
            return _cache.TryUpdate(key, 
                new CacheItem<TValue> { Value = newValue, Timestamp = DateTime.UtcNow },
                existing);
        }
        return false;
    }

    public void RemoveExpired()
    {
        var cutoff = DateTime.UtcNow - _expiration;
        foreach (var key in _cache.Where(kvp => kvp.Value.Timestamp < cutoff).Select(kvp => kvp.Key))
        {
            _cache.TryRemove(key, out _);
        }
    }
}

public class CacheItem<T>
{
    public required T Value { get; set; }
    public DateTime Timestamp { get; set; }
}
```

---

## ConcurrentQueue

### What is ConcurrentQueue?

`ConcurrentQueue<T>` implements a **lock-free FIFO queue** using a singly-linked list of segments. Multiple threads can enqueue and dequeue concurrently.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONCURRENTQUEUE STRUCTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Segments (fixed size arrays):                                 │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Segment 1      Segment 2       Segment 3                 │    │
│   │  ┌────────┐   ┌────────┐    ┌────────┐                  │    │
│   │  │ Slot 0 │──>│ Slot 0 │───>│ Slot 0 │                  │    │
│   │  │ Slot 1 │   │ Slot 1 │    │ Slot 1 │                  │    │
│   │  │ Slot 2 │   │ Slot 2 │    │ Slot 2 │                  │    │
│   │  │  ...   │   │  ...   │    │  ...   │                  │    │
│   │  └────────┘   └────────┘    └────────┘                  │    │
│   │                                                    ↓   │    │
│   │   Head ────────────────────────────────>          Tail  │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Enqueue: Add to tail using Interlocked                         │
│   Dequeue: Remove from head using Interlocked                     │
│   Both are lock-free operations!                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Usage Example

```csharp
public class TaskQueue
{
    private readonly ConcurrentQueue<WorkItem> _queue = new();
    private int _count = 0;

    public void Enqueue(WorkItem item)
    {
        _queue.Enqueue(item);
        Interlocked.Increment(ref _count);
    }

    public bool TryDequeue(out WorkItem? item)
    {
        if (_queue.TryDequeue(out item))
        {
            Interlocked.Decrement(ref _count);
            return true;
        }
        return false;
    }

    public int Count => Interlocked.CompareExchange(ref _count, 0, 0);

    public bool IsEmpty => _queue.IsEmpty;
}
```

---

## ConcurrentBag

### What is ConcurrentBag?

`ConcurrentBag<T>` is an **unordered collection** optimized for scenarios where the same thread is both producer and consumer. Each thread gets its own local queue for lock-free operations.

### Thread-Local Storage Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONCURRENTBAG STRUCTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Global List (Thread-local lists):                             │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Thread A List    Thread B List    Thread C List         │    │
│   │  ┌───────────┐   ┌───────────┐   ┌───────────┐        │    │
│   │  │ Item 1    │   │ Item 4    │   │ Item 7    │        │    │
│   │  │ Item 2    │   │ Item 5    │   │ Item 8    │        │    │
│   │  │ Item 3    │   │ Item 6    │   │ Item 9    │        │    │
│   │  └───────────┘   └───────────┘   └───────────┘        │    │
│   │       🔒                 🔒                 🔒           │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Take from current thread: O(1) lock-free                     │
│   Take from other threads: Requires lock, O(n)                   │
│                                                                  │
│   Best for: Object pooling, work stealing                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Object Pool Example

```csharp
public class ObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects;
    private int _count = 0;

    public ObjectPool(int initialCapacity = 100)
    {
        _objects = new ConcurrentBag<T>();
        
        for (int i = 0; i < initialCapacity; i++)
        {
            _objects.Add(new T());
            Interlocked.Increment(ref _count);
        }
    }

    public T Get()
    {
        if (_objects.TryTake(out var obj))
        {
            Interlocked.Decrement(ref _count);
            return obj;
        }
        return new T();
    }

    public void Return(T obj)
    {
        _objects.Add(obj);
        Interlocked.Increment(ref _count);
    }

    public int Count => Interlocked.CompareExchange(ref _count, 0, 0);
}
```

---

## ConcurrentStack

### What is ConcurrentStack?

`ConcurrentStack<T>` is a **lock-free LIFO stack** using a singly-linked list with atomic operations.

### Usage Example

```csharp
public class WorkStealingScheduler
{
    private readonly ConcurrentStack<WorkItem> _localWork = new();
    private readonly ConcurrentBag<ConcurrentStack<WorkItem>> _allStacks = new();

    public void Push(WorkItem item)
    {
        _localWork.Push(item);
    }

    public bool TryPop(out WorkItem? item)
    {
        // Try local stack first (LIFO - cache friendly)
        if (_localWork.TryPop(out item))
            return true;

        // Try stealing from other threads (work stealing)
        foreach (var otherStack in _allStacks)
        {
            if (otherStack != _localWork && otherStack.TryPop(out item))
                return true;
        }

        item = null;
        return false;
    }
}
```

---

## BlockingCollection

### What is BlockingCollection?

`BlockingCollection<T>` wraps any `IProducerConsumerCollection<T>` with **blocking and bounding** capabilities. It provides synchronous waiting for producers and consumers.

### BlockingCollection Features

```csharp
// Wrap a ConcurrentQueue with blocking
var blockingQueue = new BlockingCollection<WorkItem>(
    new ConcurrentQueue<WorkItem>(),
    boundedCapacity: 100);

// Producer - blocks when full
blockingQueue.Add(workItem);           // Blocks if full
blockingQueue.TryAdd(item, timeout);  // Non-blocking with timeout

// Consumer - blocks when empty
var item = blockingQueue.Take();      // Blocks until available
blockingQueue.TryTake(out item, timeout);

// Complete adding
blockingQueue.CompleteAdding();

// Foreach consumes until CompleteAdding
foreach (var item in blockingQueue.GetConsumingEnumerable())
{
    Process(item);
}
```

### Producer-Consumer Example

```csharp
public class ProducerConsumerPipeline<T>
{
    private readonly BlockingCollection<T> _collection;

    public ProducerConsumerPipeline(int capacity)
    {
        _collection = new BlockingCollection<T>(
            new ConcurrentQueue<T>(),
            capacity);
    }

    public void Produce(IEnumerable<T> items)
    {
        foreach (var item in items)
        {
            _collection.Add(item); // Blocks if full
        }
        _collection.CompleteAdding();
    }

    public void Consume(Action<T> processor, int consumerCount)
    {
        Parallel.For(0, consumerCount, _ =>
        {
            foreach (var item in _collection.GetConsumingEnumerable())
            {
                processor(item);
            }
        });
    }
}
```

---

## Memory and Performance

### Performance Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONCURRENT COLLECTIONS PERFORMANCE              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Single-threaded operations (ns):                              │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Dictionary (no lock)  │  ~15-30 ns                      │    │
│   │  Dictionary + lock       │  ~100-200 ns                    │    │
│   │  ConcurrentDictionary  │  ~50-100 ns                     │    │
│   │  ConcurrentQueue       │  ~40-80 ns                      │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Multi-threaded scalability:                                     │
│   ├─ ConcurrentDictionary: Excellent (fine-grained locks)        │
│   ├─ ConcurrentQueue: Excellent (lock-free)                      │
│   ├─ ConcurrentBag: Good (thread-local optimization)           │
│   └─ Dictionary + lock: Poor (single bottleneck)                 │
│                                                                  │
│   Memory overhead:                                               │
│   ├─ ConcurrentDictionary: ~2x regular Dictionary               │
│   ├─ ConcurrentQueue: ~1.5x Queue                            │
│   └─ ConcurrentBag: Variable (per-thread lists)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Use Cases

### Use Case 1: In-Memory Cache

```csharp
public class ConcurrentCache<TKey, TValue> where TKey : notnull
{
    private readonly ConcurrentDictionary<TKey, CacheEntry<TValue>> _cache;
    private readonly Timer _cleanupTimer;

    public ConcurrentCache()
    {
        _cache = new ConcurrentDictionary<TKey, CacheEntry<TValue>>();
        _cleanupTimer = new Timer(_ => Cleanup(), null, TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1));
    }

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory)
    {
        var entry = _cache.GetOrAdd(key, k => new CacheEntry<TValue>
        {
            Value = factory(k),
            Expiration = DateTime.UtcNow.AddMinutes(10)
        });

        entry.LastAccess = DateTime.UtcNow;
        return entry.Value;
    }

    public bool TryRemove(TKey key) => _cache.TryRemove(key, out _);

    private void Cleanup()
    {
        var now = DateTime.UtcNow;
        foreach (var key in _cache.Where(kvp => kvp.Value.Expiration < now).Select(kvp => kvp.Key))
        {
            _cache.TryRemove(key, out _);
        }
    }
}

public class CacheEntry<T>
{
    public required T Value { get; set; }
    public DateTime Expiration { get; set; }
    public DateTime LastAccess { get; set; }
}
```

### Use Case 2: Work Queue

```csharp
public class ThreadPoolWorkQueue
{
    private readonly ConcurrentQueue<WorkItem> _workQueue = new();
    private readonly SemaphoreSlim _semaphore = new(0);

    public void Enqueue(WorkItem work)
    {
        _workQueue.Enqueue(work);
        _semaphore.Release();
    }

    public async Task<WorkItem> DequeueAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        
        if (_workQueue.TryDequeue(out var work))
        {
            return work;
        }
        
        throw new InvalidOperationException("Queue inconsistency");
    }

    public int Count => _workQueue.Count;
}
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Use GetOrAdd/AddOrUpdate for atomic operations
var result = cache.GetOrAdd(key, k => ExpensiveCompute(k));

// 2. Don't lock around concurrent collections
lock (concurrentDict) { } // ❌ Unnecessary!

// 3. Handle Try methods return values
if (queue.TryDequeue(out var item))
{
    // Use item
}

// 4. Consider Count performance
// ConcurrentDictionary.Count is O(n) - cache if needed

// 5. Choose right collection for the pattern
// Dictionary for random access
// Queue for FIFO
// Bag for unordered, object pooling
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Assuming enumeration is snapshot
foreach (var item in concurrentDict) // May see modifications!

// PITFALL 2: Locking on concurrent collection
lock (concurrentDict) { concurrentDict.Add(...); } // Redundant

// PITFALL 3: Expecting Try methods to always succeed
while (queue.TryDequeue(out var item)) { } // May miss items!

// PITFALL 4: Using Count in tight loops
for (int i = 0; i < queue.Count; i++) // O(n) each iteration!

// PITFALL 5: Not handling null return from Try methods
var value = cache.TryGetValue(key, out var val) ? val : default; // ✅
```

---

## Interview Questions

**Q: What's the difference between Dictionary with lock and ConcurrentDictionary?**> Dictionary with lock serializes ALL operations on a single lock. ConcurrentDictionary uses fine-grained locking on hash buckets and lock-free reads, allowing multiple threads to operate concurrently on different buckets.

**Q: When should you use ConcurrentQueue vs BlockingCollection?**> Use ConcurrentQueue when you need async/await support or non-blocking operations. Use BlockingCollection when you need blocking semantics (synchronous wait) or bounded capacity with blocking producers.

**Q: Why is ConcurrentBag optimized for same-thread operations?**> ConcurrentBag maintains thread-local lists. When a thread adds and removes from its own list, it's lock-free (fast). Stealing from other threads requires locking (slower). Best for object pooling where threads typically return what they took.

**Q: Are concurrent collection enumerations thread-safe?**> Yes, but they don't provide snapshot semantics. The enumeration may reflect modifications made during enumeration, and the contents may change while enumerating.

**Q: What's the performance difference between lock(Dict) and ConcurrentDictionary?**> Single-threaded: Dictionary+lock is ~2-3x slower than plain Dictionary. Multi-threaded: ConcurrentDictionary scales much better. With 8+ threads, ConcurrentDictionary can be 10-100x faster than locked Dictionary.
