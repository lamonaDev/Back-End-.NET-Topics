# C# String Handling

## Table of Contents
1. [String Pool & Interning](#string-pool--interning)
2. [String Concatenation Performance](#string-vs-stringbuilder)
3. [char vs string Performance](#char-vs-string-performance)
4. [ToLowerInvariant](#tolowerinvariant)
5. [ToBase64String](#tobase64string)

---

## String Pool & Interning

### Overview
.NET automatically stores string literals in a shared pool to reduce memory usage for duplicate strings.

### How String Interning Works
```csharp
// String literals are automatically interned
string a = "hello";
string b = "hello";
string c = "he" + "llo";  // Compile-time concatenation

// All reference the same memory location
Console.WriteLine(ReferenceEquals(a, b));  // True
Console.WriteLine(ReferenceEquals(a, c));  // True

// Runtime-created strings are NOT interned
string d = new string("hello".ToCharArray());
Console.WriteLine(ReferenceEquals(a, d));  // False

// Manual interning
string e = string.Intern(d);
Console.WriteLine(ReferenceEquals(a, e));  // True
```

**Memory Visualization:**
```
String Literal Pool:

Stack:
┌─────────────────────────────────────────────────┐
│ a ──────┐                                       │
│ b ──────┼──┐                                    │
│ c ──────┼──┼──┐                                 │
│ d ──────┼──┼──┼──┐                              │
└─────────┼──┼──┼──┼───────────────────────────────┘
          │  │  │  │
          ↓  ↓  ↓  ↓
Heap:
┌─────────────────────────────────────────────────┐
│ Intern Pool:                                    │
│ ┌─────────────┐                                │
│ │ "hello"     │ ←────── a, b, c point here    │
│ │ Hash: ABC   │                                │
│ └─────────────┘                                │
│                                                 │
│ Separate object:                                │
│ ┌─────────────┐                                │
│ │ "hello"     │ ←────── d points here (new)   │
│ └─────────────┘                                │
└─────────────────────────────────────────────────┘
```

### == vs Equals for Strings

**Code Example:**
```csharp
string s1 = "hello";
string s2 = "hello";
string s3 = new string("hello".ToCharArray());

// Reference equality
Console.WriteLine(ReferenceEquals(s1, s2));  // True (interned)
Console.WriteLine(ReferenceEquals(s1, s3));  // False (different objects)

// Value equality (both check content)
Console.WriteLine(s1 == s2);                 // True
Console.WriteLine(s1 == s3);                 // True
Console.WriteLine(s1.Equals(s3));            // True
Console.WriteLine(string.Equals(s1, s3));    // True
```

**Comparison Table:**

| Method | Behavior | When to Use |
|--------|----------|-------------|
| `==` | Value equality for strings | Default choice |
| `Equals()` | Value equality | Explicit call |
| `ReferenceEquals()` | Same object identity | Intern checking |
| `Compare()` | Culture-aware comparison | Sorting |
| `CompareOrdinal()` | Ordinal comparison | Performance |

**Real-World Example:**
```csharp
public class StringInternDemo
{
    public void ProcessLargeDataset()
    {
        // Without interning: 1M unique "Status" strings
        // With interning: 1 reference to "Active"
        
        var records = LoadRecords();  // 1M records
        var activeCount = records.Count(r => 
            ReferenceEquals(r.Status, "Active"));  // Fast reference check
    }
}

// Custom intern pool for frequently used strings
public static class StringPool
{
    private static readonly ConcurrentDictionary<string, string> _pool = new();
    
    public static string GetOrAdd(string value)
    {
        if (value == null) return null;
        return _pool.GetOrAdd(value, v => v);
    }
}
```

---

## String vs StringBuilder

### Overview
Different strategies for string concatenation with vastly different performance characteristics.

### String Concatenation Problem
```csharp
// Inefficient: Creates new string each iteration
string result = "";
for (int i = 0; i < 1000; i++)
{
    result += $"Item {i}, ";  // New string allocation each time!
}
// Total allocations: ~1000 strings + GC pressure
```

**Memory Visualization:**
```
String concatenation creates chain of objects:

Iteration 1: "" + "Item 0, " = "Item 0, "
Iteration 2: "Item 0, " + "Item 1, " = "Item 0, Item 1, "
...

Heap after 3 iterations:
┌─────────────────────────────────────────────────┐
│ "Item 0, " (orphaned)                           │
│ "Item 0, Item 1, " (orphaned)                   │
│ "Item 0, Item 1, Item 2, " (current)            │
│ "Item 3, " (literal)                            │
└─────────────────────────────────────────────────┘

Orphaned strings become GC pressure
```

### StringBuilder Solution
```csharp
// Efficient: Modifies internal buffer
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
{
    sb.Append($"Item {i}, ");  // Grows internal array
}
string result = sb.ToString();
// Total allocations: ~1 StringBuilder + final string
```

**Memory Visualization:**
```
StringBuilder internal structure:

┌─────────────────────────────────────────────────┐
│ StringBuilder                                   │
│ ├─ _chunkChars: char[16] initially             │
│ ├─ _chunkLength: 0                             │
│ └─ _chunkPrevious: null                         │
└─────────────────────────────────────────────────┘

After Append("Hello"):
┌─────────────────────────────────────────────────┐
│ _chunkChars: ['H','e','l','l','o',...]         │
│ _chunkLength: 5                                 │
└─────────────────────────────────────────────────┘

When full, creates new chunk linked to previous
```

### Performance Comparison

**Benchmark Results:**
```csharp
|          Method |     Mean |    Error |  Gen 0 | Allocated |
|---------------- |---------:|---------:|-------:|----------:|
|    StringConcat | 145.2 us |  2.83 us | 700.00 |   2.79 MB |
| StringBuilder   |   4.1 us |  0.08 us |   2.00 |   8.24 KB |
```

**When to Use Each:**

| Scenario | Use | Why |
|----------|-----|-----|
| Few concatenations (< 5) | `+` or `$""` | Simpler, no overhead |
| Many concatenations | `StringBuilder` | Fewer allocations |
| Loops | `StringBuilder` | Predictable performance |
| LINQ/fluent | `string.Join` | Functional style |
| Fixed size known | `StringBuilder(capacity)` | Avoid resizing |

**Real-World Example:**
```csharp
public class CsvExporter
{
    public string ExportToCsv(List<Order> orders)
    {
        // Pre-size for performance
        var sb = new StringBuilder(orders.Count * 100);
        
        // Header
        sb.AppendLine("OrderId,Customer,Date,Total");
        
        // Rows
        foreach (var order in orders)
        {
            sb.Append(order.Id).Append(',')
              .Append(order.CustomerName).Append(',')
              .Append(order.Date.ToString("yyyy-MM-dd")).Append(',')
              .Append(order.Total.ToString("F2"))
              .AppendLine();
        }
        
        return sb.ToString();
    }
}
```

---

## char vs string Performance

### Overview
Single character vs single-character string have different memory and performance characteristics.

### Memory Comparison
```csharp
// char - Value type, 2 bytes, stack (usually)
char c = 'A';

// string - Reference type, ~20+ bytes, heap
string s = "A";
```

**Memory Visualization:**
```
char on stack:
┌─────────────────────────────────────────────────┐
│ c: char (2 bytes)                              │
│ Value: 0x0041 ('A')                             │
└─────────────────────────────────────────────────┘

string on heap:
┌─────────────────────────────────────────────────┐
│ s (reference on stack)                          │
│   │                                             │
│   ↓                                             │
│ ┌─────────────────┐                            │
│ │ Object header   │ 8-16 bytes                 │
│ │ Method table ptr│                            │
│ │ Length: 1       │ 4 bytes                    │
│ │ Chars: ['A']    │ 2 bytes + padding          │
│ └─────────────────┘                            │
│ Total: ~24 bytes vs 2 bytes for char           │
└─────────────────────────────────────────────────┘
```

### Performance in Operations
```csharp
// Character comparison - fast
if (char.IsLetter(c)) { }  // Direct value check

// String comparison - slower  
if (s.Length == 1 && char.IsLetter(s[0])) { }

// String operations with chars
var sb = new StringBuilder();
sb.Append('A');      // Single char append
sb.Append("A");        // String append (boxing, more overhead)
```

**Real-World Example:**
```csharp
// Text parsing - use char for performance
public int CountWords(string text)
{
    int count = 0;
    bool inWord = false;
    
    foreach (char c in text)  // char iteration
    {
        if (char.IsWhiteSpace(c))
        {
            if (inWord)
            {
                count++;
                inWord = false;
            }
        }
        else
        {
            inWord = true;
        }
    }
    
    if (inWord) count++;
    return count;
}

// Avoid: Split creates string array
public int CountWordsSlow(string text) => text.Split(' ').Length;
```

---

## ToLowerInvariant

### Overview
Culture-independent lowercase conversion for consistent, portable results.

### The Problem with ToLower()
```csharp
// Turkish "I" problem
var turkishCulture = new CultureInfo("tr-TR");
string name = "Istanbul";

// In Turkish: dotless I becomes dotted i
Console.WriteLine(name.ToLower(turkishCulture));  // "istanbul" (dot on i)

// Invariant: consistent result
Console.WriteLine(name.ToLowerInvariant());       // "istanbul" (no dot)
```

**Code Example:**
```csharp
public class CaseInsensitiveComparer
{
    // ❌ Wrong: culture-specific
    public bool AreEqualWrong(string a, string b) =
        a.ToLower() == b.ToLower();
    
    // ✅ Correct: culture-invariant
    public bool AreEqual(string a, string b) =
        a.Equals(b, StringComparison.OrdinalIgnoreCase);
    
    // ✅ Also correct: explicit invariant
    public string Normalize(string input) =
        input.ToLowerInvariant();
}

// File system operations
public string GetSafeFileName(string name)
{
    // Invariant ensures same result across cultures
    return name.ToLowerInvariant()
              .Replace(' ', '-')
              .Replace('/', '_');
}
```

**When to Use:**

| Method | Use When |
|--------|----------|
| `ToLowerInvariant()` | File names, URLs, identifiers |
| `ToLower()` | User-facing text |
| `StringComparison.OrdinalIgnoreCase` | General comparisons |
| `CultureInfo.CurrentCulture` | Display formatting |

**Real-World Example:**
```csharp
public class RouteNormalizer
{
    public string NormalizeRoute(string controller, string action)
    {
        // Routes should be culture-invariant
        var normalized = $"{controller}/{action}";
        return normalized.ToLowerInvariant();
    }
    
    // /Home/Index same in all cultures
    // /home/index (normalized)
}
```

---

## ToBase64String

### Overview
Encodes binary data as ASCII text for safe transmission in text-based protocols.

### How Base64 Works
- Takes 3 bytes (24 bits) → 4 Base64 characters
- Uses 64-character alphabet (A-Z, a-z, 0-9, +, /)
- Padding with = for leftover bytes

**Code Example:**
```csharp
// Encode binary to Base64
byte[] data = Encoding.UTF8.GetBytes("Hello World");
string base64 = Convert.ToBase64String(data);
// Result: "SGVsbG8gV29ybGQ="

// Decode Base64 to binary
byte[] decoded = Convert.FromBase64String(base64);
string text = Encoding.UTF8.GetString(decoded);
// Result: "Hello World"

// File encoding
byte[] fileBytes = File.ReadAllBytes("image.png");
string fileBase64 = Convert.ToBase64String(fileBytes);

// URL-safe Base64 (replace + and /)
string urlSafe = base64.Replace('+', '-').Replace('/', '_').TrimEnd('=');
```

**Memory Visualization:**
```
Binary to Base64 conversion:

Input: "Hello" (5 bytes = 40 bits)

┌─────────────────────────────────────────────────┐
│ H     e     l     l     o                      │
│ 72    101   108   108   111                    │
│ 01001000 01100101 01101100 01101100 01101111   │
└─────────────────────────────────────────────────┘
                  ↓
Grouped in 6-bit chunks:
010010 000110 010101 101100 011011 000110 1111
   S      G     V     s     b     G     k

Output: "SGVsbG8="
```

**Use Cases:**
1. **HTTP headers**: Binary data in JSON
2. **Email attachments**: MIME encoding
3. **Data URIs**: Inline images in HTML
4. **API tokens**: Compact binary encoding
5. **Storage**: Store binary in text columns

**Real-World Example:**
```csharp
public class ApiTokenService
{
    public string GenerateToken(int userId)
    {
        var payload = new byte[16];
        RandomNumberGenerator.Fill(payload);
        
        // Combine user ID with random bytes
        var tokenData = BitConverter.GetBytes(userId)
            .Concat(payload)
            .ToArray();
        
        // URL-safe Base64
        return Convert.ToBase64String(tokenData)
            .Replace('+', '-')
            .Replace('/', '_');
    }
    
    public int? ValidateToken(string token)
    {
        try
        {
            var base64 = token.Replace('-', '+').Replace('_', '/');
            var data = Convert.FromBase64String(base64);
            return BitConverter.ToInt32(data, 0);
        }
        catch
        {
            return null;
        }
    }
}
```

---

*Source: C# string documentation, performance optimization guides, and text encoding standards.*
