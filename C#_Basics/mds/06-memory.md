# C# Memory Management

## Table of Contents
1. [Span<T> and ReadOnlySpan<T>](#spant-and-readonlyspant)
2. [unsafe in C#](#unsafe-in-c)
3. [Release Mode vs Debug Mode](#release-mode-vs-debug-mode)

---

## Span<T> and ReadOnlySpan<T>

### Overview
`Span<T>` provides a type-safe, memory-safe view of contiguous memory without allocating on the managed heap.

### What is Span?
A ref struct that points to memory anywhere: stack, heap, or unmanaged.

**Key Characteristics:**
- Cannot be stored on the heap (ref struct)
- Works with arrays, stack memory, or unmanaged memory
- Zero-allocation slicing
- Bounds-checked for safety

**Code Example:**
```csharp
// Basic Span usage
byte[] data = new byte[100];
Span<byte> span = data;           // View entire array
Span<byte> slice = span.Slice(10, 20);  // View portion (no allocation!)

// Modifying through Span affects original array
span[0] = 42;
Console.WriteLine(data[0]);  // 42

// String as ReadOnlySpan (zero-allocation)
string text = "Hello World";
ReadOnlySpan<char> chars = text.AsSpan();
ReadOnlySpan<char> word = chars.Slice(0, 5);  // "Hello"

// Span from stackalloc (safe stack memory)
Span<byte> stackBuffer = stackalloc byte[256];
ProcessBuffer(stackBuffer);
```

**Memory Visualization:**
```
Array and Span relationship:

Heap:
┌─────────────────────────────────────────────────┐
│ byte[100] array                                 │
│ ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐           │
│ │ 0 │ 1 │ 2 │ 3 │...│   │   │   │   │           │
│ └───┴───┴───┴───┴───┴───┴───┴───┴───┘           │
│ ↑                                               │
│ Array reference points here                     │
└─────────────────────────────────────────────────┘

Stack:
┌─────────────────────────────────────────────────┐
│ Span<byte> span                                  │
│ ├─ _pointer: ───────→ (points to array[0])      │
│ └─ _length: 100                                  │
├─────────────────────────────────────────────────┤
│ Span<byte> slice                                 │
│ ├─ _pointer: ───────→ (points to array[10])      │
│ └─ _length: 20                                   │
│                                                 │
│ Note: slice shares memory, no copy!             │
└─────────────────────────────────────────────────┘
```

### ReadOnlySpan - Immutable View
```csharp
// ReadOnlySpan for string operations (no allocations)
string csv = "John,25,Engineer";
ReadOnlySpan<char> csvSpan = csv.AsSpan();

// Find comma without creating substrings
int comma = csvSpan.IndexOf(',');
ReadOnlySpan<char> name = csvSpan.Slice(0, comma);  // "John"

// Compare without allocation
if (name.Equals("John".AsSpan(), StringComparison.Ordinal))
{
    // Fast comparison, no string created
}

// Parsing without allocation
ReadOnlySpan<char> ageSpan = csvSpan.Slice(comma + 1, 2);
int age = int.Parse(ageSpan);  // Parses "25" directly
```

**Performance Comparison:**
```csharp
// Traditional (allocations)
string SubstringMethod(string text, int start, int length)
{
    return text.Substring(start, length);  // New string allocation
}

// Span (zero allocation)
ReadOnlySpan<char> SpanMethod(string text, int start, int length)
{
    return text.AsSpan(start, length);  // Just a view, no allocation
}

// Benchmark for 1M operations:
// Substring: Allocates ~50MB, slower
// Span:      Allocates 0 bytes, faster
```

**Real-World Example:**
```csharp
// High-performance CSV parser
public class SpanCsvParser
{
    public void ParseLine(ReadOnlySpan<char> line)
    {
        while (true)
        {
            int comma = line.IndexOf(',');
            if (comma == -1) break;
            
            ReadOnlySpan<char> field = line.Slice(0, comma);
            ProcessField(field);  // Zero allocation
            
            line = line.Slice(comma + 1);
        }
    }
    
    private void ProcessField(ReadOnlySpan<char> field)
    {
        // Work with field directly, no string created
        if (field.Length == 0) return;
        
        // Parse numbers efficiently
        if (int.TryParse(field, out int value))
        {
            Console.WriteLine($"Number: {value}");
        }
    }
}
```

---

## unsafe in C#

### Overview
The `unsafe` context allows direct pointer manipulation, bypassing CLR safety checks for performance-critical scenarios.

### When to Use unsafe
- Interop with native libraries
- Performance-critical algorithms
- Memory-mapped files
- Direct hardware access

**Code Example:**
```csharp
// Enable unsafe code in project settings
public unsafe class UnsafeExamples
{
    // Unsafe block
    public void PointerBasics()
    {
        int number = 42;
        
        unsafe
        {
            int* ptr = &number;  // Get address
            Console.WriteLine($"Value: {*ptr}");     // Dereference: 42
            Console.WriteLine($"Address: {(ulong)ptr:X}");  // Memory address
        }
    }
    
    // Unsafe method
    public unsafe void ProcessArray(int[] data)
    {
        fixed (int* ptr = data)  // Pin array in memory
        {
            for (int i = 0; i < data.Length; i++)
            {
                ptr[i] *= 2;  // Direct memory access
            }
        }
    }
    
    // Stack allocation
    public unsafe void StackAllocExample()
    {
        int* buffer = stackalloc int[100];  // Stack memory, no GC
        
        for (int i = 0; i < 100; i++)
        {
            buffer[i] = i;
        }
    }
}
```

**Memory Visualization:**
```
Safe vs Unsafe memory access:

Safe (managed):
┌─────────────────────────────────────────────────┐
│ int[] array = new int[3] { 1, 2, 3 }           │
│                                                 │
│ Heap:                                           │
│ ┌─────────────────────────────────────┐         │
│ │ Object header | Length | 1 | 2 | 3  │         │
│ └─────────────────────────────────────┘         │
│                                                 │
│ Access: array[0] → bounds check → return 1      │
└─────────────────────────────────────────────────┘

Unsafe (pointer):
┌─────────────────────────────────────────────────┐
│ fixed (int* ptr = array)                        │
│                                                 │
│ ptr → [header][length][ 1 ][ 2 ][ 3 ]           │
│       ↑           ↑                             │
│       │           actual data starts here       │
│       pointer address                           │
│                                                 │
│ Access: ptr[0] → direct memory, no check        │
│ ⚠️ ptr[100] → memory corruption!                │
└─────────────────────────────────────────────────┘
```

### Pointer Types
```csharp
public unsafe class PointerTypes
{
    // Value pointer
    public void ValuePointer()
    {
        int value = 100;
        int* ptr = &value;
        *ptr = 200;  // value is now 200
    }
    
    // Array pointer
    public void ArrayPointer()
    {
        int[] arr = new int[5];
        fixed (int* ptr = arr)
        {
            // Pointer arithmetic
            int* second = ptr + 1;  // Points to arr[1]
            *second = 42;           // arr[1] = 42
        }
    }
    
    // Struct pointer
    public struct Point { public int X, Y; }
    
    public void StructPointer()
    {
        Point p = new Point { X = 10, Y = 20 };
        
        unsafe
        {
            Point* ptr = &p;
            ptr->X = 30;  // Arrow operator for struct members
            (*ptr).Y = 40;  // Equivalent dereference
        }
    }
}
```

**Real-World Example:**
```csharp
// Fast bitmap processing
public unsafe class FastImageProcessor
{
    public void InvertColors(Bitmap bitmap)
    {
        var data = bitmap.LockBits(
            new Rectangle(0, 0, bitmap.Width, bitmap.Height),
            ImageLockMode.ReadWrite,
            PixelFormat.Format32bppArgb);
        
        try
        {
            byte* ptr = (byte*)data.Scan0;
            int bytes = Math.Abs(data.Stride) * bitmap.Height;
            
            for (int i = 0; i < bytes; i++)
            {
                ptr[i] = (byte)(255 - ptr[i]);  // Invert
            }
        }
        finally
        {
            bitmap.UnlockBits(data);
        }
    }
}
```

---

## Release Mode vs Debug Mode

### Overview
Different compilation configurations for development vs production with significant performance implications.

### Debug Mode
**Characteristics:**
- Optimizations disabled
- Full debugging symbols
- Edit and Continue enabled
- Assert statements active
- Bounds checks enabled

```xml
<!-- Debug configuration -->
<PropertyGroup Condition="'$(Configuration)' == 'Debug'">
  <Optimize>false</Optimize>
  <DebugSymbols>true</DebugSymbols>
  <DebugType>full</DebugType>
  <DefineConstants>DEBUG;TRACE</DefineConstants>
</PropertyGroup>
```

### Release Mode
**Characteristics:**
- JIT optimizations enabled
- Inlining
- Loop unrolling
- Dead code elimination
- Smaller binaries

```xml
<!-- Release configuration -->
<PropertyGroup Condition="'$(Configuration)' == 'Release'">
  <Optimize>true</Optimize>
  <DebugSymbols>false</DebugSymbols>
  <DebugType>pdbonly</DebugType>
  <DefineConstants>TRACE</DefineConstants>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

### Performance Differences

```csharp
public class OptimizationDemo
{
    // Method inlining in Release
    public int Add(int a, int b) => a + b;
    
    // Loop unrolling
    public int Sum(int[] numbers)
    {
        int sum = 0;
        for (int i = 0; i < numbers.Length; i++)
        {
            sum += numbers[i];
        }
        return sum;
    }
    // Debug: Each iteration, bounds check
    // Release: Bounds check hoisted, may unroll loop
}

// Debug IL (simplified):
// Load i
// Load array
// Load length
// Compare
// Branch if out of bounds
// Load element
// Add

// Release IL:
// Bounds check once before loop
// Optimized memory access
```

**Benchmark Comparison:**
```csharp
| Method        | Debug     | Release   | Speedup |
|---------------|-----------|-----------|---------|
| Math loop     | 1000 ms   | 150 ms    | 6.7x    |
| Array access  | 500 ms    | 50 ms     | 10x     |
| Method calls  | 200 ms    | 20 ms     | 10x     |
```

### When Each Mode Matters

**Debug Mode:**
- Active development
- Debugging complex issues
- Learning code flow
- Unit test debugging

**Release Mode:**
- Performance testing
- Production deployment
- Benchmarking
- Profiling

**Real-World Example:**
```csharp
// Conditional compilation
public class Logger
{
    [Conditional("DEBUG")]
    public static void Debug(string message)
    {
        Console.WriteLine($"[DEBUG] {message}");
    }
    
    public static void Info(string message)
    {
        Console.WriteLine($"[INFO] {message}");
    }
}

// Usage
Logger.Debug("This only appears in Debug builds");
Logger.Info("This appears in all builds");

// Debug output helper
public class DebugTools
{
    [Conditional("DEBUG")]
    public static void Assert(bool condition, string message)
    {
        if (!condition)
            throw new InvalidOperationException($"Assertion failed: {message}");
    }
}
```

### Common Misconceptions

| Myth | Reality |
|------|---------|
| "Debug is always slower" | True, but sometimes dramatically |
| "Release removes all debugging" | Can include PDBs, just optimized |
| "Asserts should work in production" | Use proper error handling, not asserts |
| "Always develop in Release" | Debug has features like Edit and Continue |

---

*Source: .NET memory model documentation, unsafe code guidelines, and JIT optimization references.*
