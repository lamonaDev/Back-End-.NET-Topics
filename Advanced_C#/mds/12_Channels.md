# System.Threading.Channels in C#

## Table of Contents
1. [Understanding Channels](#understanding-channels)
2. [Channel Types](#channel-types)
3. [Channel Creation](#channel-creation)
4. [Producer Patterns](#producer-patterns)
5. [Consumer Patterns](#consumer-patterns)
6. [Backpressure Strategies](#backpressure-strategies)
7. [Real-World Use Cases](#real-world-use-cases)
8. [Memory and Performance](#memory-and-performance)
9. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Channels

### What are Channels?

Channels provide a **typed, async-capable producer-consumer queue** for communication between threads or async operations. They offer built-in backpressure, completion signaling, and both bounded and unbounded capacity options.

### Channel vs Other Collections

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANNEL VS OTHER QUEUES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ConcurrentQueue:                                              │
│   ├─ Thread-safe but sync-only                                 │
│   ├─ No backpressure control                                    │
│   ├─ No async wait support                                     │
│   └─ Manual completion tracking                                │
│                                                                  │
│   BlockingCollection:                                           │
│   ├─ Thread-safe, blocking operations                          │
│   ├─ Supports bounded capacity                                  │
│   ├─ But blocking (not async-friendly)                         │
│   └─ Complex completion handling                               │
│                                                                  │
│   Channel:                                                      │
│   ├─ Full async/await support                                  │
│   ├─ Built-in backpressure (bounded)                           │
│   ├─ Completion signaling                                       │
│   ├─ Multiple readers/writers                                    │
│   ├─ Configurable waiting strategies                           │
│   └─ Zero-copy when possible                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Concept

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANNEL ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                           │
│                    │     Channel     │                           │
│                    │   ┌─────────┐   │                           │
│   Producer ───────>│   │ Item 1  │   │───────> Consumer         │
│        │            │   │ Item 2  │   │         │                 │
│        │            │   │ Item 3  │   │         │                 │
│        │ WriteAsync │   └─────────┘   │    ReadAsync              │
│        │            │                 │         │                 │
│        ▼            │  Completion     │         ▼                 │
│   ┌─────────┐       │  Source         │    ┌─────────┐              │
│   │         │──────>│                 │<───│         │              │
│   └─────────┘       └─────────────────┘    └─────────┘              │
│                                                                  │
│   Features:                                                      │
│   ├─ Async read/write                                          │
│   ├─ Backpressure on full                                        │
│   ├─ Completion signaling                                       │
│   └─ Multiple producers/consumers                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Code Example

```csharp
using System.Threading.Channels;

class ChannelDemo
{
    static async Task Main()
    {
        // Create an unbounded channel
        var channel = Channel.CreateUnbounded<string>();

        // Producer
        _ = Task.Run(async () =>
        {
            for (int i = 0; i < 10; i++)
            {
                await channel.Writer.WriteAsync($"Message {i}");
                Console.WriteLine($"Produced: {i}");
                await Task.Delay(100);
            }
            channel.Writer.Complete();
        });

        // Consumer
        await foreach (var message in channel.Reader.ReadAllAsync())
        {
            Console.WriteLine($"Consumed: {message}");
            await Task.Delay(200); // Simulate processing
        }

        Console.WriteLine("Channel complete!");
    }
}

// Output:
// Produced: 0
// Consumed: Message 0
// Produced: 1
// Produced: 2
// Consumed: Message 1
// ...
```

---

## Channel Types

### Unbounded Channel

```csharp
// Unbounded: Grows as needed (careful with memory!)
var unbounded = Channel.CreateUnbounded<int>();

// Options for unbounded
var options = new UnboundedChannelOptions
{
    SingleReader = false,  // Multiple consumers OK
    SingleWriter = false   // Multiple producers OK
};
var channel = Channel.CreateUnbounded<int>(options);
```

### Bounded Channel

```csharp
// Bounded: Fixed capacity with backpressure
var bounded = Channel.CreateBounded<int>(100); // Max 100 items

// With options
var boundedOptions = new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,     // Block when full (default)
    SingleReader = false,
    SingleWriter = false
};
var channel = Channel.CreateBounded<int>(boundedOptions);
```

### FullMode Options

```csharp
public enum BoundedChannelFullMode
{
    Wait,           // Block until space available
    DropNewest,     // Remove newest item, add new
    DropOldest,     // Remove oldest item, add new
    DropWrite       // Discard the write (return false from TryWrite)
}

// Examples
var dropOldest = Channel.CreateBounded<int>(
    new BoundedChannelOptions(10) { FullMode = BoundedChannelFullMode.DropOldest });

var dropWrite = Channel.CreateBounded<int>(
    new BoundedChannelOptions(10) { FullMode = BoundedChannelFullMode.DropWrite });
```

---

## Channel Creation

### Factory Methods

```csharp
// Unbounded channels
Channel<T> CreateUnbounded<T>();
Channel<T> CreateUnbounded<T>(UnboundedChannelOptions options);

// Bounded channels
Channel<T> CreateBounded<T>(int capacity);
Channel<T> CreateBounded<T>(BoundedChannelOptions options);

// Reader/Writer from existing channel
ChannelReader<T> channel.Reader;
ChannelWriter<T> channel.Writer;
```

### Channel Properties

```csharp
public class Channel<T>
{
    // Read from channel
    public ChannelReader<T> Reader { get; }
    
    // Write to channel  
    public ChannelWriter<T> Writer { get; }
}

public abstract class ChannelWriter<T>
{
    // Write operations
    public abstract bool TryWrite(T item);
    public virtual ValueTask WriteAsync(T item, CancellationToken ct = default);
    
    // Completion
    public virtual void Complete(Exception? error = null);
    public virtual bool TryComplete(Exception? error = null);
    
    // Wait for space (bounded)
    public virtual ValueTask<bool> WaitToWriteAsync(CancellationToken ct = default);
}

public abstract class ChannelReader<T>
{
    // Read operations
    public abstract bool TryRead(out T item);
    public virtual ValueTask<T> ReadAsync(CancellationToken ct = default);
    
    // Wait for items
    public virtual ValueTask<bool> WaitToReadAsync(CancellationToken ct = default);
    
    // Completion
    public virtual bool TryReadAllAsync(...);
    public virtual IAsyncEnumerable<T> ReadAllAsync(CancellationToken ct = default);
}
```

---

## Producer Patterns

### Basic Producer

```csharp
public async Task ProduceAsync(ChannelWriter<WorkItem> writer, IEnumerable<WorkItem> items)
{
    try
    {
        foreach (var item in items)
        {
            await writer.WriteAsync(item);
        }
    }
    catch (ChannelClosedException)
    {
        // Channel was closed while writing
        Console.WriteLine("Channel closed during production");
    }
    finally
    {
        // Signal completion
        writer.Complete();
    }
}
```

### Bounded Channel with Backpressure

```csharp
public async Task ProduceWithBackpressureAsync(
    ChannelWriter<byte[]> writer, 
    Stream source,
    CancellationToken ct = default)
{
    const int bufferSize = 4096;
    var buffer = new byte[bufferSize];
    int bytesRead;

    try
    {
        while ((bytesRead = await source.ReadAsync(buffer.AsMemory(0, bufferSize), ct)) > 0)
        {
            // For bounded channels, WaitToWriteAsync respects backpressure
            if (await writer.WaitToWriteAsync(ct))
            {
                // Copy buffer to avoid overwrite
                var data = buffer.AsMemory(0, bytesRead).ToArray();
                await writer.WriteAsync(data, ct);
            }
        }
    }
    catch (OperationCanceledException)
    {
        // Handle cancellation
    }
    finally
    {
        writer.Complete();
    }
}
```

### Multiple Producers

```csharp
public class MultiProducerChannel<T>
{
    private readonly Channel<T> _channel;
    private readonly List<Task> _producerTasks = new();

    public MultiProducerChannel(int capacity = 100)
    {
        _channel = Channel.CreateBounded<T>(capacity);
    }

    public void AddProducer(Func<ChannelWriter<T>, CancellationToken, Task> producer)
    {
        _producerTasks.Add(Task.Run(() => producer(_channel.Writer, default)));
    }

    public async Task CloseWhenAllCompleteAsync()
    {
        await Task.WhenAll(_producerTasks);
        _channel.Writer.Complete();
    }

    public ChannelReader<T> Reader => _channel.Reader;
}

// Usage
var multiProducer = new MultiProducerChannel<LogEntry>();
multiProducer.AddProducer(async (writer, ct) => { /* produce from source 1 */ });
multiProducer.AddProducer(async (writer, ct) => { /* produce from source 2 */ });
multiProducer.AddProducer(async (writer, ct) => { /* produce from source 3 */ });

_ = Task.Run(() => multiProducer.CloseWhenAllCompleteAsync());

await foreach (var log in multiProducer.Reader.ReadAllAsync())
{
    ProcessLog(log);
}
```

---

## Consumer Patterns

### Basic Consumer

```csharp
public async Task ConsumeAsync(ChannelReader<WorkItem> reader)
{
    await foreach (var item in reader.ReadAllAsync())
    {
        await ProcessItemAsync(item);
    }
    Console.WriteLine("Consumer finished - channel complete");
}
```

### Multiple Consumers

```csharp
public async Task ConsumeWithWorkersAsync(
    ChannelReader<WorkItem> reader, 
    int workerCount,
    CancellationToken ct = default)
{
    var workers = new Task[workerCount];
    
    for (int i = 0; i < workerCount; i++)
    {
        int workerId = i;
        workers[i] = Task.Run(async () =>
        {
            await foreach (var item in reader.ReadAllAsync(ct))
            {
                Console.WriteLine($"Worker {workerId} processing: {item.Id}");
                await ProcessItemAsync(item, ct);
            }
        }, ct);
    }
    
    await Task.WhenAll(workers);
}
```

### Select/Poll Pattern

```csharp
public async Task ConsumeMultipleChannelsAsync(
    ChannelReader<Command> commandChannel,
    ChannelReader<Event> eventChannel,
    CancellationToken ct = default)
{
    while (!ct.IsCancellationRequested)
    {
        // Try to read from either channel
        var commandTask = commandChannel.WaitToReadAsync(ct).AsTask();
        var eventTask = eventChannel.WaitToReadAsync(ct).AsTask();
        
        var completed = await Task.WhenAny(commandTask, eventTask);
        
        if (completed == commandTask && commandTask.Result)
        {
            if (commandChannel.TryRead(out var command))
            {
                await HandleCommandAsync(command, ct);
            }
        }
        else if (completed == eventTask && eventTask.Result)
        {
            if (eventChannel.TryRead(out var evt))
            {
                await HandleEventAsync(evt, ct);
            }
        }
    }
}
```

---

## Backpressure Strategies

### Understanding Backpressure

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKPRESSURE MECHANISMS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Bounded Channel (FullMode.Wait):                              │
│   ┌──────────────┐              ┌────────────────┐               │
│   │  Producer    │              │ Channel (Full) │               │
│   │  WriteAsync  │──────BLOCK──>│ [A][B][C]...[Z]│               │
│   │  (waiting)   │              └────────────────┘               │
│   └──────────────┘                                               │
│                                                                  │
│   Producer blocks until consumer makes space                   │
│                                                                  │
│   ─────────────────────────────────────────────────────────────  │
│                                                                  │
│   Bounded Channel (FullMode.DropOldest):                         │
│   ┌──────────────┐              ┌────────────────┐               │
│   │  Producer    │              │ Channel        │               │
│   │  WriteAsync  │──────WRITE──>│ [B][C]...[Z][NEW]│            │
│   └──────────────┘              └────────────────┘               │
│                                         ↑                        │
│                                    Oldest dropped (A)           │
│                                                                  │
│   Producer never blocks, old data may be lost                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Choosing a Strategy

```csharp
// Strategy 1: Block producer (default)
// Use when: All data is important, memory is limited
var blocking = Channel.CreateBounded<WorkItem>(
    new BoundedChannelOptions(100)
    {
        FullMode = BoundedChannelFullMode.Wait
    });

// Strategy 2: Drop oldest
// Use when: Newest data matters more (e.g., stock prices)
var dropOldest = Channel.CreateBounded<MarketData>(
    new BoundedChannelOptions(1000)
    {
        FullMode = BoundedChannelFullMode.DropOldest
    });

// Strategy 3: Drop newest
// Use when: Oldest data must be processed first
var dropNewest = Channel.CreateBounded<WorkItem>(
    new BoundedChannelOptions(100)
    {
        FullMode = BoundedChannelFullMode.DropNewest
    });

// Strategy 4: TryWrite (non-blocking)
// Use when: Optional processing, don't want to block
var optional = Channel.CreateBounded<LogEntry>(
    new BoundedChannelOptions(10000)
    {
        FullMode = BoundedChannelFullMode.DropWrite
    });

// Usage with TryWrite
if (writer.TryWrite(logEntry))
{
    Console.WriteLine("Log queued");
}
else
{
    Console.WriteLine("Log dropped (buffer full)");
}
```

---

## Real-World Use Cases

### Use Case 1: Log Aggregation

```csharp
public class AsyncLogger : IDisposable
{
    private readonly Channel<LogEntry> _channel;
    private readonly Task _processingTask;

    public AsyncLogger(int capacity = 10000)
    {
        _channel = Channel.CreateBounded<LogEntry>(
            new BoundedChannelOptions(capacity)
            {
                FullMode = BoundedChannelFullMode.DropWrite
            });

        _processingTask = Task.Run(ProcessLogsAsync);
    }

    public void Log(string message, LogLevel level)
    {
        // Fire-and-forget
        _channel.Writer.TryWrite(new LogEntry
        {
            Timestamp = DateTime.UtcNow,
            Level = level,
            Message = message
        });
    }

    private async Task ProcessLogsAsync()
    {
        await foreach (var entry in _channel.Reader.ReadAllAsync())
        {
            await WriteToFileAsync(entry);
            
            if (entry.Level == LogLevel.Error)
            {
                await SendAlertAsync(entry);
            }
        }
    }

    public void Dispose()
    {
        _channel.Writer.Complete();
        _processingTask.Wait(TimeSpan.FromSeconds(5));
    }

    private async Task WriteToFileAsync(LogEntry entry) { }
    private async Task SendAlertAsync(LogEntry entry) { }
}
```

### Use Case 2: Producer-Consumer Pipeline

```csharp
public class DataPipeline
{
    private readonly Channel<RawData> _inputChannel;
    private readonly Channel<ProcessedData> _outputChannel;

    public DataPipeline()
    {
        _inputChannel = Channel.CreateBounded<RawData>(100);
        _outputChannel = Channel.CreateBounded<ProcessedData>(100);
    }

    public ChannelWriter<RawData> Input => _inputChannel.Writer;
    public ChannelReader<ProcessedData> Output => _outputChannel.Reader;

    public async Task RunAsync(int workerCount, CancellationToken ct = default)
    {
        var workers = new Task[workerCount];
        
        for (int i = 0; i < workerCount; i++)
        {
            workers[i] = Task.Run(() => WorkerLoopAsync(ct), ct);
        }
        
        await Task.WhenAll(workers);
    }

    private async Task WorkerLoopAsync(CancellationToken ct)
    {
        await foreach (var rawData in _inputChannel.Reader.ReadAllAsync(ct))
        {
            var processed = await ProcessDataAsync(rawData, ct);
            await _outputChannel.Writer.WriteAsync(processed, ct);
        }
    }

    private async Task<ProcessedData> ProcessDataAsync(RawData data, CancellationToken ct)
    {
        await Task.CompletedTask;
        return new ProcessedData { /* ... */ };
    }
}
```

### Use Case 3: Rate-Limited Processing

```csharp
public class RateLimitedProcessor<T>
{
    private readonly Channel<T> _channel;
    private readonly Func<T, CancellationToken, Task> _processor;
    private readonly int _maxConcurrent;
    private readonly TimeSpan _minInterval;

    public RateLimitedProcessor(
        Func<T, CancellationToken, Task> processor,
        int maxConcurrent,
        TimeSpan minInterval,
        int capacity = 1000)
    {
        _processor = processor;
        _maxConcurrent = maxConcurrent;
        _minInterval = minInterval;
        
        _channel = Channel.CreateBounded<T>(
            new BoundedChannelOptions(capacity)
            {
                FullMode = BoundedChannelFullMode.Wait
            });
    }

    public ValueTask EnqueueAsync(T item, CancellationToken ct = default)
    {
        return _channel.Writer.WriteAsync(item, ct);
    }

    public async Task RunAsync(CancellationToken ct = default)
    {
        var semaphore = new SemaphoreSlim(_maxConcurrent);
        var lastProcessTime = DateTime.MinValue;
        var lockObj = new object();

        await foreach (var item in _channel.Reader.ReadAllAsync(ct))
        {
            await semaphore.WaitAsync(ct);
            
            _ = Task.Run(async () =>
            {
                try
                {
                    // Enforce minimum interval
                    lock (lockObj)
                    {
                        var elapsed = DateTime.UtcNow - lastProcessTime;
                        if (elapsed < _minInterval)
                        {
                            Thread.Sleep(_minInterval - elapsed);
                        }
                        lastProcessTime = DateTime.UtcNow;
                    }

                    await _processor(item, ct);
                }
                finally
                {
                    semaphore.Release();
                }
            }, ct);
        }
    }

    public void Complete() => _channel.Writer.Complete();
}
```

---

## Memory and Performance

### Channel Memory Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANNEL MEMORY LAYOUT                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Unbounded Channel:                                            │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Linked list of segments                                  │
│   │  ┌─────────┐    ┌─────────┐    ┌─────────┐               │    │
│   │  │ Slot 0  │───>│ Slot 0  │───>│ Slot 0  │───> ...        │    │
│   │  │ Slot 1  │    │ Slot 1  │    │ Slot 1  │                │    │
│   │  │ ...     │    │ ...     │    │ ...     │                │    │
│   │  └─────────┘    └─────────┘    └─────────┘               │    │
│   │    Segment 1       Segment 2       Segment 3              │    │
│   └─────────────────────────────────────────────────────────┘    │
│   │                                                                  │
│   │   Grows dynamically, segments added as needed                   │
│   │   Each segment: ~1KB overhead + item storage                     │
│   │                                                                  │
│   Bounded Channel:                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Fixed-size circular buffer                               │    │
│   │  ┌──────────────────────────────────────────┐            │    │
│   │  │ [0] [1] [2] [3] ... [capacity-1]         │            │    │
│   │  │  ↑                       ↑               │            │    │
│   │  │ writeIndex          readIndex           │            │    │
│   │  └──────────────────────────────────────────┘            │    │
│   └─────────────────────────────────────────────────────────┘    │
│   │                                                                  │
│   │   Pre-allocated, no growth overhead                            │
│   │   ~itemSize * capacity memory                                   │
│   │                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Performance Characteristics

| Operation | Unbounded | Bounded |
|-----------|-----------|---------|
| Write | O(1) amortized | O(1) or block |
| Read | O(1) | O(1) or block |
| Memory | Grows with items | Fixed |
| Throughput | Very high | Very high |
| Latency | Low | May block |

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Always complete the writer
producerTask.ContinueWith(_ => channel.Writer.Complete());

// 2. Use CancellationToken
await channel.Reader.ReadAsync(ct);

// 3. Handle ChannelClosedException
try
{
    await channel.Writer.WriteAsync(item, ct);
}
catch (ChannelClosedException)
{
    // Channel is complete
}

// 4. Choose appropriate capacity
// Bounded for backpressure, unbounded for flexibility

// 5. Use await foreach for consumers
await foreach (var item in channel.Reader.ReadAllAsync(ct))
{
    // Clean consumption
}

// 6. Dispose resources in pipeline
await using var stream = ...;
await channel.Writer.WriteAsync(data);
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Not completing the writer
// Consumer will wait forever for more items!
var channel = Channel.CreateUnbounded<int>();
// ... produce items ...
// ❌ Forgot: channel.Writer.Complete();

// PITFALL 2: Blocking on async operations
channel.Writer.WriteAsync(item).AsTask().Wait(); // ❌ Deadlock risk!

// PITFALL 3: Unbounded channel without limits
var channel = Channel.CreateUnbounded<byte[]>();
// Can consume all memory if producer is faster!

// PITFALL 4: Not handling cancellation
await foreach (var item in channel.Reader.ReadAllAsync())
// No cancellation - can't stop cleanly

// PITFALL 5: Multiple completes
channel.Writer.Complete();
channel.Writer.Complete(); // Throws exception

// PITFALL 6: Reading after complete without checking
if (channel.Reader.TryRead(out var item))
// Should check return value - may be false if empty
```

---

## Interview Questions

**Q: What is System.Threading.Channels?**> Channels provide a typed, async-capable producer-consumer queue with built-in backpressure and completion signaling. They're designed for high-performance communication between async components.

**Q: When should you use a bounded vs unbounded channel?**> Use bounded when memory is constrained or you need backpressure. Use unbounded when memory isn't a concern and you don't want to block producers. Bounded channels prevent memory exhaustion under load.

**Q: How does backpressure work in bounded channels?**> When a bounded channel is full, WriteAsync can either: (1) Wait for space (Wait mode), (2) Drop oldest/newest item, or (3) Return false (TryWrite). This prevents producers from overwhelming consumers.

**Q: What's the difference between Channels and BlockingCollection?**> BlockingCollection uses blocking operations (threads wait). Channels use async operations (threads are released). Channels are designed for modern async/await patterns and integrate better with Task-based code.

**Q: How do you signal completion in a channel?**> Call channel.Writer.Complete(). This causes Reader.ReadAsync to throw ChannelClosedException or makes ReadAllAsync complete normally after yielding remaining items.

**Q: Can multiple threads write to or read from a channel?**> Yes, channels are thread-safe for multiple producers and consumers. Use SingleReader/SingleWriter options only when you're certain only one thread will access for potential optimizations.
