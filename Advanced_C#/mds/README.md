# Advanced C# — Research Topics

A complete set of professional reference documents covering **Advanced Generics, Events, Delegates, Collections, Threading, Concurrency, Performance, Async patterns, and Unit Testing** in C#. Each file contains real-world explanations, memory visualizations, practical code examples, and comparison tables.

---

## 📂 Part 1: Advanced C# Foundations

| # | Topic | File | Key Concepts |
|---|---|---|---|
| 01 | **Advanced Generics** | [01-advanced-generics.md](./01-advanced-generics.md) | Type constraints, covariance, contravariance, generic methods |
| 02 | **Events & Observer Pattern** | [02-events-observer-pattern.md](./02-events-observer-pattern.md) | `event`, `EventHandler`, delegate, publisher-subscriber |
| 03 | **Threading, Concurrency & Async** | [03-threading-concurrency-async.md](./03-threading-concurrency-async.md) | `Task`, `async/await`, `Thread`, `ThreadPool`, cancellation |
| 04 | **Performance & Memory** | [04-performance-memory.md](./04-performance-memory.md) | Stack vs Heap, `Span<T>`, `Memory<T>`, GC, boxing/unboxing |

---

## 📂 Part 2: Threading & Concurrency — Deep Dives

| # | Topic | File | Key Concepts |
|---|---|---|---|
| 05 | **Thread.Join()** | [05_Thread_Join.md](./05_Thread_Join.md) | Blocking, thread coordination, timeout, deadlock |
| 06 | **Parallel.ForEachAsync** | [06_Parallel_ForEachAsync.md](./06_Parallel_ForEachAsync.md) | Async parallelism, MaxDegreeOfParallelism, vs Task.WhenAll |
| 07 | **IAsyncEnumerable** | [07_IAsyncEnumerable.md](./07_IAsyncEnumerable.md) | Async streaming, `await foreach`, memory efficiency |
| 08 | **Monitor, Mutex, Semaphore, SemaphoreSlim** | [08_Monitor_Mutex_Semaphore.md](./08_Monitor_Mutex_Semaphore.md) | Synchronization primitives, locking, cross-process |
| 09 | **Interlocked** | [09_Interlocked.md](./09_Interlocked.md) | Atomic operations, lock-free counters, CAS |
| 10 | **Async File I/O** | [10_File_IO_Async.md](./10_File_IO_Async.md) | `ReadAllTextAsync`, `StreamReader`, IOCP, async write |
| 11 | **IDisposable & using** | [11_IDisposable_Using.md](./11_IDisposable_Using.md) | Resource cleanup, using block vs declaration, `await using` |
| 12 | **System.Threading.Channels** | [12_Channels.md](./12_Channels.md) | Producer-consumer, bounded/unbounded, backpressure |
| 13 | **Concurrent Collections** | [13_Concurrent_Collections.md](./13_Concurrent_Collections.md) | ConcurrentDictionary, ConcurrentBag, ConcurrentQueue |
| 14 | **Thread Pool Starvation** | [14_Thread_Pool_Starvation.md](./14_Thread_Pool_Starvation.md) | Blocking anti-patterns, async all the way, LongRunning |

---

## 📂 Part 3: Unit Testing

| # | Topic | File | Key Concepts |
|---|---|---|---|
| 15 | **Unit Testing with xUnit & Shouldly** | [15_Unit_Testing.md](./15_Unit_Testing.md) | AAA pattern, [Fact], [Theory], test types |
| 16 | **Mocking with Moq** | [16_Mocking.md](./16_Mocking.md) | Mock vs Stub vs Fake, Setup, Verify, async mocks |

---

## 📂 Part 4: Delegates & Collections — Deep Dives

| # | Topic | File | Key Concepts |
|---|---|---|---|
| 17 | **Delegates — Complete Guide** | [15-delegates.md](./15-delegates.md) | Delegate declaration, multicast, closures, Action/Func/Predicate, lambdas, IL internals |
| 18 | **List\<T\> — Complete Deep Dive** | [16-list-deep-dive.md](./16-list-deep-dive.md) | Arrays vs collections, interface hierarchy, internals, capacity doubling, complexity |
| 19 | **Queue, Stack, Span\<T\> & Collections** | [17-queue-stack-span.md](./17-queue-stack-span.md) | FIFO/LIFO, circular array, Span zero-copy, full collection comparison |
| 20 | **SortedList, Dictionary & HashSet** | [18-sortedlist-dictionary-hashset.md](./18-sortedlist-dictionary-hashset.md) | Hashing theory, collision strategies, GetHashCode+Equals contract, internals |

