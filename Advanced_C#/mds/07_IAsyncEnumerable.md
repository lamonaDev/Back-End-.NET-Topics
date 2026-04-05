# IAsyncEnumerable\<T\> in C#

## 🧠 What Is It?

`IAsyncEnumerable<T>` is an interface that lets you **stream data asynchronously, one item at a time**, using `await foreach`. It's the async version of `IEnumerable<T>`.

Instead of waiting for **all** data to be ready, you process each item **as soon as it arrives** — like a stream, not a bucket.

---

## 🌍 Real-World Analogy

**`IEnumerable<T>` (sync):** A waiter brings your entire meal at once. You sit there waiting until every dish is plated.

**`IAsyncEnumerable<T>` (async):** A waiter brings each dish **as soon as it's ready**. You start eating the appetizer while the main course is still cooking.

This matters when data comes from a database, an API, a file, or any source where results trickle in over time.

---

## ⚙️ Memory & Execution Model

```
Producer (async generator)        Consumer (await foreach)
──────────────────────────        ──────────────────────────
yield return item1  ──────────►  receives item1, processes it
                                          │ (awaits next)
yield return item2  ──────────►  receives item2, processes it
                                          │ (awaits next)
yield return item3  ──────────►  receives item3, processes it
```

> **Memory advantage**: Only **one item at a time** lives in memory. Compare this to `ToList()` which loads the entire dataset into a `List<T>` on the heap — potentially gigabytes for large queries.

---

## 💻 Code Example — Producing and Consuming

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

class Program
{
    // Producer: async stream of sensor readings
    static async IAsyncEnumerable<int> GetSensorReadingsAsync()
    {
        for (int i = 1; i <= 5; i++)
        {
            await Task.Delay(500); // Simulate async data arrival (DB, API, sensor)
            Console.WriteLine($"  [Producer] Yielding reading {i}");
            yield return i * 10;
        }
    }

    static async Task Main()
    {
        Console.WriteLine("Starting stream...\n");

        // Consumer: processes each item as it arrives
        await foreach (var reading in GetSensorReadingsAsync())
        {
            Console.WriteLine($"[Consumer] Received: {reading}");
        }

        Console.WriteLine("\nStream complete.");
    }
}
```

### 🖨️ Output
```
Starting stream...

  [Producer] Yielding reading 1
[Consumer] Received: 10
  [Producer] Yielding reading 2
[Consumer] Received: 20
  [Producer] Yielding reading 3
[Consumer] Received: 30
  ...
Stream complete.
```

Notice: items are consumed **one by one**, interleaved with production.

---

## 🔍 Real-World Example — Streaming Database Results

```csharp
// Instead of loading ALL rows into memory:
// List<Order> allOrders = await db.Orders.ToListAsync(); // ❌ loads everything

// Stream them one at a time:
await foreach (var order in db.Orders.AsAsyncEnumerable()) // ✅ streams
{
    await ProcessOrderAsync(order);
}
```

---

## 🆚 IAsyncEnumerable vs IEnumerable

| Feature | `IEnumerable<T>` | `IAsyncEnumerable<T>` |
|---|---|---|
| Data source | In-memory / sync | Network, DB, file, async I/O |
| Blocking? | ✅ Blocks thread during data fetch | ❌ Non-blocking with `await` |
| Memory | All items must exist first | One item at a time |
| Iteration | `foreach` | `await foreach` |
| Use case | Lists, arrays, LINQ | API paging, DB streams, live feeds |

---

## ❌ With Cancellation Support

```csharp
static async IAsyncEnumerable<int> GetDataAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Usage:
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
await foreach (var item in GetDataAsync().WithCancellation(cts.Token))
{
    Console.WriteLine(item);
}
```

---

## 📌 Summary

> `IAsyncEnumerable<T>` is the foundation of **async streaming** in C#. It lets you process data as a flow rather than a batch — saving memory, reducing latency, and keeping your application responsive while data arrives asynchronously.
