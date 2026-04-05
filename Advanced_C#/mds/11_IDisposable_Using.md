# IDisposable and Using in C#

## Table of Contents
1. [Understanding IDisposable](#understanding-idisposable)
2. [Managed vs Unmanaged Resources](#managed-vs-unmanaged-resources)
3. [Implementing IDisposable](#implementing-idisposable)
4. [Using Statement and Declaration](#using-statement-and-declaration)
5. [Await Using](#await-using)
6. [Memory and Finalization](#memory-and-finalization)
7. [Real-World Patterns](#real-world-patterns)
8. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding IDisposable

### What is IDisposable?

`IDisposable` is an interface that defines a single method `Dispose()` for releasing **unmanaged resources** deterministically. It provides explicit control over resource cleanup rather than waiting for garbage collection.

### Why Dispose Matters

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESOURCE LIFECYCLE COMPARISON                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WITHOUT IDisposable (GC only):                                │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  FileStream created                                     │    │
│   │       │                                                 │    │
│   │       ↓                                                 │    │
│   │  File handle acquired by OS                             │    │
│   │       │                                                 │    │
│   │       ↓ (variable goes out of scope)                  │    │
│   │  Object "orphaned" - still holds file handle            │    │
│   │       │                                                 │    │
│   │       ↓ (unknown time passes)                         │    │
│   │  GC runs                                                │    │
│   │       │                                                 │    │
│   │       ↓                                                 │    │
│   │  Finalizer runs (if present)                            │    │
│   │       │                                                 │    │
│   │       ↓                                                 │    │
│   │  File handle finally released                            │    │
│   │  (could be minutes or hours later!)                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   WITH IDisposable:                                             │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  using var stream = new FileStream(...);                 │    │
│   │       │                                                 │    │
│   │  File handle acquired                                   │    │
│   │       │                                                 │    │
│   │       ↓ (end of scope or explicit dispose)              │    │
│   │  Dispose() called IMMEDIATELY                           │    │
│   │       │                                                 │    │
│   │       ↓                                                 │    │
│   │  File handle released - deterministic!                  │    │
│   │  (within milliseconds)                                    │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The IDisposable Interface

```csharp
public interface IDisposable
{
    void Dispose();
}
```

### Basic Usage with Using

```csharp
// ❌ Without using - resource leak risk
public void BadExample()
{
    var stream = new FileStream("file.txt", FileMode.Open);
    var reader = new StreamReader(stream);
    // If exception thrown here, resources never cleaned up!
    Console.WriteLine(reader.ReadToEnd());
    reader.Close();
    stream.Close();
}

// ✅ With using - guaranteed cleanup
public void GoodExample()
{
    using (var stream = new FileStream("file.txt", FileMode.Open))
    using (var reader = new StreamReader(stream))
    {
        Console.WriteLine(reader.ReadToEnd());
    } // Dispose called automatically, even if exception
}
```

---

## Managed vs Unmanaged Resources

### Resource Types

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESOURCE CLASSIFICATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   MANAGED RESOURCES:                                             │
│   ├─ Memory allocated by .NET runtime                           │
│   ├─ Other managed objects                                     │
│   ├─ Automatically handled by GC                                │
│   └─ No explicit cleanup needed                                 │
│                                                                  │
│   UNMANAGED RESOURCES:                                         │
│   ├─ File handles                                              │
│   ├─ Network sockets                                           │
│   ├─ Database connections                                      │
│   ├─ Memory allocated outside .NET (native)                   │
│   ├─ Windows handles (HWND, HDC, etc.)                        │
│   ├─ COM objects                                               │
│   └─ MUST be explicitly released                                │
│                                                                  │
│   IDisposable is for UNMANAGED resources!                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Examples of Resources Needing Disposal

```csharp
// File streams
using var fileStream = new FileStream("data.txt", FileMode.Open);

// Database connections
using var connection = new SqlConnection(connectionString);

// Network streams
using var tcpClient = new TcpClient();
using var networkStream = tcpClient.GetStream();

// Graphics objects
using var bitmap = new Bitmap(100, 100);
using var graphics = Graphics.FromImage(bitmap);

// Cryptography
using var aes = Aes.Create();

// COM interop
using var excel = new Microsoft.Office.Interop.Excel.Application();
```

---

## Implementing IDisposable

### Standard Dispose Pattern

```csharp
public class ResourceHolder : IDisposable
{
    // Track if already disposed
    private bool _disposed = false;
    
    // Unmanaged resource
    private IntPtr _unmanagedResource;
    
    // Managed disposable resource
    private Stream? _managedResource;

    public ResourceHolder()
    {
        _unmanagedResource = AllocateNativeResource();
        _managedResource = new FileStream("file.txt", FileMode.Open);
    }

    // Public Dispose method
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // Tell GC not to call finalizer
    }

    // Protected virtual for inheritance
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return; // Prevent double dispose

        if (disposing)
        {
            // Dispose managed resources
            _managedResource?.Dispose();
            _managedResource = null;
        }

        // Free unmanaged resources (always)
        if (_unmanagedResource != IntPtr.Zero)
        {
            FreeNativeResource(_unmanagedResource);
            _unmanagedResource = IntPtr.Zero;
        }

        _disposed = true;
    }

    // Finalizer (destructor) - only for unmanaged resources
    ~ResourceHolder()
    {
        Dispose(false);
    }

    // Helper methods
    private IntPtr AllocateNativeResource() => IntPtr.Zero;
    private void FreeNativeResource(IntPtr resource) { }

    // Guard method for public methods
    private void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(nameof(ResourceHolder));
    }
}
```

### Simplified Pattern (Managed Only)

```csharp
public class SimpleResource : IDisposable
{
    private bool _disposed;
    private readonly Stream _stream;

    public SimpleResource(Stream stream)
    {
        _stream = stream;
    }

    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        // Use _stream...
    }

    public void Dispose()
    {
        if (_disposed) return;
        
        _stream.Dispose();
        _disposed = true;
        
        GC.SuppressFinalize(this);
    }
}
```

---

## Using Statement and Declaration

### Using Statement (Traditional)

```csharp
// Single resource
using (var stream = new FileStream("file.txt", FileMode.Open))
{
    // Use stream
    var data = new byte[stream.Length];
    stream.Read(data, 0, (int)stream.Length);
} // Dispose called here

// Multiple resources
using (var stream = new FileStream("file.txt", FileMode.Open))
using (var reader = new StreamReader(stream))
using (var writer = new StreamWriter("output.txt"))
{
    // All three resources in scope
    writer.Write(reader.ReadToEnd());
}
// All disposed in reverse order (LIFO)

// With condition
Stream? stream = GetStream();
using (stream) // Works even if stream is null
{
    stream?.Read(buffer, 0, buffer.Length);
}
```

### Using Declaration (C# 8.0+)

```csharp
public void ProcessFile()
{
    using var stream = new FileStream("input.txt", FileMode.Open);
    using var reader = new StreamReader(stream);
    
    // Resources disposed at end of scope (end of method)
    string content = reader.ReadToEnd();
    Console.WriteLine(content);
} // Disposed here

// Good for early returns
public string? TryReadFile(string path)
{
    if (!File.Exists(path))
        return null; // fileStream never created, no dispose needed

    using var fileStream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(fileStream);
    
    if (reader.EndOfStream)
        return null; // Still properly disposed!

    return reader.ReadToEnd();
}
```

### Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    USING STATEMENT VS DECLARATION                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Using Statement:                                              │
│   ├─ Explicit block scope                                      │
│   ├─ Dispose at closing brace                                  │
│   ├─ Good for tight resource management                        │
│   └─ Example:                                                   │
│     using (var stream = ...)                                   │
│     {                                                           │
│         // limited scope                                        │
│     } // disposed immediately                                   │
│                                                                  │
│   Using Declaration:                                            │
│   ├─ Implicit scope to end of block                          │
│   ├─ Dispose at end of scope                                   │
│   ├─ Cleaner code, less nesting                                │
│   └─ Example:                                                   │
│     using var stream = ...;                                    │
│     // scope continues                                         │
│     // ...                                                      │
│   // disposed at end of scope                                  │
│                                                                  │
│   Choose statement for tight control,                         │
│   declaration for cleaner code flow                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Await Using

### IAsyncDisposable

```csharp
// IAsyncDisposable for async cleanup
public interface IAsyncDisposable
{
    ValueTask DisposeAsync();
}
```

### Implementing IAsyncDisposable

```csharp
public class AsyncResource : IAsyncDisposable
{
    private bool _disposed;
    private readonly Stream _stream;

    public AsyncResource(Stream stream)
    {
        _stream = stream;
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;

        // Async cleanup operations
        await _stream.FlushAsync();
        await _stream.DisposeAsync();
        
        _disposed = true;
        
        GC.SuppressFinalize(this);
    }
}

// Combined pattern (supports both sync and async)
public class UniversalResource : IDisposable, IAsyncDisposable
{
    private bool _disposed;
    private Stream _stream = null!;

    public void Dispose()
    {
        if (_disposed) return;
        
        _stream.Dispose();
        _disposed = true;
        
        GC.SuppressFinalize(this);
    }

    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        
        await _stream.DisposeAsync();
        _disposed = true;
        
        GC.SuppressFinalize(this);
    }
}
```

### Using await using

```csharp
public async Task ProcessAsync()
{
    // await using - async disposal
    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();
    
    await using var command = connection.CreateCommand();
    command.CommandText = "SELECT * FROM Users";
    
    await using var reader = await command.ExecuteReaderAsync();
    while (await reader.ReadAsync())
    {
        Console.WriteLine(reader["Name"]);
    }
} // All DisposeAsync called automatically

// Combined with ConfigureAwait
await using var stream = new FileStream("file.txt", FileMode.Open);
await using var reader = new StreamReader(stream);
var content = await reader.ReadToEndAsync().ConfigureAwait(false);
```

---

## Memory and Finalization

### Finalization Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    FINALIZATION PROCESS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Object becomes unreachable:                                   │
│        │                                                         │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  GC Generation 0 Collection                              │    │
│   │  ├── Object marked unreachable                           │    │
│   │  ├── Has finalizer?                                      │    │
│   │  │   ├── YES → Moved to finalization queue                │    │
│   │  │   └── NO  → Memory reclaimed                           │    │
│   │  └── ...                                                │    │
│   └─────────────────────────────────────────────────────────┘    │
│        │                                                         │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Finalizer Thread                                        │    │
│   │  ├── Dequeues object from finalization queue              │    │
│   │  ├── Calls Finalize() (~ method)                         │    │
│   │  ├── Object moved to next generation                     │    │
│   │  └── Memory freed (next GC cycle)                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Key points:                                                   │
│   ├─ Finalizer runs on dedicated thread                        │
│   ├─ Order of finalization is NOT guaranteed                   │
│   ├─ Objects with finalizers survive longer (promoted)         │
│   ├─ SuppressFinalize prevents finalization                    │
│   └─ Finalizers should be last resort!                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### GC.SuppressFinalize

```csharp
public void Dispose()
{
    Dispose(true);
    GC.SuppressFinalize(this); // Critical!
    // Prevents finalizer from running - object can be collected immediately
}
```

---

## Real-World Patterns

### Pattern 1: Repository Pattern with Disposal

```csharp
public class UnitOfWork : IDisposable
{
    private bool _disposed;
    private readonly DbContext _context;
    private readonly Dictionary<Type, object> _repositories = new();

    public UnitOfWork(DbContext context)
    {
        _context = context;
    }

    public IRepository<T> Repository<T>() where T : class
    {
        if (!_repositories.ContainsKey(typeof(T)))
        {
            _repositories[typeof(T)] = new Repository<T>(_context);
        }
        return (IRepository<T>)_repositories[typeof(T)];
    }

    public async Task SaveChangesAsync() => await _context.SaveChangesAsync();

    public void Dispose()
    {
        if (_disposed) return;
        
        _context.Dispose();
        _disposed = true;
        
        GC.SuppressFinalize(this);
    }
}

// Usage
public async Task UpdateCustomerAsync(int id, CustomerDto dto)
{
    using var unitOfWork = new UnitOfWork(_context);
    var repo = unitOfWork.Repository<Customer>();
    var customer = await repo.GetByIdAsync(id);
    customer.UpdateFrom(dto);
    await unitOfWork.SaveChangesAsync();
} // All resources cleaned up
```

### Pattern 2: HttpClient Factory Pattern

```csharp
public class ApiClient : IDisposable
{
    private readonly HttpClient _httpClient;
    private bool _disposed;

    public ApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<T> GetAsync<T>(string endpoint)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        
        var response = await _httpClient.GetAsync(endpoint);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>()!;
    }

    public void Dispose()
    {
        if (_disposed) return;
        
        // Don't dispose HttpClient if injected (factory manages it)
        // _httpClient.Dispose();
        
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}
```

### Pattern 3: Transaction Scope

```csharp
public async Task TransferFundsAsync(int fromId, int toId, decimal amount)
{
    await using var transaction = await _context.Database
        .BeginTransactionAsync();
    
    try
    {
        var fromAccount = await _context.Accounts.FindAsync(fromId);
        var toAccount = await _context.Accounts.FindAsync(toId);
        
        fromAccount.Balance -= amount;
        toAccount.Balance += amount;
        
        await _context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
} // Transaction disposed (rolled back if not committed)
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Implement dispose pattern if class holds unmanaged resources
public class MyClass : IDisposable
{
    private bool _disposed;
    
    public void Dispose()
    {
        if (_disposed) return; // Guard against double dispose
        // cleanup
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}

// 2. Check disposed state in public methods
public void DoWork()
{
    ObjectDisposedException.ThrowIf(_disposed, this);
    // ...
}

// 3. Use using/await using consistently
using var resource = new MyResource();

// 4. Dispose in reverse order of creation
using var stream = new FileStream(...);
using var reader = new StreamReader(stream); // reader uses stream
// reader disposed first, then stream

// 5. Don't throw from Dispose
public void Dispose()
{
    try
    {
        _stream?.Dispose();
    }
    catch
    {
        // Swallow or log - don't throw
    }
}

// 6. Make Dispose idempotent
public void Dispose()
{
    if (_disposed) return; // Already disposed? OK!
    // ...
}
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Double dispose issues
var stream = new FileStream(...);
stream.Dispose();
stream.Dispose(); // Some objects throw, most are OK
// But stream.Read() after dispose throws!

// PITFALL 2: Using disposed object
using var stream = new FileStream(...);
stream.Dispose();
stream.Read(buffer, 0, 100); // ObjectDisposedException!

// PITFALL 3: Not disposing in exception
Stream stream = null;
try
{
    stream = new FileStream(...);
    // exception thrown here
}
catch { throw; }
// stream never disposed! ❌

// ✅ Use using instead:
try
{
    using var stream = new FileStream(...);
    // exception handled, still disposed
}

// PITFALL 4: Capturing disposed resource in closure
using var stream = new FileStream(...);
Task.Run(() => stream.Read(...)); // ❌ May use after dispose!

// PITFALL 5: Dispose called on wrong thread
// Some COM objects have thread affinity
// Dispose should generally be called on same thread
```

---

## Interview Questions

**Q: What's the purpose of IDisposable?**> IDisposable provides deterministic cleanup of unmanaged resources. It allows immediate release of resources like file handles, network connections, and native memory rather than waiting for garbage collection.

**Q: What's the difference between using statement and using declaration?**> Using statement (C# 1.0) requires braces and disposes at the closing brace. Using declaration (C# 8.0) doesn't need braces and disposes at the end of the enclosing scope. Declaration produces cleaner code for single-resource scenarios.

**Q: When should you implement a finalizer (~ClassName)?**> Only when your class directly holds unmanaged resources (not wrapped in SafeHandle). Most modern code should use SafeHandle instead of implementing finalizers directly. Always call GC.SuppressFinalize(this) in Dispose().

**Q: What's the dispose pattern with bool disposing parameter?**> The protected virtual void Dispose(bool disposing) pattern allows:
- disposing=true: Called from Dispose(), clean managed and unmanaged resources
- disposing=false: Called from finalizer, only clean unmanaged resources
This handles cases where managed objects may already be finalized.

**Q: What's IAsyncDisposable and when do you need it?**> IAsyncDisposable is for resources that need async cleanup (like flushing buffers to network). Use await using for types implementing it. It's common in I/O-heavy scenarios and was introduced in C# 8.0.

**Q: Why is GC.SuppressFinalize important?**> Without SuppressFinalize, the finalizer will still run even after explicit disposal, delaying object collection by at least one extra GC generation. This hurts performance for no benefit.
