# Async File I/O in C#

## 🧠 What Is It?

Async file I/O lets you **read and write files without blocking a thread** while waiting for the disk. Since disk operations are slow (relative to CPU), blocking a thread while waiting for I/O wastes a valuable resource.

With `async/await`, the thread is **freed** while the disk does its work, and resumed when data is ready.

---

## 🌍 Real-World Analogy

**Synchronous (blocking):**  
You go to a printer, press print, and stand there staring at it until your document comes out. You can't do anything else.

**Asynchronous:**  
You press print, walk back to your desk and work on something else. When printing is done, you're notified and go pick it up.

---

## ⚙️ Memory & Execution Model

```
Thread (sync - BLOCKED)
──────────────────────────────────────────────────────
  ░░░░░░░░░░ WAITING FOR DISK ░░░░░░░░░░  → continues

Thread (async - FREE during I/O)
──────────────────────────────────────────────────────
  [start read] → yield →  [thread is FREE for other work]
                                  ↑
                           I/O Completion Port
                           signals when done
                                  ↓
                           [thread resumes] → process data
```

> Under the hood, .NET uses **I/O Completion Ports (IOCP)** on Windows (and `epoll`/`kqueue` on Linux/macOS). These are OS-level async mechanisms that notify the runtime when I/O is complete — no thread spins waiting.

---

## 💻 Reading a File Asynchronously

```csharp
using System;
using System.IO;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        string path = "data.txt";

        // Async read — entire file as string
        string content = await File.ReadAllTextAsync(path);
        Console.WriteLine(content);

        // Async read — all lines as array
        string[] lines = await File.ReadAllLinesAsync(path);
        foreach (var line in lines)
            Console.WriteLine(line);

        // Async read — binary bytes
        byte[] bytes = await File.ReadAllBytesAsync(path);
        Console.WriteLine($"File size: {bytes.Length} bytes");
    }
}
```

---

## 💻 Writing a File Asynchronously

```csharp
using System.IO;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        string path = "output.txt";

        // Async write — string content
        await File.WriteAllTextAsync(path, "Hello, async world!");

        // Async write — lines
        string[] lines = { "Line 1", "Line 2", "Line 3" };
        await File.WriteAllLinesAsync(path, lines);

        // Async append
        await File.AppendAllTextAsync(path, "\nAppended line.");
    }
}
```

---

## 💻 Advanced: StreamReader / StreamWriter for Large Files

For large files, use streams to avoid loading everything into memory at once:

```csharp
// Reading large file line-by-line (memory efficient)
static async Task ReadLargeFileAsync(string path)
{
    using var reader = new StreamReader(path);
    string? line;

    while ((line = await reader.ReadLineAsync()) != null)
    {
        Console.WriteLine(line);
        // Process line-by-line — never loads whole file into RAM
    }
}

// Writing large content with buffering
static async Task WriteLargeFileAsync(string path, IEnumerable<string> data)
{
    await using var writer = new StreamWriter(path, append: false);

    foreach (var line in data)
    {
        await writer.WriteLineAsync(line);
    }
    // FlushAsync is called by DisposeAsync (await using)
}
```

---

## 💻 FileStream with Explicit Async Options

For maximum async performance, open `FileStream` with `useAsync: true`:

```csharp
// This enables true OS-level async I/O
await using var fs = new FileStream(
    "bigfile.bin",
    FileMode.Open,
    FileAccess.Read,
    FileShare.Read,
    bufferSize: 4096,
    useAsync: true        // ← enables IOCP on Windows
);

var buffer = new byte[4096];
int bytesRead;
while ((bytesRead = await fs.ReadAsync(buffer, 0, buffer.Length)) > 0)
{
    // Process buffer...
}
```

---

## ❌ Common Mistake: Sync I/O in Async Context

```csharp
// ❌ BAD — blocks the thread, wastes async benefits
async Task ProcessFileAsync(string path)
{
    string content = File.ReadAllText(path); // ← sync! Blocks thread
    await DoSomethingAsync(content);
}

// ✅ GOOD — truly async
async Task ProcessFileAsync(string path)
{
    string content = await File.ReadAllTextAsync(path); // ← async
    await DoSomethingAsync(content);
}
```

---

## 🆚 Sync vs Async File I/O

| Feature | Sync (`File.ReadAllText`) | Async (`File.ReadAllTextAsync`) |
|---|---|---|
| Thread usage | Blocked during I/O | Free during I/O |
| Scalability | Low (1 thread per operation) | High (thread handles many ops) |
| Use in ASP.NET | ⚠️ Risky (can exhaust thread pool) | ✅ Recommended |
| Use in desktop | Acceptable for small files | ✅ Still better |
| Simple scripts | ✅ Fine | May be overkill |

---

## 📌 Summary

> Async file I/O frees threads while waiting for slow disk operations. Use `File.*Async()` methods for simple cases, `StreamReader/Writer` for large files, and `FileStream` with `useAsync: true` for maximum control. In server applications, always prefer async I/O to avoid thread pool exhaustion.
