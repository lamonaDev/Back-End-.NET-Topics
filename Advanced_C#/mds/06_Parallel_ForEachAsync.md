# Parallel.ForEachAsync in C#

## Table of Contents
1. [Understanding Parallel.ForEachAsync](#understanding-parallelforeachasync)
2. [Comparison with Alternatives](#comparison-with-alternatives)
3. [Degree of Parallelism Control](#degree-of-parallelism-control)
4. [Real-World Use Cases](#real-world-use-cases)
5. [Memory and Execution Model](#memory-and-execution-model)
6. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Parallel.ForEachAsync

### What is Parallel.ForEachAsync?

`Parallel.ForEachAsync` (introduced in .NET 6) is a method that **processes a collection of items concurrently using async/await**, with built-in control over the **degree of parallelism** (how many operations run simultaneously).

### The Problem It Solves

```csharp
// ❌ PROBLEM: Sequential processing is slow
foreach (var url in urls)
{
    await DownloadAsync(url); // One at a time!
}
// Time: url1 + url2 + url3...

// ❌ PROBLEM: Task.WhenAll can overwhelm resources
var tasks = urls.Select(DownloadAsync);
await Task.WhenAll(tasks); // ALL start at once!
// Time: max(url1, url2...) but risks overwhelming server

// ✅ SOLUTION: Parallel.ForEachAsync with controlled concurrency
await Parallel.ForEachAsync(urls, async (url, cancellationToken) =>
{
    await DownloadAsync(url);
});
// Time: controlled parallelism - fast but safe
```

### Basic Code Example

```csharp
using System.Net.Http;

class ParallelDownloadDemo
{
    private static readonly HttpClient _httpClient = new();

    static async Task Main()
    {
        var urls = new[]
        {
            "https://example.com/file1.pdf",
            "https://example.com/file2.pdf",
            "https://example.com/file3.pdf",
            "https://example.com/file4.pdf",
            "https://example.com/file5.pdf",
            "https://example.com/file6.pdf",
        };

        Console.WriteLine("Starting parallel downloads...\n");
        var stopwatch = Stopwatch.StartNew();

        await Parallel.ForEachAsync(urls, async (url, cancellationToken) =>
        {
            try
            {
                var response = await _httpClient.GetAsync(url, cancellationToken);
                var size = response.Content.Headers.ContentLength ?? 0;
                Console.WriteLine($"[{Thread.CurrentThread.ManagedThreadId}] {url} -> {size} bytes");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[{Thread.CurrentThread.ManagedThreadId}] Error: {ex.Message}");
            }
        });

        stopwatch.Stop();
        Console.WriteLine($"\n✅ All downloads complete in {stopwatch.ElapsedMilliseconds}ms");
    }
}
```

---

## Comparison with Alternatives

### Visual Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEQUENTIAL FOREACH                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Items:  [A]─────────>[B]─────────>[C]─────────>[D]              │
│              ↓           ↓           ↓           ↓               │
│           Task 1      Task 2      Task 3      Task 4            │
│                                                                  │
│   Time:  T1 + T2 + T3 + T4 (slowest)                             │
│   Memory: Low                                                    │
│   CPU:   Low                                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    TASK.WHENALL                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Items:  [A]──┐                                                 │
│           [B]──┼──> ALL AT ONCE! ──>                          │
│           [C]──┤         (Unlimited)                          │
│           [D]──┘                                                 │
│                                                                  │
│   Time:  max(T1, T2, T3, T4) (fastest)                           │
│   Memory: Very High (all concurrent)                            │
│   CPU:   Very High                                                │
│   Risk:  Can overwhelm server/DB!                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    PARALLEL.FOREACHASYNC                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Items:  [A][B][C]──┐                                            │
│           (running)  │──> Next item picks up when              │
│                      │    a slot becomes available               │
│           [D][E][F]──┘    (Controlled concurrency)              │
│           (running)                                              │
│                                                                  │
│   Time:  Fast (parallel) but controlled                          │
│   Memory: Moderate (bounded)                                     │
│   CPU:   Moderate (configurable)                                 │
│   Risk:  Minimal - configurable limits                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Comparison Table

| Feature | `foreach` + `await` | `Task.WhenAll` | `Parallel.ForEachAsync` |
|---------|---------------------|----------------|-------------------------|
| Concurrency | None (sequential) | Unlimited | Configurable |
| Async Support | ✅ Yes | ✅ Yes | ✅ Yes |
| Memory Usage | Low | High (scales with items) | Moderate (bounded) |
| Throttling | ❌ None | ❌ None | ✅ Built-in |
| Cancellation | Manual | Manual | Built-in token |
| Progress Tracking | Manual | Manual | Via `ParallelLoopState` |
| Best For | Small collections | Small bounded sets | Large collections, I/O |

---

## Degree of Parallelism Control

### Using ParallelOptions

```csharp
// Control maximum concurrent operations
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 4,  // Max 4 at a time
    CancellationToken = cts.Token // Enable cancellation
};

await Parallel.ForEachAsync(urls, options, async (url, cancellationToken) =>
{
    await ProcessUrlAsync(url, cancellationToken);
});
```

### Real-World Example: Throttled API Calls

```csharp
public class ThrottledApiClient
{
    private readonly HttpClient _httpClient;
    private readonly int _maxConcurrency;

    public ThrottledApiClient(int maxConcurrency = 10)
    {
        _httpClient = new HttpClient();
        _maxConcurrency = maxConcurrency;
    }

    public async Task<IEnumerable<Result>> ProcessBatchAsync(
        IEnumerable<Request> requests,
        CancellationToken cancellationToken = default)
    {
        var results = new ConcurrentBag<Result>();
        
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = _maxConcurrency,
            CancellationToken = cancellationToken
        };

        await Parallel.ForEachAsync(requests, options, async (request, ct) =>
        {
            try
            {
                var result = await CallApiAsync(request, ct);
                results.Add(result);
            }
            catch (HttpRequestException ex)
            {
                Console.WriteLine($"Request failed: {ex.Message}");
                results.Add(new Result { Error = ex.Message });
            }
        });

        return results;
    }

    private async Task<Result> CallApiAsync(Request request, CancellationToken ct)
    {
        var response = await _httpClient.PostAsJsonAsync(
            "https://api.example.com/process", 
            request, 
            ct);
        
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<Result>(ct);
    }
}

// Usage: Process 1000 items with max 10 concurrent
var client = new ThrottledApiClient(maxConcurrency: 10);
var results = await client.ProcessBatchAsync(thousandRequests);
```

---

## Real-World Use Cases

### Use Case 1: Database Bulk Operations

```csharp
public class BulkDatabaseInserter
{
    private readonly string _connectionString;
    private readonly int _batchSize;

    public async Task InsertBatchAsync(IEnumerable<Record> records)
    {
        var semaphore = new SemaphoreSlim(_batchSize);
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = _batchSize
        };

        await Parallel.ForEachAsync(records, options, async (record, ct) =>
        {
            await using var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync(ct);
            
            await connection.ExecuteAsync(
                "INSERT INTO Records (Data) VALUES (@Data)",
                record,
                commandTimeout: 30);
                
            Console.WriteLine($"Inserted record {record.Id}");
        });
    }
}
```

### Use Case 2: Image Processing Pipeline

```csharp
public class ImageProcessor
{
    public async Task ProcessImagesAsync(IEnumerable<string> imagePaths)
    {
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = Environment.ProcessorCount
        };

        await Parallel.ForEachAsync(imagePaths, options, async (path, ct) =>
        {
            // Load image
            await using var stream = File.OpenRead(path);
            using var image = await Image.LoadAsync(stream, ct);
            
            // Process
            image.Mutate(x => x.Resize(new ResizeOptions
            {
                Size = new Size(800, 600),
                Mode = ResizeMode.Max
            }));
            
            // Save
            var outputPath = Path.Combine(
                Path.GetDirectoryName(path)!,
                "processed",
                Path.GetFileName(path));
                
            await image.SaveAsync(outputPath, ct);
            
            Console.WriteLine($"Processed: {Path.GetFileName(path)}");
        });
    }
}
```

### Use Case 3: Web Scraping with Rate Limiting

```csharp
public class WebScraper
{
    private readonly HttpClient _httpClient = new();
    private readonly SemaphoreSlim _rateLimiter = new(1, 1);

    public async Task ScrapePagesAsync(IEnumerable<Uri> urls)
    {
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 5
        };

        await Parallel.ForEachAsync(urls, options, async (url, ct) =>
        {
            // Rate limiting: max 1 request per second
            await _rateLimiter.WaitAsync(ct);
            try
            {
                var html = await _httpClient.GetStringAsync(url, ct);
                await ParseAndStoreAsync(url, html, ct);
            }
            finally
            {
                _rateLimiter.Release();
                await Task.Delay(1000, ct); // 1 second delay
            }
        });
    }

    private async Task ParseAndStoreAsync(Uri url, string html, CancellationToken ct)
    {
        // Parse and store logic
    }
}
```

---

## Memory and Execution Model

### How Parallel.ForEachAsync Works

```
┌─────────────────────────────────────────────────────────────────┐
│              PARALLEL.FOREACHASYNC EXECUTION MODEL              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Input: IEnumerable<T> items                                    │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Step 1: Partitioner divides items into chunks          │    │
│   │           ┌─────┬─────┬─────┬─────┬─────┐               │    │
│   │           │ 1,2 │ 3,4 │ 5,6 │ 7,8 │ 9,10│               │    │
│   │           └─────┴─────┴─────┴─────┴─────┘               │    │
│   └─────────────────────────────────────────────────────────┘    │
│                          │                                       │
│   ┌──────────────────────┴────────────────────────┐             │
│   │  Step 2: AsyncEnumerable pipeline               │             │
│   │                                                 │             │
│   │   ┌──────────────┐     ┌──────────────┐      │             │
│   │   │ Partition 1  │     │ Partition 2  │      │             │
│   │   │ ┌──────────┐ │     │ ┌──────────┐ │      │             │
│   │   │ │ Item 1   │ │     │ │ Item 3   │ │      │             │
│   │   │ │ async ⏳ │ │     │ │ async ⏳ │ │      │             │
│   │   │ └──────────┘ │     │ └──────────┘ │      │             │
│   │   │ ┌──────────┐ │     │ ┌──────────┐ │      │             │
│   │   │ │ Item 2   │ │     │ │ Item 4   │ │      │             │
│   │   │ │ async ⏳ │ │     │ │ async ⏳ │ │      │             │
│   │   │ └──────────┘ │     │ └──────────┘ │      │             │
│   │   └──────────────┘     └──────────────┘      │             │
│   │                                                 │             │
│   │   MaxDegreeOfParallelism controls how many    │             │
│   │   partitions run concurrently                 │             │
│   └─────────────────────────────────────────────────┘             │
│                                                                  │
│   Step 3: Each async operation yields control back              │
│   while waiting (I/O), allowing other work to proceed           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Thread Pool Usage

```csharp
// Unlike Parallel.For, ForEachAsync uses async/await
// which means:
// 1. Thread is released during I/O wait
// 2. ThreadPool thread resumes when I/O completes
// 3. Different threads may handle continuations

await Parallel.ForEachAsync(items, async (item, ct) =>
{
    // Thread A starts here
    Console.WriteLine($"Before await: {Thread.CurrentThread.ManagedThreadId}");
    
    await Task.Delay(1000); // Thread A is released
    
    // Thread B (or A again) resumes here
    Console.WriteLine($"After await: {Thread.CurrentThread.ManagedThreadId}");
});
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Always pass CancellationToken
await Parallel.ForEachAsync(items, 
    new ParallelOptions { CancellationToken = ct },
    async (item, cancellationToken) =>
    {
        await ProcessAsync(item, cancellationToken);
    });

// 2. Handle exceptions properly
try
{
    await Parallel.ForEachAsync(items, async (item, ct) =>
    {
        await ProcessAsync(item, ct);
    });
}
catch (AggregateException ex)
{
    foreach (var inner in ex.InnerExceptions)
    {
        Console.WriteLine($"Error: {inner.Message}");
    }
}

// 3. Use appropriate MaxDegreeOfParallelism
var options = new ParallelOptions
{
    // CPU-bound: use processor count
    MaxDegreeOfParallelism = Environment.ProcessorCount,
    
    // I/O-bound: higher values may be appropriate
    // MaxDegreeOfParallelism = 50
};

// 4. Avoid blocking operations inside
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    // ✅ Good: true async I/O
    await File.ReadAllTextAsync(item, ct);
    
    // ❌ Bad: blocking call
    // File.ReadAllText(item);
});
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Using async void (not supported)
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    await ProcessAsync(item); // ✅ Task-returning method
});

// PITFALL 2: Modifying shared state without synchronization
var count = 0;
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    // ❌ Race condition!
    count++;
    
    // ✅ Use Interlocked
    Interlocked.Increment(ref count);
    
    // ✅ Or Concurrent collections
    resultsBag.Add(item);
});

