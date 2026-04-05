# IAsyncEnumerable in C#

## Table of Contents
1. [Understanding IAsyncEnumerable](#understanding-iasyncenumerable)
2. [Comparison with IEnumerable](#comparison-with-ienumerable)
3. [Creating Async Streams](#creating-async-streams)
4. [Consuming Async Streams](#consuming-async-streams)
5. [Real-World Use Cases](#real-world-use-cases)
6. [Memory and Performance](#memory-and-performance)
7. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding IAsyncEnumerable

### What is IAsyncEnumerable?

`IAsyncEnumerable<T>` is a **streaming interface** that enables asynchronous iteration over data that arrives gradually. Unlike `IEnumerable<T>` which requires all data to be available before iteration begins, `IAsyncEnumerable` can yield items as they become available.

### The Problem It Solves

```csharp
// ❌ PROBLEM: IEnumerable requires ALL data in memory
public IEnumerable<LogEntry> GetLogs(DateTime from, DateTime to)
{
    var allLogs = database.LoadAllLogs(from, to); // Memory intensive!
    foreach (var log in allLogs)
    {
        yield return log;
    }
}

// ❌ PROBLEM: Task<IEnumerable<T>> waits for ALL data
public async Task<IEnumerable<LogEntry>> GetLogsAsync(DateTime from, DateTime to)
{
    var allLogs = await database.LoadAllLogsAsync(from, to);
    return allLogs; // Nothing until ALL loaded
}

// ✅ SOLUTION: IAsyncEnumerable streams data as available
public async IAsyncEnumerable<LogEntry> GetLogsAsync(
    DateTime from, 
    DateTime to,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var log in database.StreamLogsAsync(from, to, ct))
    {
        yield return log; // Item yielded immediately!
    }
}
```

### Visual Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    IEnumerable<T> (Synchronous)                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [Data Source]                                                 │
│        │                                                        │
│        ▼                                                        │
│   ┌───────────────────────────────────────┐                    │
│   │  Load ALL data into memory            │                    │
│   │  ┌────┬────┬────┬────┬────┬────┐      │                    │
│   │  │ A  │ B  │ C  │ D  │ E  │ F  │      │                    │
│   │  └────┴────┴────┴────┴────┴────┘      │                    │
│   └───────────────────────────────────────┘                    │
│        │                                                        │
│        ▼                                                        │
│   [Consumer] ← Gets all items at once                          │
│                                                                  │
│   Memory: O(n) where n = total items                          │
│   Time: Wait for ALL before processing                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    IAsyncEnumerable<T> (Asynchronous Streaming)  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [Data Source]                                                 │
│        │                                                        │
│        │ Stream as ready:                                       │
│        ├───[A]─────[B]─────[C]─────[D]─────[E]─────[F]───      │
│        │    │      │      │      │      │      │               │
│   [Consumer]   ← Process A while loading B                      │
│        ▼                                                        │
│   Process each item IMMEDIATELY when available                  │
│                                                                  │
│   Memory: O(1) - only current item in memory                     │
│   Time: Start processing immediately                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Code Example

```csharp
using System.Runtime.CompilerServices;

class AsyncStreamingDemo
{
    // Producer: async enumerable method
    public static async IAsyncEnumerable<int> GetNumbersAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        for (int i = 0; i < 5; i++)
        {
            await Task.Delay(1000, ct); // Simulate async work
            Console.WriteLine($"Yielding: {i}");
            yield return i;
        }
    }

    static async Task Main()
    {
        Console.WriteLine("Starting async stream...\n");
        
        // Consumer: await foreach
        await foreach (var number in GetNumbersAsync())
        {
            Console.WriteLine($"Received: {number}");
        }
        
        Console.WriteLine("\nStream complete!");
    }
}

// Output:
// Starting async stream...
//
// Yielding: 0
// Received: 0
// Yielding: 1
// Received: 1
// ...
```

---

## Comparison with IEnumerable

### Feature Comparison

| Feature | `IEnumerable<T>` | `IAsyncEnumerable<T>` |
|---------|------------------|------------------------|
| Iterator keyword | `yield return` | `yield return` (in async method) |
| Consumption | `foreach` | `await foreach` |
| Can await inside? | ❌ No | ✅ Yes |
| Thread blocking? | Yes | No (async) |
| Memory efficiency | Low (all data) | High (streaming) |
| Cancellation support | Manual | Built-in with `EnumeratorCancellation` |
| LINQ support | Full | Async LINQ extensions |

### Iterator Method Differences

```csharp
// Synchronous iterator
public IEnumerable<int> GetSyncNumbers()
{
    for (int i = 0; i < 5; i++)
    {
        Thread.Sleep(100); // Blocks thread
        yield return i;
    }
}

// Asynchronous iterator
public async IAsyncEnumerable<int> GetAsyncNumbers(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 5; i++)
    {
        await Task.Delay(100, ct); // Doesn't block
        yield return i;
    }
}

// Usage difference
foreach (var n in GetSyncNumbers()) { } // Synchronous
await foreach (var n in GetAsyncNumbers()) { } // Asynchronous
```

---

## Creating Async Streams

### Basic Async Iterator

```csharp
public class AsyncDataProvider
{
    // The [EnumeratorCancellation] attribute passes
    // the CancellationToken from await foreach to the method
    public async IAsyncEnumerable<DataRecord> StreamDataAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        int page = 0;
        while (true)
        {
            var batch = await FetchPageAsync(page, ct);
            
            if (batch.Count == 0)
                yield break; // End the stream
            
            foreach (var record in batch)
            {
                yield return record; // Stream each item
            }
            
            page++;
        }
    }

    private async Task<List<DataRecord>> FetchPageAsync(
        int page, 
        CancellationToken ct)
    {
        // Simulate async database/API call
        await Task.Delay(100, ct);
        return new List<DataRecord> { /* ... */ };
    }
}
```

### Streaming from Database

```csharp
public class LogRepository
{
    public async IAsyncEnumerable<LogEntry> StreamLogsAsync(
        DateTime from,
        DateTime to,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await using var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync(ct);
        
        await using var command = new SqlCommand(
            "SELECT * FROM Logs WHERE Timestamp BETWEEN @from AND @to ORDER BY Timestamp",
            connection);
        
        command.Parameters.AddWithValue("@from", from);
        command.Parameters.AddWithValue("@to", to);
        
        await using var reader = await command.ExecuteReaderAsync(ct);
        
        while (await reader.ReadAsync(ct))
        {
            yield return new LogEntry
            {
                Id = reader.GetInt64(0),
                Timestamp = reader.GetDateTime(1),
                Level = reader.GetString(2),
                Message = reader.GetString(3)
            };
        }
    }
}

// Usage
await foreach (var log in repository.StreamLogsAsync(yesterday, now, ct))
{
    ProcessLog(log); // Process as each row arrives
}
```

### Streaming HTTP Responses

```csharp
public class HttpStreamClient
{
    private readonly HttpClient _httpClient;

    public async IAsyncEnumerable<string> StreamLinesAsync(
        string url,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        using var response = await _httpClient.GetAsync(
            url, 
            HttpCompletionOption.ResponseHeadersRead, 
            ct);
        
        response.EnsureSuccessStatusCode();
        
        await using var stream = await response.Content.ReadAsStreamAsync(ct);
        using var reader = new StreamReader(stream);
        
        string? line;
        while ((line = await reader.ReadLineAsync(ct)) != null)
        {
            yield return line;
        }
    }
}
```

---

## Consuming Async Streams

### await foreach

```csharp
// Basic consumption
await foreach (var item in GetAsyncData())
{
    Console.WriteLine(item);
}

// With cancellation
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

await foreach (var item in GetAsyncData().WithCancellation(cts.Token))
{
    if (IsTargetItem(item))
    {
        cts.Cancel(); // Stop processing
        break;
    }
}

// With ConfigureAwait
await foreach (var item in GetAsyncData().ConfigureAwait(false))
{
    // Runs on ThreadPool thread
}
```

### Manual Enumeration

```csharp
// Sometimes you need more control
public async Task ProcessWithControl(IAsyncEnumerable<int> stream)
{
    await using var enumerator = stream.GetAsyncEnumerator();
    
    try
    {
        while (await enumerator.MoveNextAsync())
        {
            var current = enumerator.Current;
            
            if (ShouldSkip(current))
                continue;
            
            await ProcessAsync(current);
            
            if (ShouldStop(current))
                break;
        }
    }
    finally
    {
        // AsyncDisposable called automatically
    }
}
```

### Async LINQ Extensions

```csharp
// Requires System.Linq.Async package

// Filter
await foreach (var num in numbers.WhereAsync(n => IsEvenAsync(n)))
{
    // Only even numbers
}

// Transform
await foreach (var dto in users.SelectAsync(async u => await MapToDtoAsync(u)))
{
    // Transformed items
}

// Take with async predicate
await foreach (var item in stream.TakeWhileAwaitAsync(async x => await IsValidAsync(x)))
{
    // Items while predicate returns true
}
```

---

## Real-World Use Cases

### Use Case 1: Real-time Log Processing

```csharp
public class RealTimeLogProcessor
{
    public async IAsyncEnumerable<Alert> ProcessLogStreamAsync(
        IAsyncEnumerable<LogEntry> logStream,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var log in logStream.WithCancellation(ct))
        {
            if (IsErrorLog(log))
            {
                var alert = await CreateAlertAsync(log, ct);
                yield return alert; // Alert yielded immediately
            }
            
            // Continue processing without waiting for storage
        }
    }

    private bool IsErrorLog(LogEntry log) => log.Level == "Error";
    
    private async Task<Alert> CreateAlertAsync(LogEntry log, CancellationToken ct)
    {
        // Analyze and create alert
        await Task.CompletedTask;
        return new Alert { /* ... */ };
    }
}

// Usage: React to errors in real-time
var processor = new RealTimeLogProcessor();
await foreach (var alert in processor.ProcessLogStreamAsync(logStream, ct))
{
    await SendAlertNotificationAsync(alert); // Immediate notification!
}
```

### Use Case 2: Large File Processing

```csharp
public class CsvProcessor
{
    public async IAsyncEnumerable<Record> ParseLargeCsvAsync(
        string filePath,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await using var stream = File.OpenRead(filePath);
        using var reader = new StreamReader(stream);
        
        // Skip header
        await reader.ReadLineAsync(ct);
        
        string? line;
        while ((line = await reader.ReadLineAsync(ct)) != null)
        {
            ct.ThrowIfCancellationRequested();
            
            var record = ParseLine(line);
            if (record != null)
            {
                yield return record;
            }
        }
    }

    private Record? ParseLine(string line)
    {
        // Parse logic
        return new Record { /* ... */ };
    }
}

// Usage: Process 1GB file with minimal memory
var processor = new CsvProcessor();
int processedCount = 0;

await foreach (var record in processor.ParseLargeCsvAsync("huge-file.csv", ct))
{
    await SaveToDatabaseAsync(record);
    processedCount++;
    
    if (processedCount % 1000 == 0)
    {
        Console.WriteLine($"Processed {processedCount} records...");
    }
}
```

### Use Case 3: Paginated API Consumption

```csharp
public class PaginatedApiClient
{
    private readonly HttpClient _httpClient;

    public async IAsyncEnumerable<Product> GetAllProductsAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        int page = 1;
        const int pageSize = 100;
        
        while (true)
        {
            var response = await _httpClient.GetFromJsonAsync<ApiResponse>(
                $"/api/products?page={page}&size={pageSize}", 
                ct);
            
            if (response?.Products.Count == 0)
                yield break;
            
            foreach (var product in response!.Products)
            {
                yield return product;
            }
            
            if (!response.HasMorePages)
                yield break;
            
            page++;
        }
    }
}

// Usage: Stream all products from paginated API
var client = new PaginatedApiClient();
await foreach (var product in client.GetAllProductsAsync(ct))
{
    await CacheProductAsync(product); // Cache as they arrive
}
```

---

## Memory and Performance

### Memory Allocation

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY COMPARISON                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Task<List<T>> Pattern:                                        │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  List allocation                                        │    │
│   │  ├─ Array backing store (n * reference size)           │    │
│   │  ├─ Each object in list                                │    │
│   │  └─ Total: O(n) memory                                  │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   IAsyncEnumerable<T> Pattern:                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  State machine allocation (fixed size)                   │    │
│   │  ├─ One item at a time                                  │    │
│   │  ├─ Previous items eligible for GC                    │    │
│   │  └─ Total: O(1) memory (amortized)                      │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   For 1 million records:                                         │
│   ├─ List: ~8MB for references + object overhead              │
│   └─ AsyncEnumerable: ~constant memory                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Performance Considerations

```csharp
// ⚠️ Bad: Creating many small async operations
public async IAsyncEnumerable<int> InefficientStream()
{
    for (int i = 0; i < 1000000; i++)
    {
        await Task.Yield(); // Overhead per item!
        yield return i;
    }
}

// ✅ Better: Batch operations
public async IAsyncEnumerable<int> EfficientStream()
{
    var batch = new List<int>(100);
    
    for (int i = 0; i < 1000000; i++)
    {
        batch.Add(i);
        
        if (batch.Count >= 100)
        {
            // Process batch
            foreach (var item in batch)
            {
                yield return item;
            }
            
            batch.Clear();
            await Task.Yield(); // Yield less frequently
        }
    }
    
    // Remaining items
    foreach (var item in batch)
    {
        yield return item;
    }
}
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Always use [EnumeratorCancellation]
public async IAsyncEnumerable<T> GetDataAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    // CancellationToken automatically flows from await foreach
}

// 2. Properly dispose resources
public async IAsyncEnumerable<T> GetDataAsync(CancellationToken ct = default)
{
    await using var connection = new SqlConnection(connString);
    // ...
}

// 3. Use ConfigureAwait in libraries
public async IAsyncEnumerable<T> GetDataAsync(CancellationToken ct = default)
{
    await foreach (var item in source.WithCancellation(ct).ConfigureAwait(false))
    {
        yield return item;
    }
}

// 4. Handle exceptions gracefully
try
{
    await foreach (var item in stream.WithCancellation(ct))
    {
        // Process
    }
}
catch (OperationCanceledException)
{
    // Expected - log or ignore
}
catch (Exception ex)
{
    // Log unexpected errors
}
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Not using [EnumeratorCancellation]
public async IAsyncEnumerable<int> Bad(CancellationToken ct)
{
    // ct is unused by await foreach consumer!
    for (int i = 0; i < 10; i++)
    {
        await Task.Delay(100);
        yield return i;
    }
}

// ✅ Correct version
public async IAsyncEnumerable<int> Good(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 10; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}

// PITFALL 2: Mixing sync and async
public IAsyncEnumerable<int> Mixed()
{
    // ❌ Can't yield in async without async keyword
    yield return 1;
    await Task.Delay(100); // Error!
    yield return 2;
}

// PITFALL 3: Forgetting ConfigureAwait in UI apps
// May cause deadlocks
await foreach (var item in GetDataAsync()) // Potential deadlock

// ✅ Use ConfigureAwait
await foreach (var item in GetDataAsync().ConfigureAwait(false))

// PITFALL 4: Consuming without await
// ❌ This won't work - GetAsyncData returns IAsyncEnumerable
var items = GetAsyncData();
foreach (var item in items) { } // Error!

// ✅ Must use await foreach
await foreach (var item in GetAsyncData()) { }
```

---

## Interview Questions

**Q: What's the difference between `IAsyncEnumerable<T>` and `Task<IEnumerable<T>>`?**
> `Task<IEnumerable<T>>` waits for ALL items before returning any, while `IAsyncEnumerable<T>` streams items as they become available using `await foreach`.

**Q: What is the purpose of `[EnumeratorCancellation]` attribute?**
> It allows the `CancellationToken` passed to `await foreach` to flow into the async iterator method automatically.

**Q: Can you use LINQ with `IAsyncEnumerable<T>`?**
> Not directly with standard LINQ. You need the `System.Linq.Async` NuGet package which provides async LINQ extensions like `WhereAsync`, `SelectAsync`, etc.

**Q: When should you use `IAsyncEnumerable<T>` over `Task<IEnumerable<T>>`?**
> Use `IAsyncEnumerable<T>` when: (1) Data arrives gradually, (2) Memory efficiency matters, (3) You want to start processing before all data is loaded, (4) Working with large datasets.

**Q: How do you cancel an async stream?**
> Pass a `CancellationToken` to `await foreach` using `.WithCancellation(token)`. The token flows to the iterator via `[EnumeratorCancellation]` attribute.

**Q: What's the memory benefit of `IAsyncEnumerable<T>`?**
> Only one item is in memory at a time (O(1) space), compared to `Task<IEnumerable<T>>` which loads all items into memory (O(n) space).
