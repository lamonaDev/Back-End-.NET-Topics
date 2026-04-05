# Performance and Memory Management in C#

## Table of Contents
1. [Span<T> and ReadOnlySpan<T>](#spant-and-readonlyspant)
2. [Ref Structs](#ref-structs)
3. [Object Pooling](#object-pooling)
4. [Time and Space Complexity (Big O)](#time-and-space-complexity-big-o)

---

## Span<T> and ReadOnlySpan<T>

### What is Span<T>?

`Span<T>` is a **stack-only, type-safe view of contiguous memory**. It allows safe, high-performance manipulation of arrays, strings, and unmanaged memory without copying.

### Memory Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPAN<T> MEMORY LAYOUT                           │
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
│   Points to memory (any of these):                               │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  ┌──────────────┐                                       │    │
│   │  │ Managed Heap │ ← Array, String data                  │    │
│   │  └──────────────┘                                       │    │
│   │                                                          │    │
│   │  ┌──────────────┐                                       │    │
│   │  │ Stack        │ ← stackalloc, local variables          │    │
│   │  └──────────────┘                                       │    │
│   │                                                          │    │
│   │  ┌──────────────┐                                       │    │
│   │  │ Native Heap  │ ← Marshal.AllocHGlobal, etc.           │    │
│   │  └──────────────┘                                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Usage

```csharp
// From array - no allocation, just a view
byte[] array = new byte[100];
Span<byte> span = array;

// Slice without copying
Span<byte> slice = span.Slice(10, 20);  // bytes 10-29

// Modify through span modifies original array
span[0] = 42;
Console.WriteLine(array[0]);  // 42

// stackalloc for stack-only allocation
Span<int> stackSpan = stackalloc int[10];

// From string (ReadOnlySpan<char>)
string text = "Hello World";
ReadOnlySpan<char> chars = text.AsSpan();
ReadOnlySpan<char> world = chars.Slice(6, 5);  // "World"
```

### Real-World Example: Efficient String Parsing

```csharp
// ❌ BAD: Creates many substring allocations
public void ParseCsvBad(string csv)
{
    var lines = csv.Split('\n');
    foreach (var line in lines)
    {
        var parts = line.Split(',');
        var name = parts[0];      // Allocates new string
        var value = parts[1];     // Allocates new string
        Process(name, value);
    }
}

// ✅ GOOD: Zero allocations with Span
public void ParseCsvGood(ReadOnlySpan<char> csv)
{
    while (!csv.IsEmpty)
    {
        // Find end of line
        ReadOnlySpan<char> line;
        var newLineIndex = csv.IndexOf('\n');
        if (newLineIndex >= 0)
        {
            line = csv.Slice(0, newLineIndex);
            csv = csv.Slice(newLineIndex + 1);
        }
        else
        {
            line = csv;
            csv = ReadOnlySpan<char>.Empty;
        }
        
        // Find comma
        var commaIndex = line.IndexOf(',');
        if (commaIndex >= 0)
        {
            var name = line.Slice(0, commaIndex);    // No allocation!
            var value = line.Slice(commaIndex + 1);    // No allocation!
            ProcessSpan(name, value);
        }
    }
}
```

### Span<T> Restrictions

```csharp
// ❌ Cannot store Span as field
public class MyClass
{
    private Span<byte> _span;  // COMPILE ERROR!
}

// ❌ Cannot box Span
object boxed = span;  // COMPILE ERROR!

// ❌ Cannot use Span in async methods
async Task MethodAsync()
{
    Span<byte> span = stackalloc byte[10];  // COMPILE ERROR!
}

// ✅ Can use in synchronous methods
void MethodSync()
{
    Span<byte> span = stackalloc byte[10];  // OK
}

// ✅ Can pass as parameter
void ProcessSpan(Span<byte> span) { }

// ✅ Can return from methods (with limitations)
ReadOnlySpan<byte> GetData() => new byte[] { 1, 2, 3 };
```

---

## Ref Structs

### What is a Ref Struct?

`ref struct` is a struct that **must live on the stack** and cannot escape to the heap. It can contain references to stack memory.

```
┌─────────────────────────────────────────────────────────────────┐
│                    REF STRUCT CHARACTERISTICS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ref struct MyStruct                                            │
│   {                                                              │
│       public Span<byte> Span;    // ✓ Contains Span (ref struct)  │
│       public int Number;          // ✓ Regular field OK           │
│       // public string Text;    // ✓ Also OK (reference type)    │
│   }                                                              │
│                                                                  │
│   Restrictions:                                                  │
│   • Cannot be boxed (no interface calls)                         │
│   • Cannot be field of class or regular struct                   │
│   • Cannot be static field                                       │
│   • Cannot be captured in lambda/closures                        │
│   • Cannot be used in async/iterator methods                   │
│                                                                  │
│   Why? The struct might contain stack references that would      │
│   become invalid if stored on the heap!                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Custom Ref Struct Example

```csharp
// Custom ref struct for high-performance parsing
public ref struct BufferReader
{
    private readonly ReadOnlySpan<byte> _buffer;
    private int _position;
    
    public BufferReader(ReadOnlySpan<byte> buffer)
    {
        _buffer = buffer;
        _position = 0;
    }
    
    public bool HasMore => _position < _buffer.Length;
    
    public ReadOnlySpan<byte> ReadLine()
    {
        var remaining = _buffer.Slice(_position);
        var newlineIndex = remaining.IndexOf((byte)'\n');
        
        if (newlineIndex < 0)
        {
            // Return rest of buffer
            _position = _buffer.Length;
            return remaining;
        }
        
        var line = remaining.Slice(0, newlineIndex);
        _position += newlineIndex + 1;
        return line;
    }
    
    public int ReadInt32()
    {
        // Parse directly from span - no allocations
        var remaining = _buffer.Slice(_position);
        // ... parsing logic ...
        return value;
    }
}

// Usage - zero allocations
void ProcessBuffer(byte[] data)
{
    var reader = new BufferReader(data);  // Lives on stack
    
    while (reader.HasMore)
    {
        var line = reader.ReadLine();
        ProcessLine(line);  // No allocations!
    }
}
```

---

## Object Pooling

### What is Object Pooling?

Object pooling **reuses objects instead of creating and destroying them**, reducing GC pressure and allocation overhead.

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBJECT POOLING FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WITHOUT POOLING:                                              │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    │
│   │  new()  │───→│  use    │───→│  GC     │───→│destroyed│   │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘    │
│       ↑                                            ↓           │
│       └────────────────────────────────────────────┘           │
│                     (Allocate again)                             │
│                                                                  │
│   WITH POOLING:                                                 │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    │
│   │ Pool.Get│───→│  use    │───→│ Return  │───→│  Pool   │    │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘    │
│       ↑                                            │           │
│       └────────────────────────────────────────────┘           │
│                    (Reuse same object)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ArrayPool<T>

```csharp
// Rent arrays from shared pool
public void ProcessData(int size)
{
    // Rent from pool (no allocation!)
    var rentedArray = ArrayPool<byte>.Shared.Rent(size);
    
    try
    {
        // Use the array
        FillWithData(rentedArray);
        Process(rentedArray);
    }
    finally
    {
        // MUST return to pool!
        ArrayPool<byte>.Shared.Return(rentedArray);
    }
}

// Important: rented array might be larger than requested
// Always use only the portion you requested:
var actualData = rentedArray.AsSpan(0, size);
```

### ObjectPool<T> with Microsoft.Extensions.ObjectPool

```csharp
// Install: Microsoft.Extensions.ObjectPool

// Define policy for object creation/reset
public class StringBuilderPolicy : IPooledObjectPolicy<StringBuilder>
{
    public StringBuilder Create() => new();
    
    public bool Return(StringBuilder obj)
    {
        obj.Clear();  // Reset state before returning to pool
        return true;  // Return to pool (false = discard)
    }
}

// Create pool
var pool = new DefaultObjectPoolProvider().CreateStringBuilderPool();

// Usage
public string BuildMessage()
{
    var sb = pool.Get();  // Get from pool or create new
    try
    {
        sb.Append("Hello");
        sb.Append(" ")
        sb.Append("World");
        return sb.ToString();
    }
    finally
    {
        pool.Return(sb);  // Return to pool for reuse
    }
}
```

### Custom Object Pool

```csharp
public class SimplePool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _pool = new();
    private readonly Func<T> _factory;
    private readonly int _maxSize;
    
    public SimplePool(Func<T> factory, int maxSize = 100)
    {
        _factory = factory;
        _maxSize = maxSize;
    }
    
    public T Rent()
    {
        if (_pool.TryTake(out var item))
            return item;
        
        return _factory();  // Create new if pool empty
    }
    
    public void Return(T item)
    {
        if (_pool.Count < _maxSize)
            _pool.Add(item);
        // Else: let it be GC'd
    }
}

// Usage for expensive objects
var httpClientPool = new SimplePool<HttpClient>(
    () => new HttpClient { Timeout = TimeSpan.FromSeconds(30) },
    maxSize: 10);
```

---

## Time and Space Complexity (Big O)

### Understanding Big O Notation

Big O describes how an algorithm's resource usage scales with input size.

| Notation | Name | Description |
|----------|------|-------------|
| O(1) | Constant | Same time/space regardless of input size |
| O(log n) | Logarithmic | Doubles with each power of 2 increase |
| O(n) | Linear | Grows proportionally with input |
| O(n log n) | Linearithmic | Slightly worse than linear |
| O(n²) | Quadratic | Grows with square of input |
| O(2ⁿ) | Exponential | Grows exponentially |

### Common .NET Operations

```csharp
// Arrays
var array = new int[100];

array[50];           // O(1) - direct index access
array.Length;        // O(1) - stored property

Array.IndexOf(array, 5);      // O(n) - linear search
Array.Sort(array);             // O(n log n) - quicksort/mergesort

// List<T>
var list = new List<int>();

list.Add(5);         // O(1) amortized (sometimes O(n) when resizing)
list.Insert(0, 5);   // O(n) - must shift all elements
list.Remove(5);      // O(n) - linear search + shift
list[50];            // O(1) - direct access
list.Contains(5);    // O(n) - linear search

// Dictionary<K,V> (hash table)
var dict = new Dictionary<int, string>();

dict.Add(1, "a");    // O(1) amortized
dict[1];             // O(1) - hash lookup
dict.ContainsKey(1); // O(1)
dict.Remove(1);      // O(1)

// LINQ
list.Where(x => x > 5);     // O(n) - filters each element
list.OrderBy(x => x);        // O(n log n)
list.FirstOrDefault();       // O(1) if source is IList
list.Count();                // O(1) if source is ICollection
list.Count(x => x > 5);      // O(n) - must enumerate all
```

### Space Complexity

```csharp
// O(1) space - uses fixed amount of extra memory
int Sum(int[] numbers)
{
    int sum = 0;  // Single variable
    foreach (var n in numbers)
        sum += n;
    return sum;
}

// O(n) space - creates new array
int[] Double(int[] numbers)
{
    var result = new int[numbers.Length];  // Grows with input
    for (int i = 0; i < numbers.Length; i++)
        result[i] = numbers[i] * 2;
    return result;
}

// O(n) space - recursion stack
int Factorial(int n)
{
    if (n <= 1) return 1;
    return n * Factorial(n - 1);  // Stack depth = n
}
```

### Amortized Complexity

```csharp
// List<T>.Add is O(1) amortized
// Most calls are O(1), but occasionally O(n) when resizing

// Total cost of n operations: O(n)
// Even though individual operations might be O(n)

var list = new List<int>();
for (int i = 0; i < 1000000; i++)
{
    list.Add(i);  // Mostly O(1), occasionally copies array
}

// Expensive operations are "spread out" so average is constant
```

### Interview Questions

**Q: When should you use Span<T>?**
> When you need high-performance access to contiguous memory without allocations. Great for parsing, network I/O, and working with unmanaged memory.

**Q: Why can't Span<T> be stored as a field?**
> Because it might contain stack references that would become invalid after the stack frame ends. It's a ref struct with strict lifetime rules.

**Q: What happens if you don't return an array to ArrayPool?**
> Memory leak - the array stays referenced by the pool but isn't reused, eventually causing new allocations. Always use try/finally to return.

**Q: What's the difference between ArrayPool and ObjectPool?**
> ArrayPool is for arrays specifically (built-in, fast). ObjectPool is for any object type (requires custom policy for creation/reset).

**Q: When is O(log n) better than O(n)?**> For large datasets - O(log n) grows extremely slowly. Even for billions of items, log₂(n) is only about 30.

**Q: Why is Dictionary lookup O(1)?**> It uses a hash table - computes hash code to find bucket, then checks equality in that small bucket. Collisions handled but statistically constant time.
