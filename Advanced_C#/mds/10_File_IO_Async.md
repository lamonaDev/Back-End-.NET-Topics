# Async File I/O in C#

## Table of Contents
1. [Understanding Async File I/O](#understanding-async-file-io)
2. [FileStream and Async Operations](#filestream-and-async-operations)
3. [Buffered vs Unbuffered I/O](#buffered-vs-unbuffered-io)
4. [I/O Completion Ports (IOCP)](#io-completion-ports-iocp)
5. [Memory and Performance](#memory-and-performance)
6. [Real-World Patterns](#real-world-patterns)
7. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Async File I/O

### Why Async File I/O?

Traditional synchronous file operations **block threads**, consuming valuable ThreadPool resources while waiting for the operating system to complete disk operations. Async I/O releases threads back to the pool during I/O waits.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SYNC VS ASYNC FILE I/O                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Synchronous Read:                                             │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Thread 1: ReadFile() ──> [ BLOCKED waiting for disk ]    │    │
│   │              ↓                                              │    │
│   │         OS schedules disk I/O                             │    │
│   │              ↓                                              │    │
│   │         Thread 1 waits... (1-10ms typical)                │    │
│   │              ↓                                              │    │
│   │         Data returned                                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│   Thread is BLOCKED for entire operation duration               │
│                                                                  │
│   Asynchronous Read:                                            │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Thread 1: ReadAsync() ──> [ Registers callback ]        │    │
│   │              ↓                                              │    │
│   │         Thread 1 returns to ThreadPool                  │    │
│   │              ↓                                              │    │
│   │         [OS handles I/O independently]                   │    │
│   │              ↓                                              │    │
│   │         I/O completes ──> ThreadPool resumes task        │    │
│   └─────────────────────────────────────────────────────────┘    │
│   Thread is FREE during I/O operation                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Async File Operations

```csharp
// ❌ Synchronous - blocks thread
string content = File.ReadAllText("data.txt");

// ✅ Asynchronous - releases thread
string content = await File.ReadAllTextAsync("data.txt");

// All File static methods have async variants:
await File.ReadAllTextAsync(path);
await File.ReadAllBytesAsync(path);
await File.ReadAllLinesAsync(path);
await File.WriteAllTextAsync(path, content);
await File.WriteAllBytesAsync(path, bytes);
await File.AppendAllTextAsync(path, content);
```

---

## FileStream and Async Operations

### Creating Async-Optimized FileStream

```csharp
// ⚠️ Default FileStream is NOT optimized for async!
using var stream = new FileStream("file.txt", FileMode.Open);
await stream.ReadAsync(buffer); // Still works but not optimal

// ✅ Properly configured for async I/O
using var stream = new FileStream(
    "file.txt",
    FileMode.Open,
    FileAccess.Read,
    FileShare.Read,
    bufferSize: 8192,        // Buffer size
    useAsync: true);          // Enable true async I/O

await stream.ReadAsync(buffer);
```

### Complete FileStream Example

```csharp
public class AsyncFileProcessor
{
    public async Task<byte[]> ReadFileAsync(string path, CancellationToken ct = default)
    {
        await using var stream = new FileStream(
            path,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            bufferSize: 8192,
            useAsync: true);

        // For known size files
        if (stream.Length > int.MaxValue)
        {
            throw new IOException("File too large");
        }

        var buffer = new byte[stream.Length];
        int bytesRead = 0;
        
        while (bytesRead < buffer.Length)
        {
            int read = await stream.ReadAsync(
                buffer.AsMemory(bytesRead), 
                ct);
            
            if (read == 0) break; // EOF
            bytesRead += read;
        }

        return buffer;
    }

    public async Task WriteFileAsync(
        string path, 
        byte[] data, 
        CancellationToken ct = default)
    {
        await using var stream = new FileStream(
            path,
            FileMode.Create,
            FileAccess.Write,
            FileShare.None,
            bufferSize: 8192,
            useAsync: true);

        await stream.WriteAsync(data, ct);
        await stream.FlushAsync(ct); // Ensure written to disk
    }
}
```

### Streaming Large Files

```csharp
public async Task ProcessLargeFileAsync(
    string inputPath,
    string outputPath,
    Func<ReadOnlyMemory<byte>, byte[]> processor,
    CancellationToken ct = default)
{
    const int bufferSize = 8192;
    
    await using var inputStream = new FileStream(
        inputPath, FileMode.Open, FileAccess.Read, FileShare.Read,
        bufferSize, useAsync: true);
    
    await using var outputStream = new FileStream(
        outputPath, FileMode.Create, FileAccess.Write, FileShare.None,
        bufferSize, useAsync: true);

    var buffer = new byte[bufferSize];
    int bytesRead;
    
    while ((bytesRead = await inputStream.ReadAsync(buffer, ct)) > 0)
    {
        var processed = processor(buffer.AsMemory(0, bytesRead));
        await outputStream.WriteAsync(processed, ct);
    }
}
```

---

## Buffered vs Unbuffered I/O

### Understanding Buffering

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUFFERED VS UNBUFFERED I/O                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Buffered I/O (default):                                        │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Application                                            │    │
│   │    └── Read(100 bytes)  ──> User Buffer                 │    │
│   │           ↑                                              │    │
│   │  FileStream Buffer (8KB)  ──> filled from disk          │    │
│   │           ↑                                              │    │
│   │  Operating System Cache                                 │    │
│   │           ↑                                              │    │
│   │  Physical Disk                                          │    │
│   └─────────────────────────────────────────────────────────┘    │
│   Benefits: Fewer system calls, read-ahead optimization         │
│                                                                  │
│   Unbuffered I/O (FileOptions.WriteThrough | FileOptions.     │
│   NoBuffering):                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Application                                            │    │
│   │    └── Read(buffer) ──> Direct to disk (no cache)       │    │
│   │           ↑                                              │    │
│   │  Physical Disk                                          │    │
│   └─────────────────────────────────────────────────────────┘    │
│   Benefits: Predictable performance, no cache pollution           │
│   Drawbacks: Requires aligned buffers, more system calls         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Each

```csharp
// Standard buffered I/O (most common)
using var standard = new FileStream(
    path, FileMode.Open, FileAccess.Read, FileShare.Read);

// Unbuffered for database-like scenarios
using var unbuffered = new FileStream(
    path, 
    FileMode.Open, 
    FileAccess.Read, 
    FileShare.Read,
    bufferSize: 0,  // No FileStream buffering
    FileOptions.WriteThrough | FileOptions.SequentialScan);

// Sequential access hint (helps OS cache)
using var sequential = new FileStream(
    path, 
    FileMode.Open, 
    FileAccess.Read, 
    FileShare.Read,
    bufferSize: 8192,
    FileOptions.SequentialScan);

// Random access (disables read-ahead)
using var random = new FileStream(
    path, 
    FileMode.Open, 
    FileAccess.Read, 
    FileShare.Read,
    bufferSize: 4096,
    FileOptions.RandomAccess);
```

---

## I/O Completion Ports (IOCP)

### What are IOCP?

I/O Completion Ports are the Windows mechanism enabling true asynchronous I/O. When an async operation completes, the OS places the result in a queue, and a ThreadPool thread picks it up - no blocking, no polling.

```
┌─────────────────────────────────────────────────────────────────┐
│                    IOCP ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ThreadPool                     I/O Completion Port            │
│   ┌──────────────┐              ┌─────────────────────┐       │
│   │ Thread 1     │<─────────────│                     │       │
│   │ Thread 2     │   (sleeps    │  ┌───────────────┐  │       │
│   │ Thread 3     │    waiting)   │  │ I/O Request 1 │──┼───────>│ Disk
│   │ Thread 4     │              │  │ I/O Request 2 │──┼───────>│ Disk
│   └──────────────┘              │  │ I/O Request 3 │──┼───────>│ Disk
│        ↑                        │  └───────────────┘  │       │
│        │                        │                     │       │
│        │ Resume on completion   │  ┌───────────────┐  │       │
│        └────────────────────────│  │ Completed I/O │──┘       │
│                                 │  │ (with result) │           │
│                                 │  └───────────────┘           │
│                                 └─────────────────────┘         │
│                                                                  │
│   Key benefits:                                                  │
│   ├─ No thread blocking during I/O                             │
│   ├─ Automatic load balancing (throttle concurrency)            │
│   └─ Most efficient Windows async mechanism                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### How .NET Uses IOCP

```csharp
// When you call FileStream.ReadAsync with useAsync:true:
// 1. .NET calls Windows ReadFile with overlapped I/O
// 2. Windows returns immediately (pending status)
// 3. Thread returns to ThreadPool
// 4. When disk I/O completes, Windows queues result to IOCP
// 5. ThreadPool thread dequeues and resumes your async method

// This is automatically handled by .NET - you just use await:
await using var stream = new FileStream(
    path, FileMode.Open, FileAccess.Read, FileShare.Read,
    bufferSize: 8192, useAsync: true);

await stream.ReadAsync(buffer); // Uses IOCP internally
```

---

## Memory and Performance

### Memory Pooling with ArrayPool

```csharp
public class PooledFileReader : IDisposable
{
    private readonly ArrayPool<byte> _pool = ArrayPool<byte>.Shared;

    public async Task ProcessFileAsync(string path, CancellationToken ct = default)
    {
        byte[]? rented = _pool.Rent(8192);
        
        try
        {
            await using var stream = new FileStream(
                path, FileMode.Open, FileAccess.Read, FileShare.Read,
                bufferSize: 8192, useAsync: true);

            int read;
            while ((read = await stream.ReadAsync(rented.AsMemory(0, 8192), ct)) > 0)
            {
                ProcessBuffer(rented.AsSpan(0, read));
            }
        }
        finally
        {
            _pool.Return(rented);
        }
    }

    private void ProcessBuffer(ReadOnlySpan<byte> buffer)
    {
        // Process the data
    }

    public void Dispose()
    {
        // Pool is shared, no disposal needed
    }
}
```

### Using Memory<T> and Span<T>

```csharp
public async Task EfficientCopyAsync(
    string sourcePath, 
    string destPath,
    CancellationToken ct = default)
{
    // Use Memory<T> for async operations
    byte[] buffer = GC.AllocateUninitializedArray<byte>(8192, pinned: true);
    
    await using var source = new FileStream(
        sourcePath, FileMode.Open, FileAccess.Read, FileShare.Read,
        bufferSize: 8192, useAsync: true);
    
    await using var dest = new FileStream(
        destPath, FileMode.Create, FileAccess.Write, FileShare.None,
        bufferSize: 8192, useAsync: true);

    int read;
    while ((read = await source.ReadAsync(buffer.AsMemory(0, 8192), ct)) > 0)
    {
        await dest.WriteAsync(buffer.AsMemory(0, read), ct);
    }
}
```

---

## Real-World Patterns

### Pattern 1: Line-by-Line Processing

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await using var stream = new FileStream(
        path, FileMode.Open, FileAccess.Read, FileShare.Read,
        bufferSize: 8192, useAsync: true);
    
    using var reader = new StreamReader(stream);
    
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) != null)
    {
        yield return line;
    }
}

// Usage
await foreach (var line in ReadLinesAsync("data.txt", ct))
{
    ProcessLine(line);
}
```

### Pattern 2: Chunked File Processing

```csharp
public class ChunkedFileProcessor
{
    public async Task ProcessInChunksAsync(
        string path,
        Func<byte[], int, Task> processChunk,
        int chunkSize = 8192,
        CancellationToken ct = default)
    {
        await using var stream = new FileStream(
            path, FileMode.Open, FileAccess.Read, FileShare.Read,
            bufferSize: chunkSize, useAsync: true);

        var buffer = new byte[chunkSize];
        int bytesRead;
        long totalRead = 0;

        while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
        {
            await processChunk(buffer, bytesRead);
            totalRead += bytesRead;
            
            Console.WriteLine($"Processed {totalRead} bytes...");
        }
    }
}
```

### Pattern 3: Async File Watcher

```csharp
public class AsyncFileWatcher : IDisposable
{
    private readonly FileSystemWatcher _watcher;
    private readonly Channel<string> _fileChannel;

    public AsyncFileWatcher(string path, string filter)
    {
        _watcher = new FileSystemWatcher(path, filter)
        {
            EnableRaisingEvents = true,
            NotifyFilter = NotifyFilters.FileName | NotifyFilters.LastWrite
        };
        
        _watcher.Created += OnFileCreated;
        _fileChannel = Channel.CreateUnbounded<string>();
    }

    private void OnFileCreated(object sender, FileSystemEventArgs e)
    {
        _fileChannel.Writer.TryWrite(e.FullPath);
    }

    public async IAsyncEnumerable<string> GetNewFilesAsync(
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var file in _fileChannel.Reader.ReadAllAsync(ct))
        {
            yield return file;
        }
    }

    public void Dispose()
    {
        _watcher.Dispose();
        _fileChannel.Writer.Complete();
    }
}
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Always use async methods with await
await File.ReadAllTextAsync(path); // ✅
File.ReadAllText(path); // ❌ Blocking

// 2. Configure FileStream for async
new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read,
    bufferSize: 8192, useAsync: true); // ✅

// 3. Use CancellationToken
await stream.ReadAsync(buffer, cancellationToken); // ✅

// 4. Proper disposal with await using
await using var stream = new FileStream(...); // ✅

// 5. Pool buffers for large operations
var buffer = ArrayPool<byte>.Shared.Rent(size);
try { /* use */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }

// 6. Flush async before closing
await stream.FlushAsync(); // Ensure data written
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Blocking in async method
public async Task Bad()
{
    var content = File.ReadAllText(path); // ❌ Blocking!
    await Task.CompletedTask;
}

// PITFALL 2: Forgetting useAsync:true
using var stream = new FileStream(path, FileMode.Open); // ❌ Sync!
await stream.ReadAsync(buffer); // Still works but not optimal

// PITFALL 3: Not handling exceptions
// File I/O can fail - always handle IOException

// PITFALL 4: Large buffer sizes
// Buffers > 85KB go to LOH (Large Object Heap)
new FileStream(..., bufferSize: 100_000); // ❌ Risky

// PITFALL 5: Not checking read amount
var read = await stream.ReadAsync(buffer);
// May have read less than buffer.Length!

// PITFALL 6: Opening files with wrong sharing
new FileStream(path, FileMode.Open); // Exclusive by default
// Use FileShare.Read for concurrent reads
```

---

## Interview Questions

**Q: What's the difference between File.ReadAllText and File.ReadAllTextAsync?**
> ReadAllText blocks the thread during file operation. ReadAllTextAsync releases the thread during I/O, allowing it to process other work. Async uses IOCP on Windows for true asynchronous I/O.

**Q: Why should you set useAsync:true in FileStream constructor?**> useAsync:true enables true overlapped I/O on Windows (IOCP). Without it, async operations may fall back to sync-over-async, blocking threads. It also optimizes internal buffering for async patterns.

**Q: What is IOCP and why does it matter?**> I/O Completion Ports are Windows' mechanism for efficient async I/O. They allow the OS to queue completed I/O operations, which ThreadPool threads pick up. This enables true async without blocking or polling.

**Q: When should you use ArrayPool for file operations?**> Use ArrayPool when doing repeated file I/O operations with large buffers. It reduces GC pressure by reusing arrays instead of allocating new ones. Important for high-throughput scenarios.

**Q: What's the impact of buffer size on FileStream?**> Larger buffers reduce system calls but use more memory. Default 4KB is usually good. Buffers over 85KB go to LOH causing memory issues. For sequential reads, larger buffers (8-64KB) can help.

**Q: Can async file I/O be slower than sync in some cases?**> Yes, for small files the overhead of async setup may exceed benefits. Async shines with large files, many concurrent operations, or when thread pool starvation is a concern. For tiny files (<4KB), sync may be faster.
