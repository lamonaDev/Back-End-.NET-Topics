# Synchronization: Monitor, Mutex, Semaphore & SemaphoreSlim

## 🧠 The Problem They Solve

When multiple threads access **shared resources** simultaneously, you get **race conditions** — unpredictable bugs where the outcome depends on thread scheduling. Synchronization primitives are locks that control access so only the right number of threads can be in a critical section at a time.

---

## 🌍 Real-World Analogy

| Primitive | Analogy |
|---|---|
| **Monitor / lock** | A single-occupancy office. Only 1 person inside. Key hangs on the door. |
| **Mutex** | Same office, but the key works **across multiple buildings** (processes). |
| **Semaphore** | A car park with N spaces. N cars can park at once. If full, you wait outside. |
| **SemaphoreSlim** | Same car park, but **for one building only** — lighter, async-friendly. |

---

## 🔒 1. Monitor (and `lock`)

`Monitor` is the foundation of the `lock` keyword. It provides **exclusive access** — only one thread at a time.

```csharp
private readonly object _syncRoot = new object();
private int _counter = 0;

// Using lock (syntactic sugar over Monitor.Enter/Exit)
public void Increment()
{
    lock (_syncRoot)
    {
        _counter++; // Only one thread at a time
    }
}

// Equivalent using Monitor directly:
public void IncrementManual()
{
    Monitor.Enter(_syncRoot);
    try
    {
        _counter++;
    }
    finally
    {
        Monitor.Exit(_syncRoot); // Always release in finally!
    }
}
```

### Memory Model
```
Thread 1: lock(_syncRoot) → acquires lock → executes → releases lock
Thread 2: lock(_syncRoot) → BLOCKED (waits) ──────────────────────► acquires lock
Thread 3: lock(_syncRoot) → BLOCKED (waits) ──────────────────────────────────────► ...
```

> **Scope**: In-process only. `Monitor` objects live on the **managed heap**. The sync root object's header contains a lock bit used by the CLR.

---

## 🔑 2. Mutex

A `Mutex` works like `Monitor` but can be named and **shared across processes**. It's heavier but cross-process safe.

```csharp
using System.Threading;

// Named mutex — visible across processes on the OS
using var mutex = new Mutex(false, "MyApp_GlobalMutex");

Console.WriteLine("Waiting for mutex...");
mutex.WaitOne(); // Blocks until acquired

try
{
    Console.WriteLine("Mutex acquired. Doing exclusive work...");
    Thread.Sleep(2000);
}
finally
{
    mutex.ReleaseMutex(); // MUST release
    Console.WriteLine("Mutex released.");
}
```

### ⚠️ Key Difference: Monitor vs Mutex

| Feature | Monitor / lock | Mutex |
|---|---|---|
| Scope | Single process | Cross-process |
| Performance | Very fast (managed) | Slower (OS kernel object) |
| Named? | ❌ | ✅ |
| Async-friendly? | ❌ | ❌ |
| Auto-released on thread exit? | ✅ (if using `lock`) | ❌ Can become "abandoned" |

---

## 🚦 3. Semaphore

A `Semaphore` allows **N threads** to enter a section concurrently — not just one. Like a bouncer at a club who lets in up to N people.

```csharp
// Allow max 3 threads concurrently
using var semaphore = new Semaphore(3, 3); // initialCount=3, maxCount=3

void DoWork(int id)
{
    Console.WriteLine($"Thread {id} waiting...");
    semaphore.WaitOne(); // Decrements count; blocks if count == 0

    try
    {
        Console.WriteLine($"Thread {id} working...");
        Thread.Sleep(1000);
    }
    finally
    {
        semaphore.Release(); // Increments count
        Console.WriteLine($"Thread {id} done.");
    }
}

var threads = Enumerable.Range(1, 8).Select(i => new Thread(() => DoWork(i))).ToList();
threads.ForEach(t => t.Start());
threads.ForEach(t => t.Join());
```

---

## ⚡ 4. SemaphoreSlim

`SemaphoreSlim` is a **lightweight, async-capable** semaphore designed for **in-process use**. It's the go-to choice for throttling async operations.

```csharp
using System.Threading;
using System.Threading.Tasks;

var slim = new SemaphoreSlim(3); // Allow 3 concurrent async tasks

var tasks = Enumerable.Range(1, 10).Select(async i =>
{
    await slim.WaitAsync(); // Async wait — does NOT block a thread
    try
    {
        Console.WriteLine($"Task {i} started");
        await Task.Delay(500); // Simulate async work
        Console.WriteLine($"Task {i} finished");
    }
    finally
    {
        slim.Release();
    }
});

await Task.WhenAll(tasks);
```

### Memory & Execution Model for SemaphoreSlim

```
 Tasks: [1][2][3]  ← running (count = 0, at limit)
        [4][5][6]  ← waiting in async queue (NOT blocking threads!)
                        │
        [1] finishes → count++ → [4] awakened and starts
```

> `SemaphoreSlim.WaitAsync()` uses a **continuation queue** internally — waiting tasks do not hold a thread. This is critical for scalable async code.

---

## 📊 Full Comparison Table

| Feature | Monitor/lock | Mutex | Semaphore | SemaphoreSlim |
|---|---|---|---|---|
| Max concurrent threads | 1 | 1 | N | N |
| Cross-process | ❌ | ✅ | ✅ (named) | ❌ |
| Async (`WaitAsync`) | ❌ | ❌ | ❌ | ✅ |
| Performance | 🔥 Fastest | 🐢 Slowest | Medium | Fast |
| Best use case | Simple locks | Cross-process exclusion | Rate limiting (sync) | Rate limiting (async) |

---

## 📌 Summary

> Use **`lock`/Monitor** for simple single-threaded exclusion. Use **Mutex** when you need cross-process locks. Use **Semaphore** for sync rate limiting. Use **SemaphoreSlim** — with `WaitAsync()` — for async throttling in modern .NET applications.