// PITFALL 3: Very small MaxDegreeOfParallelism with few items
// If you have 2 items and set MaxDOP to 4, you waste resources

// PITFALL 4: Not handling OperationCanceledException
await Parallel.ForEachAsync(items, 
    new ParallelOptions { CancellationToken = ct },
    async (item, cancellationToken) =>
    {
        try
        {
            await ProcessAsync(item, cancellationToken);
        }
        catch (OperationCanceledException)
        {
            // Handle cancellation gracefully
            Console.WriteLine("Processing cancelled");
            throw; // Re-throw if needed
        }
    });

// PITFALL 5: Order is not guaranteed
await Parallel.ForEachAsync(urls, async (url, ct) =>
{
    await DownloadAsync(url, ct);
});
// Downloads may complete in ANY order!
```

### Performance Considerations

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHEN TO USE WHAT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Sequential foreach + await                                     │
│   ├── Small collections (< 10 items)                          │
│   ├── Order matters                                             │
│   └── Each item depends on previous                            │
│                                                                  │
│   Task.WhenAll                                                   │
│   ├── Small bounded sets (you know the count)                  │
│   ├── Need ALL results before continuing                         │
│   └── Resources can handle max concurrency                       │
│                                                                  │
│   Parallel.ForEachAsync                                        │
│   ├── Large collections (100+ items)                            │
│   ├── I/O-bound operations (HTTP, DB, files)                   │
│   ├── Need throttling                                            │
│   └── Order doesn't matter                                       │
│                                                                  │
│   Parallel.For (non-async)                                      │
│   ├── CPU-bound operations                                       │
│   ├── Transformations                                            │
│   └── Parallel LINQ (PLINQ) alternatives                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: What is Parallel.ForEachAsync and when should you use it?**
> It's a .NET 6+ method for processing collections with async operations and controlled concurrency. Use it when you have many async I/O operations and want to limit how many run simultaneously (throttling).

**Q: What's the difference between Parallel.ForEachAsync and Task.WhenAll?**
> `Task.WhenAll` starts ALL tasks immediately (unlimited concurrency), while `Parallel.ForEachAsync` controls concurrency with `MaxDegreeOfParallelism`. Use WhenAll for small bounded sets, ForEachAsync for large collections.

**Q: How does Parallel.ForEachAsync handle the async state machine?**
> Each iteration runs on the ThreadPool. When an async operation awaits, the thread is released back to the pool. When the operation completes, any available ThreadPool thread can resume execution.

**Q: Can you guarantee order with Parallel.ForEachAsync?**
> No, order is not guaranteed. Items are processed concurrently and may complete in any order.

**Q: What happens if one iteration throws an exception?**
> `Parallel.ForEachAsync` collects all exceptions and throws an `AggregateException` at the end. You can use try/catch inside each iteration for individual error handling.

**Q: How do you cancel a Parallel.ForEachAsync operation?**
> Pass a `CancellationToken` in `ParallelOptions`. The token is passed to each iteration, and operations can check `cancellationToken.IsCancellationRequested` or use `ThrowIfCancellationRequested()`.
