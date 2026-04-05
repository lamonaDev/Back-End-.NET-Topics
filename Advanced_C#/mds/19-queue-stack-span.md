# Queue, Stack, Span, and Collections in C#

## Table of Contents
1. [Understanding Queue (FIFO)](#understanding-queue-fifo)
2. [Understanding Stack (LIFO)](#understanding-stack-lifo)
3. [Span&lt;T&gt; and Memory&lt;T&gt;](#spant-and-memoryt)
4. [Collection Performance Comparison](#collection-performance-comparison)
5. [Real-World Use Cases](#real-world-use-cases)
6. [Memory and Performance](#memory-and-performance)
7. [Best Practices and Pitfalls](#best-practices-and-pitfalls)

---

## Understanding Queue (FIFO)

### What is a Queue?

A `Queue<T>` is a **First-In-First-Out (FIFO)** collection where elements are added at the end (enqueue) and removed from the front (dequeue).

### Visual Representation

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUEUE FIFO OPERATIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Initial State:                                                │
│   ┌─────────┐                                                  │
│   │  Front  │ ← Dequeue from here                               │
│   │    ↓    │                                                  │
│   │  [ ]    │                                                  │
│   │  [ ]    │                                                  │
│   │  [ ]    │                                                  │
│   │  Rear   │ ← Enqueue to here                                 │
│   └─────────┘                                                  │
│                                                                  │
│   After Enqueue(10):                                            │
│   ┌─────────┐                                                  │
│   │ Front   │                                                  │
│   │   ↓     │                                                  │
│   │  [10] ← │ Rear now here                                    │
│   │  [ ]    │                                                  │
│   └─────────┘                                                  │
│                                                                  │
│   After Enqueue(20), Enqueue(30):                               │
│   ┌─────────┐                                                  │
│   │ Front   │                                                  │
│   │   ↓     │                                                  │
│   │  [10]   │ ← Will be dequeued first                         │
│   │  [20]   │                                                  │
│   │  [30] ← │ Rear                                            │
│   └─────────┘                                                  │
│                                                                  │
│   After Dequeue() → returns 10:                                 │
│   ┌─────────┐                                                  │
│   │ Front   │                                                  │
│   │   ↓     │                                                  │
│   │  [20]   │ ← Now front                                       │
│   │  [30]   │                                                  │
│   └─────────┘                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Queue Implementation

```csharp
using System.Collections.Generic;

public class QueueDemo
{
    public static void Main()
    {
        // Create queue
        Queue<string> queue = new();
        
        // Enqueue (O(1))
        queue.Enqueue("Customer 1");
        queue.Enqueue("Customer 2");
        queue.Enqueue("Customer 3");
        
        Console.WriteLine($"Count: {queue.Count}"); // 3
        Console.WriteLine($"Peek: {queue.Peek()}"); // Customer 1 (front)
        
        // Dequeue (O(1))
        string first = queue.Dequeue(); // Customer 1
        Console.WriteLine($"Dequeued: {first}");
        Console.WriteLine($"Count: {queue.Count}"); // 2
        
        // Process all items
        while (queue.Count > 0)
        {
            Console.WriteLine(queue.Dequeue());
        }
    }
}
```

### Queue Internal Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUEUE INTERNAL IMPLEMENTATION                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Queue uses a circular array:                                   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Array: [A][B][C][ ][ ][ ]                                │    │
│   │         ↑              ↑                                 │    │
│   │       head             tail (next position)              │    │
│   │                                                          │    │
│   │   After Dequeue():                                      │    │
│   │   Array: [ ][B][C][ ][ ][ ]                              │    │
│   │            ↑  ↑                                          │    │
│   │          head tail                                       │    │
│   │                                                          │    │
│   │   Circular behavior wraps around:                         │    │
│   │   ┌────────────────────────────────────────────┐         │    │
│   │   │ [ ][ ][C][D][E][ ][ ][ ]                  │         │    │
│   │   │      ↑  ↑                                │         │    │
│   │   │     head tail                            │         │    │
│   │   └────────────────────────────────────────────┘         │    │
│   │                                                          │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Enqueue: Add at tail, increment tail (O(1))                  │
│   Dequeue: Return head, increment head (O(1))                  │
│   Peek: Return head without removing (O(1))                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Understanding Stack (LIFO)

### What is a Stack?

A `Stack<T>` is a **Last-In-First-Out (LIFO)** collection where elements are added and removed from the same end (top).

### Visual Representation

```
┌─────────────────────────────────────────────────────────────────┐
│                    STACK LIFO OPERATIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Initial State:                                                │
│   ┌─────────┐                                                  │
│   │  TOP    │ ← Push and Pop from here                          │
│   │   [ ]   │                                                  │
│   └─────────┘                                                  │
│                                                                  │
│   After Push(10):                                               │
│   ┌─────────┐                                                  │
│   │  TOP    │                                                  │
│   │  [10]   │ ← Only element                                    │
│   └─────────┘                                                  │
│                                                                  │
│   After Push(20), Push(30):                                     │
│   ┌─────────┐                                                  │
│   │  TOP    │                                                  │
│   │  [30]   │ ← Will be popped first                            │
│   │  [20]   │                                                  │
│   │  [10]   │                                                  │
│   └─────────┘                                                  │
│                                                                  │
│   After Pop() → returns 30:                                     │
│   ┌─────────┐                                                  │
│   │  TOP    │                                                  │
│   │  [20]   │ ← Now top                                         │
│   │  [10]   │                                                  │
│   └─────────┘                                                  │
│                                                                  │
│   Stack grows DOWN in visualizations, but same end operations!  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Stack Implementation

```csharp
public class StackDemo
{
    public static void Main()
    {
        // Create stack
        Stack<string> stack = new();
        
        // Push (O(1))
        stack.Push("Page 1");
        stack.Push("Page 2");
        stack.Push("Page 3");
        
        Console.WriteLine($"Count: {stack.Count}"); // 3
        Console.WriteLine($"Peek: {stack.Peek()}"); // Page 3 (top)
        
        // Pop (O(1))
        string top = stack.Pop(); // Page 3
        Console.WriteLine($"Popped: {top}");
        
        // Process all (reverse order)
        while (stack.Count > 0)
        {
            Console.WriteLine(stack.Pop());
        }
        // Output: Page 2, then Page 1
    }
}
```

---

## Span<T> and Memory<T>

### What is Span<T>?

`Span<T>` is a **stack-only, type-safe view of contiguous memory**. It allows safe, high-performance manipulation of arrays, strings, and unmanaged memory without copying.

### Span<T> Capabilities

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPAN<T> MEMORY LAYOUT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Span<T> is a ref struct (lives on stack):                   │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  struct Span<T>                                           │    │
│   │  {                                                        │    │
│   │    internal readonly ref T _pointer;  // ByRef-like field│    │
│   │    internal readonly int _length;     // Length of span   │    │
│   │  }                                                        │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Points to memory from:                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  ┌──────────────┐                                       │    │
│   │  │ Managed Heap │ ← Array, String data                  │    │
│   │  └──────────────┘                                       │    │
│   │  ┌──────────────┐                                       │    │
│   │  │ Stack        │ ← stackalloc, local variables          │    │
│   │  └──────────────┘                                       │    │
│   │  ┌──────────────┐                                       │    │
│   │  │ Native Heap  │ ← Marshal.AllocHGlobal                  │    │
│   │  └──────────────┘                                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Span<T> Usage Examples

```csharp
public class SpanExamples
{
    public void SpanBasics()
    {
        // From array (no copy - just a view)
        byte[] array = new byte[100];
        Span<byte> span = array;
        
        // Slice without copying
        Span<byte> slice = span.Slice(10, 20); // bytes 10-29
        
        // Modify through span affects original array
        span[0] = 42;
        Console.WriteLine(array[0]); // 42
        
        // From stackalloc (stack-only allocation)
        Span<int> stackSpan = stackalloc int[10];
        
        // From string (ReadOnlySpan<char>)
        string text = "Hello World";
        ReadOnlySpan<char> chars = text.AsSpan();
        ReadOnlySpan<char> world = chars.Slice(6, 5); // "World"
    }
    
    // Efficient string parsing
    public void ParseCsvLine(ReadOnlySpan<char> line)
    {
        // No allocations - work with spans
        int commaIndex = line.IndexOf(',');
        if (commaIndex > 0)
        {
            ReadOnlySpan<char> name = line.Slice(0, commaIndex);
            ReadOnlySpan<char> value = line.Slice(commaIndex + 1);
            // Process without creating substring objects
        }
    }
}
```

### Memory<T> for Async

```csharp
public class MemoryExamples
{
    // Span can't be used in async methods (ref struct restriction)
    // Use Memory<T> instead
    
    public async Task ProcessAsync(Memory<byte> memory)
    {
        await _stream.ReadAsync(memory); // Async compatible
    }
    
    // Convert between Span and Memory
    public void Conversion()
    {
        Memory<int> memory = new int[10];
        Span<int> span = memory.Span; // Get span from memory
    }
}
```

---

## Collection Performance Comparison

### Time Complexity

```
┌─────────────────────────────────────────────────────────────────┐
│                    COLLECTION PERFORMANCE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Operation          │ List  │ Queue │ Stack │ Dictionary       │
│   ─────────────────────────────────────────────────────────────│
│   Add at end         │ O(1)* │ O(1)  │ O(1)  │ O(1)             │
│   Add at beginning   │ O(n)  │ -     │ -     │ -                │
│   Add at middle      │ O(n)  │ -     │ -     │ -                │
│   Remove from end    │ O(1)  │ -     │ O(1)  │ O(1)             │
│   Remove from front  │ O(n)  │ O(1)  │ -     │ O(1)             │
│   Remove from middle   │ O(n)  │ -     │ -     │ O(1)             │
│   Index access         │ O(1)  │ -     │ -     │ O(1)             │
│   Search (unsorted)    │ O(n)  │ O(n)  │ O(n)  │ O(1)             │
│   Search (sorted)      │ O(logn)│ -     │ -     │ -                │
│                                                                  │
│   * amortized - occasional O(n) resize                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Use Cases

### Queue: Print Spooler

```csharp
public class PrintSpooler
{
    private readonly Queue<PrintJob> _jobs = new();
    private bool _isProcessing;
    
    public void AddJob(PrintJob job)
    {
        _jobs.Enqueue(job);
        if (!_isProcessing)
        {
            _ = ProcessJobsAsync();
        }
    }
    
    private async Task ProcessJobsAsync()
    {
        _isProcessing = true;
        
        while (_jobs.Count > 0)
        {
            var job = _jobs.Dequeue();
            await PrintJobAsync(job);
        }
        
        _isProcessing = false;
    }
    
    private async Task PrintJobAsync(PrintJob job)
    {
        await Task.Delay(1000); // Simulate printing
        Console.WriteLine($"Printed: {job.DocumentName}");
    }
}
```

### Stack: Expression Evaluation

```csharp
public class ExpressionEvaluator
{
    public int Evaluate(string expression)
    {
        // Simple RPN (Reverse Polish Notation) evaluator
        Stack<int> stack = new();
        
        foreach (var token in expression.Split(' '))
        {
            if (int.TryParse(token, out int number))
            {
                stack.Push(number);
            }
            else
            {
                int b = stack.Pop();
                int a = stack.Pop();
                stack.Push(token switch
                {
                    "+" => a + b,
                    "-" => a - b,
                    "*" => a * b,
                    "/" => a / b,
                    _ => throw new InvalidOperationException()
                });
            }
        }
        
        return stack.Pop();
    }
}
// Usage: Evaluate("5 3 + 2 *") returns 16
```

### Span: High-Performance Parsing

```csharp
public class SpanParser
{
    // Zero-allocation CSV parsing
    public void ParseCsv(ReadOnlySpan<byte> csvData)
    {
        while (!csvData.IsEmpty)
        {
            // Find end of line
            int newline = csvData.IndexOf((byte)'\n');
            ReadOnlySpan<byte> line = newline >= 0 
                ? csvData.Slice(0, newline)
                : csvData;
            
            // Process line
            ProcessLine(line);
            
            // Advance
            csvData = newline >= 0 
                ? csvData.Slice(newline + 1)
                : ReadOnlySpan<byte>.Empty;
        }
    }
    
    private void ProcessLine(ReadOnlySpan<byte> line)
    {
        // Parse fields without allocations
        int comma = line.IndexOf((byte)',');
        // ...
    }
}
```

---

## Memory and Performance

### Memory Usage Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY USAGE PATTERNS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   List<T>:                                                      │
│   ├─ Contiguous array (may have unused capacity)               │
│   ├─ Over-allocated to avoid frequent resizes                │
│   └─ Cache-friendly (sequential memory access)                 │
│                                                                  │
│   Queue<T>:                                                     │
│   ├─ Circular array                                            │
│   ├─ Head and tail indices                                     │
│   └─ Automatic growth like List                                │
│                                                                  │
│   Stack<T>:                                                     │
│   ├─ Simple array with top index                               │
│   ├─ LIFO access pattern                                       │
│   └─ Simple implementation, minimal overhead                   │
│                                                                  │
│   Span<T>:                                                      │
│   ├─ Reference to existing memory (no allocation)              │
│   ├─ Stack-only (ref struct)                                   │
│   └─ Zero-copy operations                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Best Practices and Pitfalls

### ✅ Best Practices

```csharp
// 1. Use Queue for FIFO patterns
var queue = new Queue<Task>();

// 2. Use Stack for undo/redo, parsing
var undoStack = new Stack<EditorState>();

// 3. Use Span for high-performance parsing
ReadOnlySpan<char> span = text.AsSpan();

// 4. Pre-allocate when size is known
var queue = new Queue<int>(100);
var stack = new Stack<int>(100);

// 5. Check Count before Peek/Pop/Dequeue
if (stack.Count > 0) { var item = stack.Pop(); }

// 6. Prefer Try methods for safety
if (stack.TryPop(out var item)) { /* use item */ }
```

### ❌ Common Pitfalls

```csharp
// PITFALL 1: Using wrong collection for the pattern
// Need FIFO but use Stack - wrong order!

// PITFALL 2: Modifying Span after source changes
byte[] data = { 1, 2, 3 };
Span<byte> span = data;
data[0] = 10; // span[0] is now 10 too - shared memory!

// PITFALL 3: Storing Span in fields
public class BadClass
{
    private Span<byte> _span; // ❌ Compile error - ref struct
}

// PITFALL 4: Using LINQ on Queue/Stack
queue.Where(x => x > 0); // Works but materializes - inefficient!

// PITFALL 5: Not handling empty collections
stack.Pop(); // InvalidOperationException if empty!

// PITFALL 6: Using stackalloc with large buffers
Span<byte> big = stackalloc byte[1000000]; // ❌ Stack overflow risk
```

---

## Interview Questions

**Q: What's the difference between Queue and Stack?**> Queue is FIFO (First-In-First-Out) - elements are added at the end and removed from the front. Stack is LIFO (Last-In-First-Out) - elements are added and removed from the same end (top).

**Q: When would you use Queue vs Stack?**> Use Queue for: task scheduling, print spooling, breadth-first search, message processing. Use Stack for: undo/redo functionality, expression evaluation, depth-first search, call stack simulation.

**Q: What is Span<T> and when should you use it?**> Span<T> is a stack-only type-safe view of contiguous memory. Use it for high-performance operations that work with arrays, strings, or unmanaged memory without copying. Great for parsing, network I/O, and working with slices of data.

**Q: What's the difference between Span<T> and Memory<T>?**> Span<T> is a ref struct that can only live on the stack - it can't be stored in fields or used in async methods. Memory<T> is a heap-friendly alternative that can be used across async boundaries. You can get a Span from Memory via the .Span property.

**Q: What are the time complexities of Queue operations?**> Enqueue: O(1), Dequeue: O(1), Peek: O(1), Count: O(1). Queue uses a circular array internally for O(1) operations at both ends.

**Q: What are the restrictions on Span<T>?**> Span<T> is a ref struct, so it: (1) Can only live on the stack, (2) Can't be a field of a class, (3) Can't be boxed, (4) Can't be used in async methods, (5) Can't be stored in arrays. These restrictions ensure the referenced memory doesn't outlive its scope.