---

## 🔁 Required Comparisons Quick Reference

| Comparison | Covered In |
|---|---|
| `Parallel.ForEachAsync` vs `Task.WhenAll` | [06_Parallel_ForEachAsync.md](./06_Parallel_ForEachAsync.md) |
| `IAsyncEnumerable` vs `IEnumerable` | [07_IAsyncEnumerable.md](./07_IAsyncEnumerable.md) |
| `Interlocked` vs `lock` | [09_Interlocked.md](./09_Interlocked.md) |
| `Semaphore` vs `SemaphoreSlim` | [08_Monitor_Mutex_Semaphore.md](./08_Monitor_Mutex_Semaphore.md) |
| `Monitor` vs `Mutex` | [08_Monitor_Mutex_Semaphore.md](./08_Monitor_Mutex_Semaphore.md) |
| `ConcurrentQueue` vs `Channel` | [13_Concurrent_Collections.md](./13_Concurrent_Collections.md) + [12_Channels.md](./12_Channels.md) |
| `ConcurrentDictionary` vs `Dictionary` | [13_Concurrent_Collections.md](./13_Concurrent_Collections.md) |
| `using` block vs `using` declaration | [11_IDisposable_Using.md](./11_IDisposable_Using.md) |
| `Task` vs `Thread` vs `ThreadPool` | [03-threading-concurrency-async.md](./03-threading-concurrency-async.md) |
| `Span<T>` vs `Memory<T>` | [04-performance-memory.md](./04-performance-memory.md) |
| `Action` vs `Func` vs `Predicate` | [15-delegates.md](./15-delegates.md) |
| `Delegate` vs `Interface` | [15-delegates.md](./15-delegates.md) |
| `Array` vs `List<T>` | [16-list-deep-dive.md](./16-list-deep-dive.md) |
| `Generic` vs `Non-Generic` collections | [16-list-deep-dive.md](./16-list-deep-dive.md) |
| `Queue<T>` vs `Stack<T>` | [17-queue-stack-span.md](./17-queue-stack-span.md) |
| `Span<T>` vs LINQ slicing | [17-queue-stack-span.md](./17-queue-stack-span.md) |
| `Dictionary` vs `SortedList` vs `SortedDictionary` | [18-sortedlist-dictionary-hashset.md](./18-sortedlist-dictionary-hashset.md) |
| `List.Contains` O(n) vs `HashSet.Contains` O(1) | [18-sortedlist-dictionary-hashset.md](./18-sortedlist-dictionary-hashset.md) |
| Linear Search vs Binary Search vs Hashing | [18-sortedlist-dictionary-hashset.md](./18-sortedlist-dictionary-hashset.md) |

---

## 📋 Recommended Reading Order

**If new to Advanced C#, start here:**
1. `01-advanced-generics.md` — foundation for generic types
2. `02-events-observer-pattern.md` — event-driven design
3. `15-delegates.md` — understand delegates before events
4. `03-threading-concurrency-async.md` — the big picture of async
5. `04-performance-memory.md` — understand memory before going deep

**Collections track:**
6. `16-list-deep-dive.md` → `17-queue-stack-span.md` → `18-sortedlist-dictionary-hashset.md`

**Threading deep dives:**
7. `14_Thread_Pool_Starvation.md` — understand the danger first
8. `05_Thread_Join.md` → `09_Interlocked.md` → `08_Monitor_Mutex_Semaphore.md`
9. `07_IAsyncEnumerable.md` → `06_Parallel_ForEachAsync.md`
10. `12_Channels.md` → `13_Concurrent_Collections.md`
11. `10_File_IO_Async.md` → `11_IDisposable_Using.md`

**Unit Testing track:**
12. `15_Unit_Testing.md` → `16_Mocking.md`

---

## 🛠️ Tools & Frameworks Used in Examples

| Tool | Purpose | Install |
|---|---|---|
| xUnit | Unit test framework | `dotnet add package xunit` |
| Shouldly | Fluent assertion library | `dotnet add package Shouldly` |
| Moq | Mocking framework | `dotnet add package Moq` |
| System.Threading.Channels | Built-in async channel | Part of .NET 5+ |
| System.Collections.Concurrent | Thread-safe collections | Part of .NET (all versions) |