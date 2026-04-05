# Monitor, Mutex, Semaphore, and SemaphoreSlim in C#

## Table of Contents
1. [Understanding Synchronization Primitives](#understanding-synchronization-primitives)
2. [Monitor (lock)](#monitor-lock)
3. [Mutex](#mutex)
4. [Semaphore](#semaphore)
5. [SemaphoreSlim](#semaphoreslim)
6. [Comparison and Selection Guide](#comparison-and-selection-guide)
7. [Memory and Performance](#memory-and-performance)
8. [Real-World Use Cases](#real-world-use-cases)
9. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Synchronization Primitives

### What Are Synchronization Primitives?

Synchronization primitives are mechanisms that control access to shared resources in multi-threaded environments. They prevent **race conditions** where multiple threads simultaneously modify shared data, leading to inconsistent or corrupted state.

### The Problem: Race Conditions

```csharp
// ❌ PROBLEM: Race condition without synchronization
public class UnsafeCounter
{
    private int _count = 0;
    
    public void Increment()
    {
        // This looks atomic, but it's NOT!
        // Actually: read _count → add 1 → write _count
        _count++;
    }
}

// Thread 1 reads _count=0
// Thread 2 reads _count=0 (before Thread 1 writes)
// Thread 1 writes _count=1
// Thread 2 writes _count=1 (LOST UPDATE!)
// Expected: 2, Actual: 1
```

### Memory Visibility Problem

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREAD MEMORY MODEL                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without Synchronization:                                      │
│   ┌──────────────┐           ┌──────────────┐                  │
│   │  Thread A    │           │  Thread B    │                  │
│   │  ─────────── │           │  ─────────── │                  │
│   │  CPU Cache   │           │  CPU Cache   │                  │
│   │  _count = 10 │           │  _count = 5  │ ← Stale value!   │
│   └──────────────┘           └──────────────┘                  │
│           │                           │                         │
│           └───────────┬───────────────┘                         │
│                       │                                         │
│               ┌───────▼────────┐                               │
│               │ Main Memory    │                               │
│               │ _count = ?     │ ← Inconsistent!                │
│               └────────────────┘                               │
│                                                                  │
│   Problem: Each thread may have cached, stale values!          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Monitor (lock)

### What is Monitor?

`Monitor` (commonly accessed via the `lock` statement) is C#'s most common synchronization mechanism. It provides **mutual exclusion** - only one thread can execute the locked code block at a time.

### How lock Works

```csharp
// What you write:
lock (_syncObject)
{
    // Critical section - only one thread at a time
}

// What the compiler generates:
Monitor.Enter(_syncObject);
try
{
    // Critical section
}
finally
{
    Monitor.Exit(_syncObject);
}
```

### Basic Code Example

```csharp
public class SafeCounter
{
    private int _count = 0;
    private readonly object _lock = new();
    
    public void Increment()
    {
        lock (_lock)
        {
            _count++;
            Console.WriteLine($"Count: {_count} [Thread: {Thread.CurrentThread.ManagedThreadId}]");
        }
    }
    
    public int GetCount()
    {
        lock (_lock)
        {
            return _count;
        }
    }
}

// Usage
var counter = new SafeCounter();
var threads = new Thread[10];

for (int i = 0; i < 10; i++)
{
    threads[i] = new Thread(() =>
    {
        for (int j = 0; j < 100; j++)
        {
            counter.Increment();
        }
    });
    threads[i].Start();
}

foreach (var t in threads) t.Join();
Console.WriteLine($"Final count: {counter.GetCount()}"); // Exactly 1000
```

### Monitor Memory Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITOR LOCK MECHANISM                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Object Header (sync block):                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  _syncObject                                            │    │
│   │  ├─ Object Header (4/8 bytes)                         │    │
│   │  │   └─ Sync Block Index → Points to sync block        │    │
│   │  │                                                     │    │
│   │  ├─ Sync Block Table Entry                             │    │
│   │  │   ├─ Owner Thread ID                               │    │
│   │  │   ├─ Recursion Count (for reentrant locks)         │    │
│   │  │   ├─ Wait Queue (threads waiting)                  │    │
│   │  │   └─ Event Handle (OS primitive)                   │    │
│   │  │                                                     │    │
│   │  └─ Object Data                                         │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Lock Acquisition Process:                                      │
│   1. Try CAS (Compare-And-Swap) on sync block                   │
│   2. If fail, spin briefly (SpinLock optimization)              │
│   3. If still fail, block thread via OS primitive               │
│   4. Add to wait queue, release CPU                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Monitor.Wait and Monitor.Pulse

```csharp
public class ProducerConsumerQueue<T>
{
    private readonly Queue<T> _queue = new();
    private readonly object _lock = new();
    private bool _isCompleted = false;

    public void Enqueue(T item)
    {
        lock (_lock)
        {
            _queue.Enqueue(item);
            // Signal waiting consumers
            Monitor.Pulse(_lock);
        }
    }

    public bool TryDequeue(out T? item, TimeSpan timeout)
    {
        lock (_lock)
        {
            // Wait while queue is empty
            while (_queue.Count == 0 && !_isCompleted)
            {
                if (!Monitor.Wait(_lock, timeout))
                {
                    item = default;
                    return false; // Timeout
                }
            }

            if (_queue.Count > 0)
            {
                item = _queue.Dequeue();
                return true;
            }

            item = default;
            return false;
        }
    }
}
```

---

## Mutex

### What is Mutex?

A `Mutex` (Mutual Exclusion) is similar to Monitor but can work **across process boundaries**. It's an OS-level synchronization primitive that can be named and shared between multiple processes.

### Basic Usage

```csharp
// Local mutex (same process only)
using (var mutex = new Mutex())
{
    mutex.WaitOne(); // Acquire lock
    try
    {
        // Critical section
    }
    finally
    {
        mutex.ReleaseMutex();
    }
}

// Named mutex (cross-process)
using (var mutex = new Mutex(false, "Global\MyAppMutex"))
{
    // Other processes can use the same name
    mutex.WaitOne();
    try
    {
        // Critical section - only one process at a time!
    }
    finally
    {
        mutex.ReleaseMutex();
    }
}
```

### Cross-Process Singleton Pattern

```csharp
public class SingleInstanceApp
{
    private static Mutex? _mutex;
    
    public static bool TryAcquireSingleton()
    {
        const string mutexName = "Global\MyCompany.MyApp.SingleInstance";
        
        _mutex = new Mutex(true, mutexName, out bool createdNew);
        
        if (!createdNew)
        {
            // Another instance is running
            _mutex.Dispose();
            _mutex = null;
            return false;
        }
        
        return true;
    }
    
    public static void ReleaseSingleton()
    {
        _mutex?.ReleaseMutex();
        _mutex?.Dispose();
    }
}

// Usage in Program.cs
if (!SingleInstanceApp.TryAcquireSingleton())
{
    MessageBox.Show("Application is already running!");
    return;
}

// Run application
try
{
    Application.Run(new MainForm());
}
finally
{
    SingleInstanceApp.ReleaseSingleton();
}
```

### Mutex vs Monitor Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    MUTEX VS MONITOR                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Monitor (lock):                                                │
│   ├─ Process-local only                                         │
│   ├─ Faster (~50-100 nanoseconds)                               │
│   ├─ Automatic release on exception                             │
│   ├─ Cannot be named                                            │
│   └─ Language-integrated (lock keyword)                          │
│                                                                  │
│   Mutex:                                                         │
│   ├─ Cross-process capable                                      │
│   ├─ Slower (~microseconds, kernel transition)                  │
│   ├─ Must manually ReleaseMutex()                               │
│   ├─ Can be named                                               │
│   └─ OS-level primitive                                         │
│                                                                  │
│   Memory Cost:                                                   │
│   ├─ Monitor: ~0 (uses object header)                           │
│   ├─ Mutex: Kernel object (~1000+ bytes)                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Semaphore

### What is Semaphore?

A `Semaphore` controls access to a **pool of resources**. Unlike Monitor/Mutex (which allow only 1 thread), Semaphore can allow a **specified number** of threads to access a resource simultaneously.

### Semaphore Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEMAPHORE CONCEPTS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Initial Count = 3 (3 slots available)                         │
│   Maximum Count = 3 (can never exceed 3)                        │
│                                                                  │
│   Thread A ──Acquire──┐                                         │
│   Thread B ──Acquire──┼──> Semaphore [3 slots]                │
│   Thread C ──Acquire──┘                                         │
│                                                                  │
│   After 3 acquires:                                              │
│   ┌────────────────────────────────────────┐                    │
│   │  Slots: [A] [B] [C]                   │                    │
│   │  Available: 0                         │                    │
│   └────────────────────────────────────────┘                    │
│                                                                  │
│   Thread D tries Acquire ──> BLOCKED (waiting)                  │
│                                                                  │
│   When Thread A releases:                                        │
│   Slots: [ ] [B] [C] ──> Thread D unblocks, acquires            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Code Example

```csharp
public class ConnectionPool
{
    private readonly Semaphore _semaphore;
    private readonly ConcurrentQueue<Connection> _availableConnections;
    private readonly int _maxConnections;

    public ConnectionPool(int maxConnections)
    {
        _maxConnections = maxConnections;
        _semaphore = new Semaphore(maxConnections, maxConnections);
        _availableConnections = new ConcurrentQueue<Connection>();
        
        // Pre-populate connections
        for (int i = 0; i < maxConnections; i++)
        {
            _availableConnections.Enqueue(new Connection { Id = i });
        }
    }

    public Connection AcquireConnection(TimeSpan timeout)
    {
        // Wait for available slot
        if (!_semaphore.WaitOne(timeout))
        {
            throw new TimeoutException("Could not acquire connection");
        }

        if (!_availableConnections.TryDequeue(out var connection))
        {
            _semaphore.Release(); // Return the slot
            throw new InvalidOperationException("No connections available");
        }

        return connection;
    }

    public void ReleaseConnection(Connection connection)
    {
        _availableConnections.Enqueue(connection);
        _semaphore.Release(); // Signal slot is available
    }
}

// Usage
var pool = new ConnectionPool(maxConnections: 10);

// Multiple threads can use up to 10 connections simultaneously
var threads = new Thread[20];
for (int i = 0; i < 20; i++)
{
    threads[i] = new Thread(() =>
    {
        var conn = pool.AcquireConnection(TimeSpan.FromSeconds(5));
        try
        {
            conn.ExecuteQuery();
        }
        finally
        {
            pool.ReleaseConnection(conn);
        }
    });
    threads[i].Start();
}
```

---

## SemaphoreSlim

### What is SemaphoreSlim?

`SemaphoreSlim` is a lightweight, **async-compatible** version of Semaphore. It's optimized for single-process scenarios and supports asynchronous waiting.

### Why SemaphoreSlim Over Semaphore?

| Feature | Semaphore | SemaphoreSlim |
|---------|-----------|---------------|
| Async support | ❌ No | ✅ Yes (WaitAsync) |
| Cross-process | ✅ Yes | ❌ No |
| Performance | Slower (kernel) | Faster (user mode) |
| Cancellation | Limited | Full support |
| Memory | Higher | Lower |

### Basic Code Example

```csharp
public class AsyncThrottledProcessor
{
    private readonly SemaphoreSlim _semaphore;

    public AsyncThrottledProcessor(int maxConcurrency)
    {
        _semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    }

    public async Task<TResult> ProcessAsync<TResult>(
        Func<CancellationToken, Task<TResult>> operation,
        CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            return await operation(ct);
        }
        finally
        {
            _semaphore.Release();
        }
    }

    public async Task ProcessBatchAsync<T>(
        IEnumerable<T> items,
        Func<T, CancellationToken, Task> operation,
        CancellationToken ct = default)
    {
        await Parallel.ForEachAsync(items, new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount,
            CancellationToken = ct
        }, async (item, ct) =>
        {
            await ProcessAsync(_ => operation(item, ct), ct);
        });
    }
}

// Usage
var throttler = new AsyncThrottledProcessor(maxConcurrency: 5);

await throttler.ProcessBatchAsync(urls, async (url, ct) =>
{
    var content = await _httpClient.GetStringAsync(url, ct);
    await SaveToDatabaseAsync(content, ct);
});
```

### SemaphoreSlim with Timeout

```csharp
public async Task<bool> TryProcessAsync(
    Func<CancellationToken, Task> operation,
    TimeSpan timeout,
    CancellationToken ct = default)
{
    // Wait with timeout
    if (!await _semaphore.WaitAsync(timeout, ct))
    {
        Console.WriteLine("Could not acquire slot within timeout");
        return false;
    }

    try
    {
        await operation(ct);
        return true;
    }
    finally
    {
        _semaphore.Release();
    }
}
```

---

## Comparison and Selection Guide

### Quick Reference Table

| Primitive | Scope | Max Concurrent | Async Support | Use When |
|-----------|-------|----------------|---------------|----------|
| `lock` / Monitor | Process | 1 | ❌ | Simple mutual exclusion, same process |
| `Mutex` | System | 1 | ❌ | Cross-process synchronization |
| `Semaphore` | System | N | ❌ | Resource pool, cross-process |
| `SemaphoreSlim` | Process | N | ✅ | Resource pool, async code |

### Decision Flowchart

```
┌─────────────────────────────────────────────────────────────────┐
│                    SELECTION DECISION TREE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Need synchronization?                                         │
│        │                                                         │
│        ├── Multiple processes need to synchronize?              │
│        │   ├── YES ──> Use Mutex or Semaphore                  │
│        │   │                                                      │
│        │   └── NO (same process) ──> Continue                   │
│        │                                                          │
│        ├── Need to limit concurrent access?                      │
│        │   ├── YES ──> What is N?                              │
│        │   │   ├── N = 1 ──> Use lock                          │
│        │   │   ├── N > 1 ──> Use SemaphoreSlim (async)         │
│        │   │                 or Semaphore (sync)                │
│        │   │                                                      │
│        └── N = 1 (binary lock) ──> Continue                     │
│                                                                  │
│   Using async/await?                                            │
│        ├── YES ──> Use SemaphoreSlim or lock (in sync context) │
│        └── NO ──> Use lock for simplest cases                  │
│                   Use Monitor.Wait/Pulse for signaling          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory and Performance

### Performance Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE BENCHMARKS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Operation Latency (approximate):                              │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  lock (Monitor)       │  ~50-100 nanoseconds           │    │
│   │  Interlocked           │  ~10-20 nanoseconds           │    │
│   │  SemaphoreSlim (fast)  │  ~100-200 nanoseconds        │    │
│   │  SemaphoreSlim (slow)  │  ~1-2 microseconds (kernel)    │    │
│   │  Mutex                 │  ~1-5 microseconds (kernel)    │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Notes:                                                         │
│   - "Fast" path: uncontended acquisition                        │
│   - "Slow" path: requires kernel transition/wait                 │
│   - SemaphoreSlim uses SpinLock briefly before blocking        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Memory Overhead

| Primitive | Memory per Instance |
|-----------|---------------------|
| `lock` object | 0 (uses object header) |
| `Mutex` | ~1KB+ (kernel object) |
| `Semaphore` | ~1KB+ (kernel object) |
| `SemaphoreSlim` | ~100 bytes (user mode) |

---

## Real-World Use Cases

### Use Case 1: Rate-Limited HTTP Client

```csharp
public class RateLimitedHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly SemaphoreSlim _rateLimiter;
    private readonly TimeSpan _minInterval;
    private DateTime _lastRequest = DateTime.MinValue;
    private readonly object _timeLock = new();

    public RateLimitedHttpClient(int requestsPerSecond)
    {
        _httpClient = new HttpClient();
        _rateLimiter = new SemaphoreSlim(requestsPerSecond, requestsPerSecond);
        _minInterval = TimeSpan.FromSeconds(1.0 / requestsPerSecond);
    }

    public async Task<HttpResponseMessage> GetAsync(
        string url,
        CancellationToken ct = default)
    {
        await _rateLimiter.WaitAsync(ct);
        
        try
        {
            // Ensure minimum interval between requests
            lock (_timeLock)
            {
                var elapsed = DateTime.UtcNow - _lastRequest;
                if (elapsed < _minInterval)
                {
                    Thread.Sleep(_minInterval - elapsed);
                }
                _lastRequest = DateTime.UtcNow;
            }

            return await _httpClient.GetAsync(url, ct);
        }
        finally
        {
            // Replenish after time window
            _ = Task.Run(async () =>
            {
                await Task.Delay(TimeSpan.FromSeconds(1));
                _rateLimiter.Release();
            });
        }
    }
}
```

### Use Case 2: Resource Pool Management

```csharp
public class DatabaseConnectionPool : IDisposable
{
    private readonly SemaphoreSlim _availableSlots;
    private readonly ConcurrentQueue<IDbConnection> _connections;
    private readonly Func<IDbConnection> _connectionFactory;
    private int _createdConnections = 0;
    private readonly int _maxConnections;

    public DatabaseConnectionPool(
        int maxConnections,
        Func<IDbConnection> connectionFactory)
    {
        _maxConnections = maxConnections;
        _availableSlots = new SemaphoreSlim(maxConnections, maxConnections);
        _connections = new ConcurrentQueue<IDbConnection>();
        _connectionFactory = connectionFactory;
    }

    public async Task<IDbConnection> AcquireAsync(
        TimeSpan timeout,
        CancellationToken ct = default)
    {
        if (!await _availableSlots.WaitAsync(timeout, ct))
        {
            throw new TimeoutException("Could not acquire connection");
        }

        try
        {
            // Try to reuse existing connection
            if (_connections.TryDequeue(out var existing) && 
                existing.State == ConnectionState.Open)
            {
                return existing;
            }

            // Create new if under limit
            if (Interlocked.Increment(ref _createdConnections) <= _maxConnections)
            {
                return _connectionFactory();
            }

            Interlocked.Decrement(ref _createdConnections);
            throw new InvalidOperationException("Pool exhausted");
        }
        catch
        {
            _availableSlots.Release();
            throw;
        }
    }

    public void Release(IDbConnection connection)
    {
        if (connection.State == ConnectionState.Open)
        {
            _connections.Enqueue(connection);
        }
        _availableSlots.Release();
    }

    public void Dispose()
    {
        _availableSlots.Dispose();
        while (_connections.TryDequeue(out var conn))
        {
            conn.Dispose();
        }
    }
}
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Always use private readonly object for locks
private readonly object _lock = new();

// 2. Keep lock scope minimal
lock (_lock)
{
    // Only critical section
    _counter++;
} // Release immediately

// 3. Use try-finally with Mutex/Semaphore
mutex.WaitOne();
try
{
    // Work
}
finally
{
    mutex.ReleaseMutex();
}

// 4. Prefer SemaphoreSlim for async
await _semaphore.WaitAsync(ct);
try { /* work */ }
finally { _semaphore.Release(); }

// 5. Use ConfigureAwait(false) in library code
await _semaphore.WaitAsync(ct).ConfigureAwait(false);
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Locking on public objects
public void BadMethod()
{
    lock (this) { } // ❌ External code can also lock!
    lock (typeof(MyClass)) { } // ❌ Shared across AppDomain!
}

// PITFALL 2: Deadlock through nested locks
void ThreadA()
{
    lock (_lockA)
    {
        lock (_lockB) { } // May wait for ThreadB
    }
}

void ThreadB()
{
    lock (_lockB)
    {
        lock (_lockA) { } // May wait for ThreadA
    }
}
// DEADLOCK!

// PITFALL 3: Holding lock during I/O
lock (_lock)
{
    await File.ReadAllTextAsync("file.txt"); // ❌ async in lock!
}

// PITFALL 4: Missing release
semaphore.Wait();
// ❌ No try-finally - if exception, slot lost forever

// PITFALL 5: Recursive lock with SemaphoreSlim
// Unlike lock, SemaphoreSlim is NOT reentrant
semaphore.Wait();
semaphore.Wait(); // ❌ Deadlock - same thread blocked!
```

---

## Interview Questions

**Q: What's the difference between Monitor (lock) and Mutex?**
> Monitor is process-local and faster. Mutex can work across processes and is slower due to kernel transitions. Use Monitor for same-process synchronization, Mutex for cross-process.

**Q: When should you use Semaphore over lock?**
> Use Semaphore when you need to allow multiple threads (N > 1) to access a resource simultaneously. Use lock when only one thread should access at a time.

**Q: Why is SemaphoreSlim preferred over Semaphore in async code?**
> SemaphoreSlim supports `WaitAsync()` for non-blocking waits, has lower overhead (user-mode vs kernel-mode), and provides better cancellation support.

**Q: What is a deadlock and how can you prevent it?**
> Deadlock occurs when two threads wait for each other indefinitely. Prevent by: (1) Always acquire locks in the same order, (2) Use timeout-based acquisition, (3) Avoid holding locks while waiting for other resources.

**Q: Can you await inside a lock statement?**
> No, the compiler prevents this. `await` could resume on a different thread, but the lock is thread-affinity based. Use `SemaphoreSlim` instead for async synchronization.

**Q: What's the difference between Semaphore and SemaphoreSlim?**
> Semaphore is a kernel object that can work across processes but doesn't support async. SemaphoreSlim is lighter, process-local only, and supports async waits via `WaitAsync()`.
