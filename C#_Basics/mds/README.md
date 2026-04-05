# C# Research Documentation

This repository contains comprehensive, professional documentation covering C# language features, .NET fundamentals, and best practices. Each topic is explained with simple code examples, memory visualizations, and real-world scenarios.

## Table of Contents

### Error Handling & Exceptions
1. **[Error Handling](01-error-handling.md)**
   - Throw vs Result Pattern
   - InnerException
   - Custom Exceptions
   - InvalidOperationException

### Language Fundamentals
2. **[Language Fundamentals](02-language-fundamentals.md)**
   - var vs dynamic vs object
   - const vs readonly
   - nameof operator
   - string[] args in Main

### Type System
3. **[Type System Deep Dive](03-type-system.md)**
   - Readonly with Value vs Reference Types
   - Immutable vs Mutable Types
   - Method Tables / VTables
   - Private Fields as readonly

### String Handling
4. **[String Handling](04-strings.md)**
   - String Pool & Interning
   - String Concatenation Performance
   - char vs string Performance
   - ToLowerInvariant
   - ToBase64String

### DateTime and Time Handling
5. **[DateTime and Time](05-datetime.md)**
   - DateTime Basics
   - DateTime Immutability
   - DateTime vs DateTimeOffset vs DateOnly vs TimeOnly

### Memory Management
6. **[Memory Management](06-memory.md)**
   - Span<T> and ReadOnlySpan<T>
   - unsafe in C#
   - Release Mode vs Debug Mode

### Best Practices
7. **[Best Practices](07-best-practices.md)**
   - Early Exit & Guard Clauses
   - Why Returning Null is Bad
   - Parse vs Convert vs TryParse

## File Mapping to Original Research

| Source HTML | Documentation Files |
|-------------|---------------------|
| research 1-ccharp.html (Topics 1-19) | 01-error-handling.md, 02-language-fundamentals.md, 03-type-system.md, 04-strings.md |
| research2.html (Deep Dive Topics) | 05-datetime.md, 06-memory.md, 07-best-practices.md |

## Learning Path

### Beginner
1. Start with **02-language-fundamentals.md** (var, const, nameof)
2. Read **04-strings.md** basics (String Pool)
3. Learn **05-datetime.md** basics

### Intermediate
1. Study **01-error-handling.md** (Throw vs Result)
2. Deep dive into **03-type-system.md** (readonly, immutability)
3. Master **04-strings.md** (StringBuilder, performance)

### Advanced
1. Explore **06-memory.md** (Span, unsafe)
2. Understand **03-type-system.md** (VTables, internals)
3. Apply **07-best-practices.md** patterns

## Key Concepts Summary

### Memory Management
- **Value types** live on stack (fast), **Reference types** on heap
- **Span<T>** provides zero-allocation slicing
- **unsafe** blocks allow pointer manipulation
- **Release mode** enables JIT optimizations

### Type System
- **readonly** prevents reassignment, not mutation (for classes)
- **Immutable types** (string, DateTime) return new instances
- **VTables** enable polymorphic method dispatch
- **Prefer readonly fields** for dependency injection

### String Handling
- **String interning** reduces memory for literals
- **StringBuilder** for concatenation in loops
- **Use Span** for high-performance parsing
- **char** (2 bytes) vs **string** (~24 bytes overhead)

### Error Handling
- **Throw** for exceptional cases
- **Result pattern** for expected failures
- **Guard clauses** for early validation
- **Avoid null returns** - use Result or Try pattern

### Performance
- **TryParse** over Parse with try-catch
- **DateTimeOffset** for cross-timezone apps
- **ToLowerInvariant** for culture-independent operations
- **Debug vs Release** have significant performance differences

## Code Example Patterns

### Guard Clauses
```csharp
public void ProcessOrder(Order order)
{
    if (order == null) throw new ArgumentNullException(nameof(order));
    if (order.Items.Count == 0) throw new ArgumentException("No items");
    
    // Main logic here
}
```

### Result Pattern
```csharp
public Result<User> GetUser(int id)
{
    var user = _repository.Find(id);
    if (user == null)
        return Result<User>.Failure($"User {id} not found");
    return Result<User>.Success(user);
}
```

### Span for Performance
```csharp
public int CountWords(ReadOnlySpan<char> text)
{
    int count = 0;
    bool inWord = false;
    
    foreach (char c in text)
    {
        if (char.IsWhiteSpace(c))
        {
            if (inWord) { count++; inWord = false; }
        }
        else { inWord = true; }
    }
    
    return inWord ? count + 1 : count;
}
```

## Additional Resources

- [C# Language Specification](https://docs.microsoft.com/dotnet/csharp/)
- [.NET Runtime Documentation](https://github.com/dotnet/runtime)
- [Performance Best Practices](https://docs.microsoft.com/dotnet/standard/performance/)
- [Unsafe Code Guidelines](https://docs.microsoft.com/dotnet/csharp/language-reference/unsafe-code/)

---

*Compiled from C# research assignments and coursework materials. Last updated: April 2026*
