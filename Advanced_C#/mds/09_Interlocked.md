# Interlocked Operations in C#

## Table of Contents
1. [Understanding Interlocked](#understanding-interlocked)
2. [Atomic Operations](#atomic-operations)
3. [Compare-and-Swap (CAS)](#compare-and-swap-cas)
4. [Memory Barriers](#memory-barriers)
5. [Lock-Free Data Structures](#lock-free-data-structures)
6. [Real-World Use Cases](#real-world-use-cases)
7. [Performance Comparison](#performance-comparison)
8. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Interlocked

### What is Interlocked?

`Interlocked` is a static class providing **atomic operations** on variables shared between threads. These operations are hardware-supported and complete in a single, uninterruptible step - no locking required.

### Why Use Interlocked?

```csharp
// ❌ PROBLEM: Simple operations are NOT atomic
public class UnsafeCounter
{
    private int _value = 0;
    
    public void Increment()
    {
        _value++; // Actually 3 operations: READ → ADD → WRITE
    }
}

// Race condition:
// Thread 1: Read _value=0
// Thread 2: Read _value=0 (before Thread 1 writes)
// Thread 1: Write _value=1
// Thread 2: Write _value=1 (LOST UPDATE!)

// ✅ SOLUTION: Atomic operation
public class SafeCounter
{
    private int _value = 0;
    
    public void Increment()
    {
        Interlocked.Increment(ref _value); // Single atomic operation
    }
}
```

### Hardware Support

```
┌─────────────────────────────────────────────────────────────────┐
│                    ATOMIC OPERATION HARDWARE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Modern CPUs provide atomic instructions:                      │
│                                                                  │
│   x86/x64: LOCK prefix                                          │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  lock inc [address]      ; Atomic increment               │    │
│   │  lock xchg [addr], reg  ; Atomic exchange                 │    │
│   │  lock cmpxchg [addr], reg ; Atomic compare-and-swap       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   ARM: LDREX/STREX (Load-Exclusive/Store-Exclusive)             │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  ldrex r0, [address]     ; Load with exclusive access     │    │
│   │  add r0, r0, #1         ; Modify                         │    │
│   │  strex r1, r0, [address] ; Store conditional             │    │
│   │  cmp r1, #0             ; Check success                  │    │
│   │  bne retry               ; Retry if failed               │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   These operations are:                                          │
│   ├─ Indivisible (no thread can see intermediate state)         │
│   ├─ Thread-safe (no locks needed)                             │
│   └─ Very fast (~10-20 nanoseconds)                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Atomic Operations

### Increment and Decrement

```csharp
public class AtomicCounter
{
    private long _value = 0;
    
    // Atomic increment - returns NEW value
    public long Increment()
    {
        return Interlocked.Increment(ref _value);
    }
    
    // Atomic decrement - returns NEW value
    public long Decrement()
    {
        return Interlocked.Decrement(ref _value);
    }
    
    // Add arbitrary value - returns NEW value
    public long Add(long amount)
    {
        return Interlocked.Add(ref _value, amount);
    }
    
    // Read current value atomically
    public long Read()
    {
        return Interlocked.Read(ref _value);
    }
}

// Usage
var counter = new AtomicCounter();
Parallel.For(0, 1000000, _ => counter.Increment());
Console.WriteLine(counter.Read()); // Exactly 1000000
```

### Exchange Operations

```csharp
public class AtomicReference<T> where T : class
{
    private T? _value;
    
    // Get current value and set new value
    public T? Exchange(T? newValue)
    {
        return Interlocked.Exchange(ref _value, newValue);
    }
    
    // Get current value
    public T? Read()
    {
        return _value;
    }
}

// Usage - Thread-safe reference swapping
var reference = new AtomicReference<string>();
reference.Exchange("Initial");

// In multiple threads:
var oldValue = reference.Exchange("New");
// oldValue is guaranteed to be the previous value
// Even if multiple threads exchange simultaneously
```

### 64-bit Operations on 32-bit Systems

```csharp
// On 32-bit systems, 64-bit reads/writes are NOT atomic!
// Interlocked ensures atomicity

long sharedValue = 0;

// ❌ Not atomic on 32-bit systems
// long current = sharedValue;

// ✅ Always atomic
long current = Interlocked.Read(ref sharedValue);

// ❌ Not atomic
// sharedValue = 123456789012;

// ✅ Atomic
Interlocked.Exchange(ref sharedValue, 123456789012);
```

---

## Compare-and-Swap (CAS)

### What is CAS?

Compare-and-Swap is the fundamental building block of lock-free algorithms. It atomically:
1. Compares a value to an expected value
2. If equal, swaps in a new value
3. Returns the original value (whether swap occurred or not)

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAS OPERATION FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Initial: value = 10                                          │
│                                                                  │
│   CAS(ref value, expected: 10, newValue: 20)                    │
│        │                                                         │
│   ┌────┴────────────────────────────────────────────────────┐    │
│   │  ATOMIC CHECK: Is value == expected (10)?              │    │
│   │                                                         │    │
│   │     YES → Set value = newValue (20)                   │    │
│   │           Return original (10)                        │    │
│   │                                                         │    │
│   │     NO  → Return current value (not 10)               │    │
│   │           No change made                              │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Result: value = 20 (success)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Interlocked.CompareExchange

```csharp
public class LockFreeStack<T>
{
    private class Node
    {
        public T Value = default!;
        public Node? Next;
    }
    
    private Node? _head;
    
    public void Push(T value)
    {
        var newNode = new Node { Value = value };
        
        do
        {
            newNode.Next = _head; // Point to current head
        }
        // Try to update head to newNode
        // Only succeeds if _head hasn't changed since we read it
        while (Interlocked.CompareExchange(ref _head, newNode, newNode.Next) != newNode.Next);
    }
    
    public bool TryPop(out T? value)
    {
        Node? currentHead;
        
        do
        {
            currentHead = _head;
            if (currentHead == null)
            {
                value = default;
                return false; // Empty stack
            }
        }
        // Try to update head to head.Next
        // Only succeeds if head hasn't changed
        while (Interlocked.CompareExchange(ref _head, currentHead.Next, currentHead) != currentHead);
        
        value = currentHead.Value;
        return true;
    }
}
```

### Retry Pattern with CAS

```csharp
public class LockFreeAdder
{
    private long _sum = 0;
    
    public void Add(long value)
    {
        long current;
        long newSum;
        
        do
        {
            current = Interlocked.Read(ref _sum);          // Read current
            newSum = current + value;                       // Calculate new
        }
        while (Interlocked.CompareExchange(ref _sum, newSum, current) != current);
        // Retry if another thread changed _sum
    }
}
```

---

## Memory Barriers

### Why Memory Barriers Matter

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY REORDERING PROBLEM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without memory barriers, the compiler/CPU may reorder:        │
│                                                                  │
│   Code written:                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  _data = 42;                                            │    │
│   │  _ready = true;                                         │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Possible execution order:                                      │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  _ready = true;      ← OBSERVED FIRST!                  │    │
│   │  _data = 42;                                             │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Another thread might see: _ready=true but _data=0!          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Interlocked.MemoryBarrier

```csharp
public class SafePublisher
{
    private int _data = 0;
    private bool _ready = false;
    
    public void Publish(int value)
    {
        _data = value;
        Interlocked.MemoryBarrier(); // Ensure _data is written before _ready
        _ready = true;
    }
    
    public bool TryRead(out int value)
    {
        if (_ready)
        {
            Interlocked.MemoryBarrier(); // Ensure we see _data after _ready
            value = _data;
            return true;
        }
        value = 0;
        return false;
    }
}

// Note: Modern code usually uses volatile or lock instead
public class ModernPublisher
{
    private volatile int _data = 0;  // Implies memory barriers
    private volatile bool _ready = false;
    
    public void Publish(int value)
    {
        _data = value;
        _ready = true; // volatile ensures ordering
    }
}
```

---

## Lock-Free Data Structures

### Lock-Free Stack

```csharp
public class LockFreeStack<T>
{
    private class Node
    {
        public T Value = default!;
        public Node? Next;
    }
    
    private volatile Node? _head;
    private int _count = 0;
    
    public int Count => Interlocked.CompareExchange(ref _count, 0, 0);
    
    public void Push(T value)
    {
        var newNode = new Node { Value = value };
        
        do
        {
            newNode.Next = _head;
        }
        while (Interlocked.CompareExchange(ref _head, newNode, newNode.Next) != newNode.Next);
        
        Interlocked.Increment(ref _count);
    }
    
    public bool TryPop(out T? value)
    {
        Node? currentHead;
        
        do
        {
            currentHead = _head;
            if (currentHead == null)
            {
                value = default;
                return false;
            }
        }
        while (Interlocked.CompareExchange(ref _head, currentHead.Next, currentHead) != currentHead);
        
        Interlocked.Decrement(ref _count);
        value = currentHead.Value;
        return true;
    }
    
    public bool IsEmpty => _head == null;
}
```

### Lock-Free Queue (Michael-Scott Queue)

```csharp
public class LockFreeQueue<T>
{
    private class Node
    {
        public T Value = default!;
        public volatile Node? Next;
    }
    
    private volatile Node _dummy;
    private volatile Node _tail;
    private volatile Node _head;
    
    public LockFreeQueue()
    {
        _dummy = new Node();
        _head = _dummy;
        _tail = _dummy;
    }
    
    public void Enqueue(T value)
    {
        var newNode = new Node { Value = value };
        Node currentTail;
        
        while (true)
        {
            currentTail = _tail;
            var next = currentTail.Next;
            
            // Help complete pending enqueue
            if (currentTail == _tail)
            {
                if (next == null)
                {
                    // Try to link new node
                    if (Interlocked.CompareExchange(ref currentTail.Next!, newNode, null) == null)
                    {
                        // Successfully linked, try to advance tail
                        Interlocked.CompareExchange(ref _tail, newNode, currentTail);
                        return;
                    }
                }
                else
                {
                    // Tail is behind, help advance it
                    Interlocked.CompareExchange(ref _tail, next, currentTail);
                }
            }
        }
    }
    
    public bool TryDequeue(out T? value)
    {
        while (true)
        {
            var currentHead = _head;
            var currentTail = _tail;
            var next = currentHead.Next;
            
            if (currentHead == _head)
            {
                if (currentHead == currentTail)
                {
                    if (next == null)
                    {
                        value = default;
                        return false; // Empty
                    }
                    // Tail is behind, help advance it
                    Interlocked.CompareExchange(ref _tail, next, currentTail);
                }
                else
                {
                    value = next!.Value;
                    if (Interlocked.CompareExchange(ref _head, next, currentHead) == currentHead)
                    {
                        return true;
                    }
                }
            }
        }
    }
}
```

---

## Real-World Use Cases

### Use Case 1: High-Performance Counters

```csharp
public class PerformanceMetrics
{
    private long _requestCount = 0;
    private long _errorCount = 0;
    private long _totalLatency = 0;
    
    public void RecordRequest(TimeSpan latency, bool isError)
    {
        Interlocked.Increment(ref _requestCount);
        Interlocked.Add(ref _totalLatency, latency.Ticks);
        
        if (isError)
        {
            Interlocked.Increment(ref _errorCount);
        }
    }
    
    public MetricsSnapshot GetSnapshot()
    {
        return new MetricsSnapshot
        {
            RequestCount = Interlocked.Read(ref _requestCount),
            ErrorCount = Interlocked.Read(ref _errorCount),
            TotalLatency = Interlocked.Read(ref _totalLatency),
            AverageLatency = TimeSpan.FromTicks(
                Interlocked.Read(ref _requestCount) > 0 
                    ? Interlocked.Read(ref _totalLatency) / Interlocked.Read(ref _requestCount)
                    : 0)
        };
    }
}

public class MetricsSnapshot
{
    public long RequestCount { get; set; }
    public long ErrorCount { get; set; }
    public long TotalLatency { get; set; }
    public TimeSpan AverageLatency { get; set; }
}
```

### Use Case 2: Lazy Initialization

```csharp
public class SingletonService
{
    private static Service? _instance;
    private static int _initialized = 0;
    
    public static Service Instance
    {
        get
        {
            if (_initialized == 0)
            {
                var service = new Service();
                
                // Try to set (only first thread succeeds)
                if (Interlocked.CompareExchange(ref _instance, service, null) == null)
                {
                    Interlocked.Exchange(ref _initialized, 1);
                }
            }
            
            return _instance!;
        }
    }
}

// Modern approach using Lazy<T>
public class ModernSingleton
{
    private static readonly Lazy<Service> _instance = new(() => new Service());
    public static Service Instance => _instance.Value;
}
```

### Use Case 3: Connection ID Generation

```csharp
public class ConnectionIdGenerator
{
    private long _currentId = 0;
    
    public long GenerateId()
    {
        return Interlocked.Increment(ref _currentId);
    }
    
    // Thread-safe batch generation
    public (long Start, long End) GenerateBatch(int count)
    {
        var start = Interlocked.Add(ref _currentId, count) - count + 1;
        return (start, start + count - 1);
    }
}
```

---

## Performance Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE COMPARISON                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Operation          │  Latency          │  Thread-Safe?       │
│   ─────────────────────────────────────────────────────────────│
│   Regular int++      │  ~1 ns            │  ❌ No               │
│   lock { count++ }   │  ~50-100 ns       │  ✅ Yes              │
│   Interlocked.Inc    │  ~10-20 ns        │  ✅ Yes              │
│   Interlocked.CAS    │  ~10-20 ns        │  ✅ Yes              │
│                                                                  │
│   Key insight: Interlocked is ~5-10x faster than lock          │
│                                                                  │
│   But limitations:                                               │
│   ├─ Only basic operations (inc, dec, add, exchange, CAS)     │
│   ├─ No complex logic                                           │
│   ├─ ABA problem in lock-free structures                       │
│   └─ Retry loops may waste CPU under heavy contention            │
│                                                                  │
│   When to use what:                                             │
│   ├─ Simple counters → Interlocked                            │
│   ├─ Complex operations → lock                                  │
│   ├─ Multiple related variables → lock                         │
│   └─ High-frequency simple ops → Interlocked                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Use Interlocked for simple counters
private long _counter;
public void Increment() => Interlocked.Increment(ref _counter);

// 2. Read with Interlocked.CompareExchange for consistency
long current = Interlocked.CompareExchange(ref _counter, 0, 0);

// 3. CAS retry loops
long current, newValue;
do
{
    current = Interlocked.Read(ref _sum);
    newValue = current + amount;
}
while (Interlocked.CompareExchange(ref _sum, newValue, current) != current);

// 4. Know when NOT to use Interlocked
// Complex logic needs lock:
lock (_sync)
{
    // Multiple operations on multiple variables
    _balance -= amount;
    _transactionCount++;
    _lastTransactionTime = DateTime.UtcNow;
}
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Thinking Interlocked helps with complex operations
// ❌ This is NOT atomic
Interlocked.Increment(ref _counter);
Console.WriteLine(_counter); // May see different value!

// ✅ Read atomically too
var current = Interlocked.Increment(ref _counter);
Console.WriteLine(current); // Correct

// PITFALL 2: ABA problem in lock-free structures
// Thread 1: Read head=A
// Thread 2: Pop A, Push A (same address, different content)
// Thread 1: CAS succeeds, but stack changed!

// PITFALL 3: Starvation under contention
// CAS may retry many times under heavy load
// Eventually consider using lock instead

// PITFALL 4: Memory visibility without barriers
// Interlocked operations provide full fences
// But mixing with regular reads/writes is dangerous

// PITFALL 5: Not handling compare exchange failure
// Must retry if CompareExchange fails
var oldValue = Interlocked.CompareExchange(ref _value, newValue, expected);
if (oldValue != expected)
{
    // Someone else changed it - handle this case!
}
```

---

## Interview Questions

**Q: What is Interlocked and when should you use it?**
> Interlocked provides atomic operations on shared variables. Use it for simple thread-safe operations like counters when you need better performance than lock (5-10x faster).

**Q: How does Interlocked.CompareExchange work?**
> It atomically compares a value to an expected value and, if equal, replaces it with a new value. It returns the original value. Used for lock-free algorithms and CAS retry loops.

**Q: What's the ABA problem?**
> In lock-free structures, if a value changes from A→B→A, a CAS operation might succeed incorrectly thinking nothing changed. The address is same but content may differ.

**Q: When should you prefer lock over Interlocked?**
> Use lock when: (1) Multiple operations need to be atomic together, (2) Complex logic involved, (3) Coordinating multiple variables, (4) Under heavy contention where CAS retries waste CPU.

**Q: Does Interlocked provide memory barriers?**
> Yes, Interlocked operations act as full memory barriers (fence), ensuring proper ordering of memory operations across threads.

**Q: Can Interlocked be used with reference types?**
> Yes, Interlocked.Exchange and CompareExchange work with reference types. However, remember that you're swapping references, not modifying the referenced object atomically.
