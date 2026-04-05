# IDisposable and the `using` Statement in C#

## 🧠 What Is It?

`IDisposable` is an interface that signals a class holds **unmanaged resources** (file handles, database connections, network sockets, etc.) that must be explicitly released. The `using` statement/declaration ensures `Dispose()` is **always called**, even if an exception occurs.

---

## 🌍 Real-World Analogy

When you rent a car, you're responsible for returning it. The rental company doesn't know when you're done unless you return it. If you forget — the resource is stuck, unavailable to everyone else.

`IDisposable` + `using` is the **contract that guarantees the car gets returned**, even if your trip goes wrong (exception).

---

## ⚙️ Memory Model: Managed vs Unmanaged Resources

```
.NET Managed Heap
┌───────────────────────────────────────────────────┐
│  MyObject                                         │
│  ├── int field        ← GC handles this           │
│  ├── string field     ← GC handles this           │
│  └── FileStream       ← UNMANAGED: file handle    │
│       └── OS File Handle (outside GC reach)  ──── ┼──► OS Resource
└───────────────────────────────────────────────────┘
                                                         Must be freed explicitly!
```

The **GC cleans up managed memory** automatically. But OS handles, DB connections, etc. live **outside the GC**. Without `Dispose()`, these leak until the process ends or the OS reclaims them.

---

## 💻 Implementing IDisposable

```csharp
public class DatabaseConnection : IDisposable
{
    private readonly SqlConnection _connection;
    private bool _disposed = false;

    public DatabaseConnection(string connectionString)
    {
        _connection = new SqlConnection(connectionString);
        _connection.Open();
        Console.WriteLine("Connection opened.");
    }

    public void ExecuteQuery(string sql)
    {
        if (_disposed) throw new ObjectDisposedException(nameof(DatabaseConnection));
        // ... execute query
    }

    public void Dispose()
    {
        if (!_disposed)
        {
            _connection.Close();
            _connection.Dispose();
            _disposed = true;
            Console.WriteLine("Connection closed and disposed.");
        }
    }
}
```

---

## 💻 Classic `using` Block

The **classic syntax** (C# 1.0+): `Dispose()` is called when the block exits.

```csharp
static void ReadFile(string path)
{
    using (var reader = new StreamReader(path)) // ← opens resource
    {
        string content = reader.ReadToEnd();
        Console.WriteLine(content);
    } // ← Dispose() called here automatically (even if exception thrown)
}
```

**Equivalent expanded form** — this is exactly what the compiler generates:
```csharp
StreamReader reader = new StreamReader(path);
try
{
    string content = reader.ReadToEnd();
    Console.WriteLine(content);
}
finally
{
    reader?.Dispose(); // Always runs
}
```

---

## 💻 Modern `using` Declaration (C# 8.0+)

The **modern syntax**: no braces required. `Dispose()` is called when the **enclosing scope** ends.

```csharp
static void ReadFile(string path)
{
    using var reader = new StreamReader(path); // ← no braces!
    string content = reader.ReadToEnd();
    Console.WriteLine(content);
} // ← Dispose() called here when method exits
```

Cleaner, less indentation, same safety guarantee.

---

## 🆚 `using` Block vs `using` Declaration

| Feature | `using (var x = ...) { }` | `using var x = ...;` |
|---|---|---|
| C# version | 1.0+ | 8.0+ |
| Braces required | ✅ Yes | ❌ No |
| Dispose scope | End of `{ }` block | End of enclosing method/scope |
| Multiple resources | Nest blocks (verbose) | Multiple `using var` lines (clean) |
| Explicit early disposal | ✅ Yes (close the block) | ⚠️ Only at end of scope |

```csharp
// Classic — verbose but early disposal control
using (var a = new FileStream("a.txt", FileMode.Open))
using (var b = new FileStream("b.txt", FileMode.Open))
{
    // both disposed at end of block
}

// Modern — cleaner
using var a = new FileStream("a.txt", FileMode.Open);
using var b = new FileStream("b.txt", FileMode.Open);
// both disposed at end of method
```

---

## 💻 Async Dispose: `IAsyncDisposable` and `await using`

For async resources (e.g., async DB connections, streams):

```csharp
public class AsyncDbContext : IAsyncDisposable
{
    private readonly DbConnection _conn;

    public AsyncDbContext(string connStr)
    {
        _conn = new SqlConnection(connStr);
    }

    public async ValueTask DisposeAsync()
    {
        await _conn.CloseAsync();
        Console.WriteLine("Async connection closed.");
    }
}

// Usage:
await using var ctx = new AsyncDbContext(connectionString);
// ... do async work
// DisposeAsync() called automatically
```

---

## ⚠️ Dispose Pattern for Inherited Classes

For classes with both managed and unmanaged resources and possible inheritance:

```csharp
public class ResourceHolder : IDisposable
{
    private bool _disposed = false;
    private IntPtr _unmanagedHandle; // e.g., P/Invoke handle
    private Stream _managedStream;

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // Release managed resources
                _managedStream?.Dispose();
            }
            // Release unmanaged resources (always)
            if (_unmanagedHandle != IntPtr.Zero)
            {
                NativeMethods.CloseHandle(_unmanagedHandle);
                _unmanagedHandle = IntPtr.Zero;
            }
            _disposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // Tell GC: no need to finalize
    }

    ~ResourceHolder() // Finalizer — safety net
    {
        Dispose(false);
    }
}
```

---

## 📌 Summary

> `IDisposable` is your contract for deterministic cleanup of resources. Always use `using` (block or declaration) to ensure `Dispose()` is called. For async code, use `IAsyncDisposable` with `await using`. The **modern `using var` declaration** is cleaner and preferred in C# 8.0+.
