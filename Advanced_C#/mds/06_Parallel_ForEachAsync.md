# Parallel.ForEachAsync in C#

## 🧠 What Is It?

`Parallel.ForEachAsync` (introduced in .NET 6) is a method that **processes a collection of items concurrently using async/await**, with built-in control over the **degree of parallelism** (how many tasks run at once).

It's the modern answer to: *"How do I process 1000 items concurrently, without spawning 1000 tasks and crashing my server?"*

---

## 🌍 Real-World Analogy

Imagine a restaurant kitchen with 4 chefs. Orders come in one by one. You don't want to assign each order a brand-new chef (you'd run out of chefs fast). Instead, you keep **4 chefs busy at all times**, each picking up the next order whenever they finish one. That's exactly what `Parallel.ForEachAsync` with `MaxDegreeOfParallelism = 4` does.

---

## ⚙️ Memory & Execution Model

```
Items:  [A]  [B]  [C]  [D]  [E]  [F]  [G]  [H]
         │    │    │    │
         ▼    ▼    ▼    ▼
       Task Task Task Task     ← MaxDegreeOfParallelism = 4
        (concurrent async work)
              │
              ▼  when one finishes → picks up next item [E], [F]...
```

> **Thread Pool**: `Parallel.ForEachAsync` schedules work on the **ThreadPool**, not raw threads. The `async` lambdas can yield with `await`, freeing up threads during I/O — this is extremely memory- and CPU-efficient.

---

## 💻 Code Example

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

class Program
{
    static readonly HttpClient _http = new HttpClient();

    static async Task Main()
    {
        var urls = new List<string>
        {
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
            "https://httpbin.org/delay/1",
        };

        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 3  // max 3 concurrent requests at a time
        };

        await Parallel.ForEachAsync(urls, options, async (url, cancellationToken) =>
        {
            var response = await _http.GetAsync(url, cancellationToken);
            Console.WriteLine($"[{Thread.CurrentThread.ManagedThreadId}] {url} → {response.StatusCode}");
        });

        Console.WriteLine("All URLs processed.");
    }
}
```

---

## 🔁 Comparison: `Parallel.ForEachAsync` vs `foreach` vs `Task.WhenAll`

| Feature | `foreach` (sequential) | `Task.WhenAll` | `Parallel.ForEachAsync` |
|---|---|---|---|
| Concurrency | ❌ None — one at a time | ✅ All at once | ✅ Controlled batches |
| Async support | ✅ (with `await`) | ✅ | ✅ |
| Degree of parallelism control | ❌ | ❌ (all fire at once) | ✅ `MaxDegreeOfParallelism` |
| Risk of overwhelming server | ❌ Low | ⚠️ High (1000 items = 1000 tasks) | ✅ Controlled |
| Best for | Simple sequential work | Small fixed sets | Large collections, I/O-bound |

### Visual: Task.WhenAll — "Fire all at once"
```
Items: [1][2][3][4][5][6][7][8][9][10]
        ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓
       ALL tasks launched simultaneously
       → Can overwhelm DB / API / server
```

### Visual: Parallel.ForEachAsync — "Sliding window"
```
Items: [1][2][3]  ← running (maxDegree=3)
              ↓ [1] finishes → [4] starts
       [2][3][4]  ← running
              ↓ [2] finishes → [5] starts
       [3][4][5]  ← running ...
```

---

## ✅ When to Use

| Scenario | Best Choice |
|---|---|
| Simple sequential processing | `foreach` |
| Small fixed set, all async | `Task.WhenAll` |
| Large collection, need rate limiting | `Parallel.ForEachAsync` |
| CPU-bound work (no async) | `Parallel.ForEach` |

---

## 📌 Summary

> `Parallel.ForEachAsync` is the **go-to tool** for processing large async collections efficiently. It prevents overloading external services by capping concurrency, while still being far faster than sequential processing.
