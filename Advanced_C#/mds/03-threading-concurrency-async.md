# Threading, Concurrency, and Async in C#

## Table of Contents
1. [Thread Fundamentals](#thread-fundamentals)
2. [Thread Synchronization Primitives](#thread-synchronization-primitives)
3. [Thread Pool and Thread Pool Starvation](#thread-pool-and-thread-pool-starvation)
4. [Async/Await Deep Dive](#asyncawait-deep-dive)
5. [Parallel Programming](#parallel-programming)
6. [Concurrent Collections](#concurrent-collections)
7. [Channels](#channels)
8. [IAsyncEnumerable](#iasyncenumerable)

---

## Thread Fundamentals

### What is a Thread?

A thread is an **independent execution path** within a process. A process can have multiple threads executing concurrently.

### Thread States

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREAD LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Unstarted ───> Started ───> Running ───> Completed            │
│       │              │           │                              │
│       │              │           ▼                              │
│       │              │      [Wait/Sleep/Join] ───> Running    │
│       │              │                                           │
│       └──────────────┴─────────────────────────────────────────>│
│                                                                  │
│   States:                                                        │
│   • Unstarted: Thread created but not started                    │
│   • Running: Actively executing                                  │
│   • WaitSleepJoin: Blocked waiting                               │
│   • Completed: Finished execution                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Creating and Managing Threads

```csharp
// Method 1: Thread class (low-level, avoid in most cases)
var thread = new Thread(() =>
{
    Console.WriteLine($"Running on thread: {Thread.CurrentThread.ManagedThreadId}");
});
thread.Start();
thread.Join();  // Wait for completion

// Method 2: Task (preferred in modern C#)
var task = Task.Run(() =>
{
    Console.WriteLine($"Running on thread pool: {Thread.CurrentThread.ManagedThreadId}");
});
await task;

// Method 3: ThreadPool directly (rarely needed)
ThreadPool.QueueUserWorkItem(_ =>
d{
    Console.WriteLine("Thread pool work item");
});
```

### Thread.Join()

`Thread.Join()` blocks the calling thread until the target thread completes.

```csharp
public void DownloadFiles()
{
    var threads = new List<Thread>();
    
    foreach (var url in urls)
    {
        var thread = new Thread(() => DownloadFile(url));
        thread.Start();
        threads.Add(thread);
    }
    
    // Wait for ALL threads to complete
    foreach (var thread in threads)
    {
        thread.Join();  // Blocks until this thread finishes
    }
    
    Console.WriteLine("All downloads complete!");
}
```

---

## Thread Synchronization Primitives

### Comparison Table

| Primitive | Use Case | Scope | Performance |
|-----------|----------|-------|-------------|
| **lock** | Simple mutual exclusion | Process-only | Fast |
| **Monitor** | Fine-grained control | Process-only | Fast |
| **Mutex** | Cross-process synchronization | System-wide | Slow |
| **Semaphore** | Limit concurrent access | System-wide | Medium |
| **SemaphoreSlim** | Async-compatible semaphore | Process-only | Fast |
| **Interlocked** | Atomic operations on variables | N/A | Very fast |

### lock Statement

```csharp
private readonly object _lock = new();
private int _counter = 0;

public void Increment()
{
    lock (_lock)
    {
        _counter++;  // Only one thread at a time
    }
}

// Compiler transforms to:
Monitor.Enter(_lock);
try
{
    _counter++;
}
finally
{
    Monitor.Exit(_lock);
}
```

### Mutex vs lock

```csharp
// lock - fast, process-local
lock (_sync) { /* critical section */ }

// Mutex - can be cross-process
using (var mutex = new Mutex(false, "Global\\MyAppMutex"))
{
    mutex.WaitOne();  // Wait for ownership
    try
    {
        // critical section - only one process at a time
    }
    finally
    {
        mutex.ReleaseMutex();
    }
}

// Use case: preventing multiple app instances
var mutex = new Mutex(true, "MyAppInstance", out bool createdNew);
if (!createdNew)
{
    MessageBox.Show("App already running!");
    Application.Exit();
    return;
}
```

### Semaphore vs SemaphoreSlim

```csharp
// Semaphore - heavyweight, can be named/cross-process
var semaphore = new Semaphore(3, 3);  // 3 concurrent, max 3

// SemaphoreSlim - lightweight, async-compatible
var slim = new SemaphoreSlim(5, 10);  // 5 initial, max 10

// Usage with async
public async Task<T> GetResourceAsync()
{
    await slim.WaitAsync();  // Async wait!
    try
    {
        return await FetchDataAsync();
    }
    finally
    {
        slim.Release();
    }
}
```

### Interlocked - Lock-Free Atomic Operations

```csharp
private int _counter = 0;

// Atomic increment
Interlocked.Increment(ref _counter);

// Atomic compare-and-swap
Interlocked.CompareExchange(ref _counter, newValue, expectedValue);

// Memory barrier - ensures ordering
Interlocked.MemoryBarrier();

// Use case: Simple thread-safe counter
public class ThreadSafeCounter
{
    private long _value = 0;
    
    public void Increment() => Interlocked.Increment(ref _value);
    public long GetValue() => Interlocked.Read(ref _value);
}
```

---

## Thread Pool and Thread Pool Starvation

### What is Thread Pool Starvation?

```
┌─────────────────────────────────────────────────────────────────┐
│                    THREAD POOL STARVATION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Normal Operation:                                              │
│   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      │
│   │Task1│ │Task2│ │Task3│ │Task4│ │Task5│ │Task6│ │Task7│      │
│   └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘      │
│      └───────┴───────┴───────┘       └───────┴───────┘          │
│          Reuses threads efficiently                              │
│                                                                  │
│   Starvation:                                                    │
│   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                      │
│   │Task1│ │Task2│ │Task3│ │Task4│ │Task5│ (ALL BLOCKED!)         │
│   │🔒   │ │🔒   │ │🔒   │ │🔒   │ │🔒   │                      │
│   └─────┘ └─────┘ └─────┘ └─────┘ └─────┘                        │
│                                                                  │
│   Thread 6+ cannot run because all threads blocked waiting!    │
│   Tasks 6, 7, 8... queued but no threads available              │
│                                                                  │
│   Causes:                                                        │
│   • Blocking async calls (.Wait(), .Result)                     │
│   • Long-running operations on thread pool                      │
│   • Synchronization that never releases                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Best Practices

```csharp
// ❌ BAD: Blocking thread pool threads
public T GetDataBad()
{
    return GetDataAsync().Result;  // DEADLOCK/STARVATION RISK!
}

// ✅ GOOD: Use async all the way
public async Task<T> GetDataGoodAsync()
{
    return await GetDataAsync();  // Releases thread while waiting
}

// ❌ BAD: Long-running on thread pool
Task.Run(() => { /* runs for 10 minutes */ });

// ✅ GOOD: Use dedicated thread for long-running
new Thread(() => { /* long running */ }) 
{ 
    IsBackground = true 
}.Start();
```

---

## Async/Await Deep Dive

### How async/await Works

```csharp
// What you write:
public async Task<string> GetDataAsync()
{
    var response = await httpClient.GetAsync(url);
    var content = await response.Content.ReadAsStringAsync();
    return content;
}

// What the compiler generates:
public Task<string> GetDataAsync()
{
    var stateMachine = new GetDataAsyncStateMachine();
    stateMachine.httpClient = httpClient;
    stateMachine.url = url;
    stateMachine.Builder = AsyncTaskMethodBuilder<string>.Create();
    stateMachine.State = -1;
    stateMachine.Builder.Start(ref stateMachine);
    return stateMachine.Builder.Task;
}

// State machine:
[CompilerGenerated]
private struct GetDataAsyncStateMachine : IAsyncStateMachine
{
    public int State;
    public AsyncTaskMethodBuilder<string> Builder;
    // ... fields for locals and parameters ...
    
    public void MoveNext()
    {
        // Implementation that:
        // 1. Runs code up to first await
        // 2. When operation completes, callback resumes at right position
        // 3. Runs to next await or completion
    }
}
```

### Async State Machine Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASYNC/AWAIT STATE MACHINE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. CALLER calls async method                                   │
│      │                                                           │
│      ▼                                                           │
│   2. State machine created, starts executing                      │
│      │                                                           │
│      ▼                                                           │
│   3. Method runs until first 'await'                           │
│      │                                                           │
│      ▼                                                           │
│   4. If operation not complete:                                   │
│      • Register continuation callback                           │
│      • Return Task to caller (uncompleted)                        │
│      • Thread returns to thread pool                             │
│      │                                                           │
│      ▼ (when operation completes)                                │
│   5. Callback invoked → MoveNext() called                       │
│      │                                                           │
│      ▼                                                           │
│   6. Resume execution after await                                │
│      │                                                           │
│      └──────> Continue to next await or completion ────────>    │
│                                                                  │
│   Key insight: NO THREAD is blocked during await!                 │
│   The thread is released and can do other work.                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ConfigureAwait(false)

```csharp
// UI Application context (has synchronization context)
public async Task LoadDataAsync()
{
    // Captures UI context - will resume on UI thread
    var data = await httpClient.GetStringAsync(url);
    textBox.Text = data;  // ✓ Safe - on UI thread
    
    // Library code should use ConfigureAwait(false)
    var moreData = await httpClient.GetStringAsync(url2)
        .ConfigureAwait(false);  // Don't capture context
    
    // After ConfigureAwait(false), not necessarily on UI thread!
    // textBox.Text = moreData;  // ❌ Potential cross-thread exception
}

// Best practice for library code:
public async Task<T> FetchDataAsync()
{
    var response = await httpClient.GetAsync(url)
        .ConfigureAwait(false);  // Library code - no UI needed
    return await response.Content.ReadAsAsync<T>()
        .ConfigureAwait(false);
}
```

---

## Parallel Programming

### Parallel.ForEach

```csharp
// Sequential
foreach (var item in items)
{
    Process(item);
}

// Parallel
Parallel.ForEach(items, item =>
{
    Process(item);  // Multiple items processed concurrently
});

// With options
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount
};

Parallel.ForEach(items, options, item =>
{
    Process(item);
});
```

### Parallel.ForEachAsync (.NET 6+)

```csharp
// async-compatible parallel processing
await Parallel.ForEachAsync(urls, async (url, cancellationToken) =>
{
    var content = await httpClient.GetStringAsync(url, cancellationToken);
    Console.WriteLine($"Downloaded {url}: {content.Length} bytes");
});

// Comparison:
// Task.WhenAll        - All tasks start immediately, unlimited concurrency
// Parallel.ForEachAsync - Controls concurrency, async-aware
// Sequential await    - One at a time
```

### Comparison: foreach vs Task.WhenAll vs Parallel.ForEachAsync

```csharp
// 1. Sequential - one at a time
foreach (var url in urls)
{
    await DownloadAsync(url);  // Slowest but most predictable
}

// 2. Task.WhenAll - all at once
var tasks = urls.Select(DownloadAsync);
await Task.WhenAll(tasks);  // Fastest but can overwhelm resources

// 3. Parallel.ForEachAsync - controlled concurrency
await Parallel.ForEachAsync(urls, new ParallelOptions 
{ 
    MaxDegreeOfParallelism = 4 
}, async (url, ct) =>
{
    await DownloadAsync(url);  // Balanced - 4 at a time
});
```

---

## Concurrent Collections

### Thread-Safe Collections Overview

| Collection | Use Case | Thread Safety |
|------------|----------|---------------|
| **ConcurrentDictionary** | Key-value storage, caching | Lock-free, very fast |
| **ConcurrentQueue** | Producer-consumer, work queue | Lock-free enqueue/dequeue |
| **ConcurrentBag** | Unordered collection, no duplicates needed | Thread-safe add/remove |
| **BlockingCollection** | Bounded buffering, producer-consumer | Wraps other collections |

### ConcurrentDictionary

```csharp
var cache = new ConcurrentDictionary<int, User>();

// Thread-safe operations - no explicit locking needed
var user = cache.GetOrAdd(userId, id => LoadUserFromDb(id));

// Atomic update
cache.AddOrUpdate(userId, 
    addValueFactory: id => new User { Id = id },
    updateValueFactory: (id, existing) => 
    {
        existing.LastAccessed = DateTime.Now;
        return existing;
    });

// Thread-safe removal
cache.TryRemove(userId, out var removedUser);
```

### ConcurrentQueue vs Channel

```csharp
// ConcurrentQueue - basic thread-safe queue
var queue = new ConcurrentQueue<WorkItem>();
queue.Enqueue(new WorkItem { Data = "task1" });

if (queue.TryDequeue(out var item))
{
    Process(item);
}

// Channel (.NET Core 3.0+) - async-capable, backpressure support
var channel = Channel.CreateUnbounded<WorkItem>();

// Writer (async)
await channel.Writer.WriteAsync(new WorkItem { Data = "task1" });
channel.Writer.Complete();  // Signal completion

// Reader (async)
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessAsync(item);
}
```

---

## Channels

### What are Channels?

Channels provide a **typed, async-capable producer-consumer queue** with built-in backpressure support.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANNEL ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Producer(s)                 Channel                   Consumer(s)│
│       │                      ┌─────┐                       │    │
│       │────────────────────>│     │───────────────────────│    │
│       │     WriteAsync()     │     │     ReadAsync()      │    │
│       │                      │     │                       │    │
│       │     (blocks if full) │     │     (blocks if empty) │    │
│       │                      └─────┘                       │    │
│       │                       ↑                           │    │
│       │                    Bounded                        │    │
│       │                   capacity                        │    │
│                                                                  │
│   Features:                                                      │
│   • Async read/write                                            │
│   • Bounded or unbounded capacity                               │
│   • Multiple producers/consumers                                │
│   • Completion signaling                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Channel Examples

```csharp
// Unbounded channel - unlimited capacity
var unbounded = Channel.CreateUnbounded<int>();

// Bounded channel - limits memory usage
var bounded = Channel.CreateBounded<int>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait  // Block writers when full
});

// Producer
async Task ProducerAsync(ChannelWriter<int> writer)
{
    for (int i = 0; i < 100; i++)
    {
        await writer.WriteAsync(i);
        Console.WriteLine($"Produced: {i}");
    }
    writer.Complete();  // Signal no more data
}

// Consumer
async Task ConsumerAsync(ChannelReader<int> reader)
{
    await foreach (var item in reader.ReadAllAsync())
    {
        Console.WriteLine($"Consumed: {item}");
        await Task.Delay(100);  // Simulate work
    }
    Console.WriteLine("Consumer done");
}

// Usage
var channel = Channel.CreateBounded<int>(10);
_ = Task.Run(() => ProducerAsync(channel.Writer));
_ = Task.Run(() => ConsumerAsync(channel.Reader));
```

---

## IAsyncEnumerable

### What is IAsyncEnumerable?

`IAsyncEnumerable<T>` allows **async streaming** of data - yielding items as they become available, without waiting for the entire collection.

```
┌─────────────────────────────────────────────────────────────────┐
│                    IAsyncEnumerable vs IEnumerable              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   IEnumerable<T> (Synchronous):                                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  All data loaded first:                                 │    │
│   │  [A] → [B] → [C] → [D] → [E] → Consumer                  │    │
│   │           ↑                                              │    │
│   │     Wait for ALL to be ready                            │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   IAsyncEnumerable<T> (Asynchronous):                           │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Stream as ready:                                      │    │
│   │  [A] ──────> [B] ──────> [C] ──────> [D] ────> Consumer │    │
│   │    ↑           ↑           ↑                            │    │
│   │  Ready       Ready       Ready                          │    │
│   │                                                          │    │
│   │  Consumer processes each item as it arrives!            │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Use case: Database streaming, file reading, API pagination   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```csharp
// Producer - async enumerable method
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync(ct);
    
    await using var command = new SqlCommand("SELECT * FROM Orders", connection);
    await using var reader = await command.ExecuteReaderAsync(ct);
    
    while (await reader.ReadAsync(ct))
    {
        yield return new Order
        {
            Id = reader.GetInt32(0),
            CustomerName = reader.GetString(1),
            Amount = reader.GetDecimal(2)
        };
        // Item yielded immediately - consumer can process while reading continues
    }
}

// Consumer - await foreach
public async Task ProcessOrdersAsync()
{
    await foreach (var order in GetOrdersAsync())
    {
        Console.WriteLine($"Processing order {order.Id}");
        await SendNotificationAsync(order);
    }
}
```

---

## Interlocked vs lock

```csharp
// ❌ NOT thread-safe
_counter++;

// ✅ Thread-safe with lock (slower, but safe for complex operations)
lock (_sync)
{
    _counter++;
    _lastUpdated = DateTime.Now;
}

// ✅ Thread-safe with Interlocked (fastest for simple operations)
Interlocked.Increment(ref _counter);

// Performance comparison (approximate):
// Interlocked: ~10-20 ns (hardware atomic instruction)
// lock: ~50-100 ns (OS mutex + context switch potential)
```

---

## Interview Questions

**Q: What's the difference between Thread and Task?**
> Thread is a low-level OS construct that consumes ~1MB of stack memory. Task is a higher-level abstraction that runs on the thread pool, supports async/await, and is much more efficient.

**Q: When should you use ConfigureAwait(false)?**> In library code where you don't need to return to the original context (like UI thread). It avoids capturing the synchronization context, improving performance.

**Q: What's thread pool starvation?**> When all thread pool threads are blocked waiting, preventing other work from executing. Caused by blocking calls (.Wait(), .Result) or long-running operations on thread pool threads.

**Q: ConcurrentDictionary vs Dictionary + lock?**> ConcurrentDictionary uses fine-grained locking (lock-free for many operations), scales better, and has atomic operations like GetOrAdd and AddOrUpdate.

**Q: When would you use Channels over ConcurrentQueue?**> Channels support async operations, backpressure (bounded capacity), and completion signaling. Use when you need async producer-consumer patterns.

**Q: What's the difference between IAsyncEnumerable and Task<IEnumerable>?**> IAsyncEnumerable streams items as they become available (await foreach). Task<IEnumerable> waits for ALL items before returning any.
