# System.Threading.Channels in C#

## 🧠 What Is It?

`System.Threading.Channels` provides a **thread-safe, async-first pipeline** for passing data between producers and consumers. It's the modern, high-performance alternative to `ConcurrentQueue` when you need async communication between threads or tasks.

Think of it as a **message pipe**: one side writes data in, the other side reads it out — safely, efficiently, and without blocking.

---

## 🌍 Real-World Analogy

Imagine an **order management system** at a warehouse:
- **Producers**: Customer service reps place orders into a queue.
- **Channel**: The order pipeline (with a configurable backlog limit).
- **Consumers**: Warehouse workers pick up and fulfill orders one at a time.

Neither side knows about the other — they just talk through the channel. The channel handles all the thread-safety and backpressure automatically.

---

## ⚙️ Architecture & Memory Model

```
Producer(s)                 Channel                  Consumer(s)
──────────────              ───────────              ──────────────
  Task A ──► WriteAsync ──► [item1]                  ReadAsync ──► Task X
  Task B ──► WriteAsync ──► [item2]   internal       ReadAsync ──► Task Y
  Task C ──► WriteAsync ──► [item3]   queue          ReadAsync ──► Task Z
                            [item4]
                         (bounded or unbounded)
```

> Internally, `Channel<T>` uses a **lock-free queue** backed by linked segments or arrays. `WriteAsync` and `ReadAsync` use **async continuations** — no thread blocks while waiting. This makes Channels extremely efficient under high throughput.

---

## 💻 Basic Producer-Consumer Example

```csharp
using System;
using System.Threading.Channels;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        // Create an unbounded channel
        var channel = Channel.CreateUnbounded<string>();

        // Producer task
        var producer = Task.Run(async () =>
        {
            string[] orders = { "Order-001", "Order-002", "Order-003", "Order-004" };

            foreach (var order in orders)
            {
                Console.WriteLine($"[Producer] Writing: {order}");
                await channel.Writer.WriteAsync(order);
                await Task.Delay(300); // Simulate order arriving over time
            }

            channel.Writer.Complete(); // Signal: no more items coming
            Console.WriteLine("[Producer] Done writing.");
        });

        // Consumer task
        var consumer = Task.Run(async () =>
        {
            // ReadAllAsync streams items as they arrive
            await foreach (var order in channel.Reader.ReadAllAsync())
            {
                Console.WriteLine($"[Consumer] Processing: {order}");
                await Task.Delay(100); // Simulate processing
            }
            Console.WriteLine("[Consumer] Done reading.");
        });

        await Task.WhenAll(producer, consumer);
    }
}
```

### 🖨️ Output
```
[Producer] Writing: Order-001
[Consumer] Processing: Order-001
[Producer] Writing: Order-002
[Consumer] Processing: Order-002
...
[Producer] Done writing.
[Consumer] Done reading.
```

---

## ⚙️ Bounded vs Unbounded Channels

### Unbounded Channel — grows without limit
```csharp
var channel = Channel.CreateUnbounded<int>();
// Producer can always write; memory grows if consumer is slow
```

### Bounded Channel — backpressure control
```csharp
var options = new BoundedChannelOptions(capacity: 5)
{
    FullMode = BoundedChannelFullMode.Wait // Producer waits if full
    // Other modes: DropOldest, DropNewest, DropWrite
};
var channel = Channel.CreateBounded<int>(options);
```

```
Bounded channel (capacity=3):
[item1][item2][item3]  ← FULL
                        Producer.WriteAsync() → AWAITS here (backpressure)
                                  ↓ consumer reads item1
[item2][item3][new]    ← space opens, producer resumes
```

---

## 💻 Multiple Producers, Multiple Consumers

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(10)
{
    FullMode = BoundedChannelFullMode.Wait
});

// 3 producers
var producers = Enumerable.Range(1, 3).Select(id => Task.Run(async () =>
{
    for (int i = 0; i < 5; i++)
    {
        await channel.Writer.WriteAsync(id * 100 + i);
    }
})).ToArray();

// 2 consumers
var consumers = Enumerable.Range(1, 2).Select(id => Task.Run(async () =>
{
    await foreach (var item in channel.Reader.ReadAllAsync())
    {
        Console.WriteLine($"Consumer {id} got: {item}");
    }
})).ToArray();

await Task.WhenAll(producers);
channel.Writer.Complete(); // Signal all producers are done
await Task.WhenAll(consumers);
```

---

## 🆚 ConcurrentQueue vs Channel

| Feature | `ConcurrentQueue<T>` | `Channel<T>` |
|---|---|---|
| Async wait for data | ❌ Must poll/spin | ✅ `ReadAllAsync()` |
| Backpressure (bounded) | ❌ | ✅ `BoundedChannel` |
| Completion signal | ❌ | ✅ `Writer.Complete()` |
| Integration with `await foreach` | ❌ | ✅ |
| Multiple producers/consumers | ✅ | ✅ |
| Performance | Fast | Fast + async-native |

---

## 📌 Summary

> `System.Threading.Channels` is the **modern producer-consumer primitive** in C#. It supports async reading/writing, backpressure via bounded capacity, completion signaling, and multiple producers/consumers — all without blocking threads. Prefer it over `ConcurrentQueue` whenever you need async communication between tasks.
