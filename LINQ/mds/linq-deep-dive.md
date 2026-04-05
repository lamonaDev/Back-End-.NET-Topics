# C# LINQ Deep Dive — Professional Guide

_A comprehensive exploration of Language Integrated Query internals, patterns, and professional usage._

---

## Table of Contents

1. [The LINQ Philosophy](#1-the-linq-philosophy)
2. [Imperative vs Declarative Programming](#2-imperative-vs-declarative-programming)
3. [Deferred vs Immediate Execution](#3-deferred-vs-immediate-execution)
4. [Streaming vs Non-Streaming Operators](#4-streaming-vs-non-streaming-operators)
5. [Query Syntax vs Method Syntax](#5-query-syntax-vs-method-syntax)
6. [Core Operators Deep Dive](#6-core-operators-deep-dive)
7. [Advanced Operators](#7-advanced-operators)
8. [Performance Considerations](#8-performance-considerations)
9. [LINQ Provider Architecture](#9-linq-provider-architecture)
10. [Professional Patterns](#10-professional-patterns)

---

## 1. The LINQ Philosophy

### What LINQ Actually Is

LINQ is not just a set of extension methods. It is a **query expression framework** that unifies data access patterns across:
- In-memory collections (`IEnumerable<T>`)
- Database queries (`IQueryable<T>`)
- XML (`XElement`)
- Parallel processing (`ParallelQuery<T>`)

```csharp
// Same syntax, different execution contexts
var query1 = list.Where(x => x > 5);           // LINQ to Objects
var query2 = table.Where(x => x.Value > 5);    // LINQ to Entities (SQL)
var query3 = xml.Elements().Where(x => ...);   // LINQ to XML
```

---

## 2. Imperative vs Declarative Programming

### Imperative Approach — The "How"

You describe **step-by-step** how to achieve the result:

```csharp
// Imperative: You control every step
public List<int> GetEvenSquares(List<int> numbers)
{
    var evens = new List<int>();      // Step 1: Create storage
    
    foreach (var n in numbers)       // Step 2: Iterate
    {
        if (n % 2 == 0)              // Step 3: Check condition
        {
            evens.Add(n);             // Step 4: Store matching item
        }
    }
    
    var squares = new List<int>();    // Step 5: Create new storage
    
    foreach (var n in evens)          // Step 6: Iterate again
    {
        squares.Add(n * n);         // Step 7: Transform
    }
    
    squares.Sort((a, b) => b.CompareTo(a)); // Step 8: Sort descending
    
    return squares;
}
```

**Characteristics:**
- Explicit control flow
- Mutable state management
- Step-by-step instructions
- Developer responsible for optimization

### Declarative Approach — The "What"

You describe **what you want**, not how to get it:

```csharp
// Declarative: State the intent, let LINQ handle execution
public IEnumerable<int> GetEvenSquares(List<int> numbers)
{
    return numbers
        .Where(n => n % 2 == 0)    // Filter
        .Select(n => n * n)         // Transform
        .OrderByDescending(n => n);   // Sort
}
```

**Characteristics:**
- Intent is explicit, implementation is abstracted
- Immutable data flow
- Composable pipeline
- LINQ provider optimizes execution

### Comparison Matrix

| Aspect | Imperative | Declarative |
|--------|------------|-------------|
| Focus | How to solve | What to achieve |
| Code Verbosity | High | Low |
| Readability | Step-oriented | Intent-oriented |
| Optimization | Manual | Provider-managed |
| Composability | Hard | Natural |
| Side Effects | Common | Minimized |

---

## 3. Deferred vs Immediate Execution

### Deferred Execution (Lazy Evaluation)

Operators that build a **query plan** without executing it.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Build query (NO execution yet!)
var query = numbers
    .Where(n => { Console.WriteLine($"Filtering {n}"); return n > 2; })
    .Select(n => { Console.WriteLine($"Selecting {n}"); return n * 10; });

Console.WriteLine("Query built, but no filtering happened yet!");

// NOW execution happens
foreach (var n in query)
{
    Console.WriteLine($"Result: {n}");
}

// Output order reveals lazy evaluation:
// Query built, but no filtering happened yet!
// Filtering 1
// Filtering 2
// Filtering 3
// Selecting 3
// Result: 30
// Filtering 4
// Selecting 4
// Result: 40
// Filtering 5
// Selecting 5
// Result: 50
```

**Deferred Operators:**
- `Where`, `Select`, `SelectMany`
- `Take`, `Skip`, `TakeWhile`, `SkipWhile`
- `OrderBy`, `ThenBy`, `OrderByDescending`
- `GroupBy`, `Join`, `GroupJoin`
- `Distinct`, `Except`, `Intersect`, `Union`
- `Reverse`, `Concat`, `Zip`

### Immediate Execution (Greedy Evaluation)

Operators that **execute immediately** and materialize results.

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// ALL filtering and transforming happens NOW
var result = numbers
    .Where(n => { Console.WriteLine($"Filtering {n}"); return n > 2; })
    .Select(n => { Console.WriteLine($"Selecting {n}"); return n * 10; })
    .ToList(); // ← Triggers immediate execution

Console.WriteLine("Results ready");

// Output:
// Filtering 1
// Filtering 2
// Filtering 3
// Selecting 3
// Filtering 4
// Selecting 4
// Filtering 5
// Selecting 5
// Results ready
// Notice: ALL filtering before ANY selecting (different than deferred!)
```

**Immediate Operators:**
- `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup`
- `Count`, `LongCount`, `Any`, `All`
- `First`, `FirstOrDefault`, `Last`, `LastOrDefault`
- `Single`, `SingleOrDefault`
- `ElementAt`, `ElementAtOrDefault`
- `Min`, `Max`, `Sum`, `Average`, `Aggregate`

### Execution Strategy Comparison

```csharp
// Deferred execution benefits: Composability
IQueryable<Product> query = dbContext.Products;

if (userFilter.HasPriceRange)
{
    query = query.Where(p => p.Price >= userFilter.MinPrice && 
                              p.Price <= userFilter.MaxPrice);
}

if (userFilter.HasCategory)
{
    query = query.Where(p => p.CategoryId == userFilter.CategoryId);
}

if (userFilter.SortBy == SortOption.Price)
{
    query = query.OrderBy(p => p.Price);
}

// Single database query with all conditions
var results = query.ToList();
```

---

## 4. Streaming vs Non-Streaming Operators

### Streaming Operators

Process elements **one at a time** without loading entire source.

```csharp
// Streaming: Processes 100M items without loading all into memory
var hugeStream = Enumerable.Range(1, 100_000_000)
    .Where(n => n % 2 == 0)     // Stream: checks one, yields one
    .Select(n => n * n)          // Stream: transforms one at a time
    .Take(10);                   // Stream: stops after 10

// Only 10 elements actually processed!
foreach (var item in hugeStream)
{
    Console.WriteLine(item);
}
```

**Streaming Operators:**
| Category | Operators |
|----------|-----------|
| Filtering | `Where`, `OfType<T>`, `Take`, `Skip`, `TakeWhile`, `SkipWhile` |
| Projection | `Select`, `SelectMany` |
| Ordering | `OrderBy`, `ThenBy`, `OrderByDescending`, `ThenByDescending` |
| Grouping | `GroupBy` |
| Set | `Concat`, `Distinct`, `Except`, `Intersect`, `Union` |
| Conversion | `Cast<T>`, `AsEnumerable` |

### Non-Streaming Operators

Must load **entire source** before producing first result.

```csharp
// Non-streaming: Must see all data to produce first result
var sorted = new[] { 5, 1, 3, 2, 4 }
    .OrderBy(n => n);  // Needs all to sort

// Internally:
// 1. Buffer all elements: [5, 1, 3, 2, 4]
// 2. Sort entire buffer: [1, 2, 3, 4, 5]
// 3. Yield first: 1
```

**Non-Streaming Operators:**
| Category | Operators |
|----------|-----------|
| Ordering | `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending`, `Reverse` |
| Grouping | `GroupBy`, `GroupJoin` |
| Aggregation | `Count`, `Sum`, `Average`, `Min`, `Max`, `Aggregate` |
| Set | `Distinct`, `Except`, `Intersect`, `Union` (with comparer) |
| Terminal | `Last`, `LastOrDefault`, `ToList`, `ToArray` |

### Visual Comparison

```
Streaming (Where + Select):
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Source  │───▶│ Filter  │───▶│ Transform│───▶│ Output  │
│ 1,2,3,4 │    │ (n>2)   │    │ (n*10)  │    │ 30,40   │
└─────────┘    └────┬────┘    └────┬────┘    └─────────┘
      │             │              │
      │ Process 3   │ Filter passes│ Transform
      │ Process 4   │ Filter passes│ Transform
      │ Done        │              │

Non-Streaming (OrderBy):
┌─────────┐    ┌─────────────────┐    ┌─────────┐
│ Source  │───▶│ Buffer + Sort   │───▶│ Output  │
│ 5,1,3,2 │    │ [1,2,3,5]       │    │ 1,2,3,5 │
└─────────┘    └─────────────────┘    └─────────┘
      │        ↑ All must load first!
      │        │ Then sorting occurs
      └────────┘
```

---

## 5. Query Syntax vs Method Syntax

### Method Syntax (Fluent)

Uses extension methods with lambda expressions:

```csharp
var query = customers
    .Where(c => c.Orders.Count > 5)
    .OrderByDescending(c => c.TotalSpent)
    .Select(c => new { c.Name, c.TotalSpent });
```

### Query Syntax (Comprehension)

SQL-like syntax compiled to method calls:

```csharp
var query = from c in customers
            where c.Orders.Count > 5
            orderby c.TotalSpent descending
            select new { c.Name, c.TotalSpent };
```

### Translation Rules

```csharp
// Query Syntax:
from x in source
where condition
select projection

// Compiles to:
source.Where(x => condition).Select(x => projection)

// Multi-clause:
from c in customers
from o in c.Orders  // Multiple from = SelectMany
where o.Total > 100
orderby o.Date
select new { c.Name, o.Total }

// Compiles to:
customers
    .SelectMany(c => c.Orders, (c, o) => new { c, o })
    .Where(x => x.o.Total > 100)
    .OrderBy(x => x.o.Date)
    .Select(x => new { x.c.Name, x.o.Total });
```

### Capabilities Matrix

| Feature | Query Syntax | Method Syntax |
|---------|--------------|---------------|
| `Where`, `Select` | Yes | Yes |
| `OrderBy` + `ThenBy` | Yes (`orderby a, b`) | Yes (chaining) |
| `SelectMany` | Yes (`from ... from`) | Yes |
| `Join` | Yes (`join ... in ... on`) | Yes |
| `GroupJoin` | Yes (`join ... into`) | Yes |
| `GroupBy` | Limited (`group by ... into`) | Full control |
| `Skip`, `Take` | No | Yes |
| `Aggregate` | No | Yes |
| `Distinct` | No | Yes |
| `Zip` | No | Yes |

### Professional Recommendation

```csharp
// Use Query Syntax for:
// - Complex joins with multiple data sources
// - Let clauses for computed values

var query = from c in customers
            join o in orders on c.Id equals o.CustomerId
            join p in products on o.ProductId equals p.Id
            let totalWithTax = o.Quantity * p.Price * 1.08m
            where totalWithTax > 1000
            orderby totalWithTax descending
            select new
            {
                CustomerName = c.Name,
                ProductName = p.Name,
                Total = totalWithTax
            };

// Use Method Syntax for:
// - Simple chains
// - Operations not in query syntax

var topProducts = products
    .Where(p => p.IsActive)
    .OrderByDescending(p => p.Sales)
    .Take(10)
    .ToList();

// Mix both for optimal readability
var query = (from c in customers
             where c.IsActive
             orderby c.CreatedDate descending
             select c)
             .Take(20)
             .ToList();
```

---

## 6. Core Operators Deep Dive

### Where — Filtering with Index

```csharp
// Basic overload
var evens = numbers.Where(n => n % 2 == 0);

// Index overload — access to position
var everyOther = names.Where((name, index) => index % 2 == 0);
// Returns: names[0], names[2], names[4]...

// Practical: Skip first 5 odd numbers
var query = numbers
    .Select((n, i) => new { Value = n, Index = i })
    .Where(x => x.Value % 2 == 1)
    .Skip(5)
    .Select(x => x.Value);
```

### OfType<T> — Safe Type Filtering

```csharp
// Mixed collection
var mixed = new object[] { 1, "two", 3, "four", 5.5, null, "six" };

// OfType filters by runtime type (null-safe)
var strings = mixed.OfType<string>();      // "two", "four", "six"
var ints = mixed.OfType<int>();            // 1, 3
var doubles = mixed.OfType<double>();      // 5.5

// vs Cast<T> (throws on mismatch)
var castStrings = mixed.Cast<string>();    // InvalidCastException at 1!

// Practical: Event handler cleanup
public void HandleEvent(object sender, EventArgs e)
{
    var specificEvents = e.Args
        .OfType<CustomEventArg>()
        .Where(arg => arg.IsImportant)
        .ToList();
}
```

### SelectMany — Flattening Collections

```csharp
// SelectMany flattens nested collections

// Data: Customers with multiple orders
var customers = new[]
{
    new Customer
    {
        Name = "Alice",
        Orders = new[] { new Order { Id = 1 }, new Order { Id = 2 } }
    },
    new Customer
    {
        Name = "Bob",
        Orders = new[] { new Order { Id = 3 } }
    }
};

// Select: IEnumerable<IEnumerable<Order>> (nested)
var nested = customers.Select(c => c.Orders);

// SelectMany: IEnumerable<Order> (flattened)
var flat = customers.SelectMany(c => c.Orders);
// Result: Order 1, Order 2, Order 3

// SelectMany with resultSelector (2-parameter overload)
var customerOrders = customers.SelectMany(
    c => c.Orders,                          // collectionSelector: source of inner sequence
    (c, o) => new { Customer = c.Name, OrderId = o.Id }  // resultSelector: combine outer + inner
);
// Result: { "Alice", 1 }, { "Alice", 2 }, { "Bob", 3 }
```

### SelectMany Overload Comparison

```csharp
// Overload 1: Single collection selector
public static IEnumerable<TResult> SelectMany<TSource, TResult>(
    this IEnumerable<TSource> source,
    Func<TSource, IEnumerable<TResult>> selector);

var flatOrders = customers.SelectMany(c => c.Orders);
// Output: Order, Order, Order...

// Overload 2: Collection + Result selector
public static IEnumerable<TResult> SelectMany<TSource, TCollection, TResult>(
    this IEnumerable<TSource> source,
    Func<TSource, IEnumerable<TCollection>> collectionSelector,
    Func<TSource, TCollection, TResult> resultSelector);

var withCustomer = customers.SelectMany(
    c => c.Orders,                          // For each customer, get orders
    (customer, order) => new { customer.Name, order.Id }  // Combine both
);
// Output: { "Alice", 1 }, { "Alice", 2 }, { "Bob", 3 }

// Practical: Cross join
var sizes = new[] { "S", "M", "L" };
var colors = new[] { "Red", "Blue" };

var combinations = sizes.SelectMany(
    size => colors,
    (size, color) => new { Size = size, Color = color }
);
// Result: S-Red, S-Blue, M-Red, M-Blue, L-Red, L-Blue
```

### OrderBy / ThenBy — Multi-Level Sorting

```csharp
// Single level sorting
var byPrice = products.OrderBy(p => p.Price);

// Multi-level: OrderBy + ThenBy
var sorted = customers
    .OrderBy(c => c.City)           // Primary sort
    .ThenBy(c => c.LastName)         // Secondary sort (when cities equal)
    .ThenBy(c => c.FirstName);        // Tertiary sort

// Descending variants
var sortedDesc = products
    .OrderByDescending(p => p.Category)  // Primary: Category Z-A
    .ThenBy(p => p.Price);                // Secondary: Price low-high

// Stability: LINQ to Objects is STABLE
// (Equal elements maintain relative order)

// Reversing sort order (not data!)
var reverseSorted = sorted.Reverse();

// Practical: Complex business sorting
var prioritized = tickets
    .OrderByDescending(t => t.IsUrgent)
    .ThenBy(t => t.Priority)           // 1, 2, 3...
    .ThenBy(t => t.CreatedDate);       // Oldest first
```

---

## 7. Advanced Operators

### GroupBy — Aggregation Foundation

```csharp
// Basic grouping
var byCategory = products.GroupBy(p => p.Category);
foreach (var group in byCategory)
{
    Console.WriteLine($"Category: {group.Key}");
    foreach (var product in group)
    {
        Console.WriteLine($"  {product.Name}");
    }
}

// With result selector
var categorySummary = products.GroupBy(
    p => p.Category,
    (key, items) => new
    {
        Category = key,
        Count = items.Count(),
        AveragePrice = items.Average(p => p.Price),
        TotalStock = items.Sum(p => p.Stock)
    }
);

// With element selector (transform before grouping)
var nameGroups = people.GroupBy(
    p => p.LastName,
    p => p.FirstName  // Only first names in group
);

// Comparer overload for custom equality
var caseInsensitive = strings.GroupBy(
    s => s,
    StringComparer.OrdinalIgnoreCase
);
```

### Join Operations

```csharp
// Inner Join
var innerJoin = customers.Join(
    orders,                          // inner sequence
    c => c.Id,                       // outer key selector
    o => o.CustomerId,               // inner key selector
    (c, o) => new { Customer = c.Name, OrderId = o.Id }  // result
);

// Group Join (outer join with grouping)
var groupJoin = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, os) => new
    {
        Customer = c.Name,
        OrderCount = os.Count(),
        Orders = os.ToList()
    }
);

// Query syntax equivalents
var innerQuery = from c in customers
                 join o in orders on c.Id equals o.CustomerId
                 select new { c.Name, o.Id };

var groupQuery = from c in customers
                 join o in orders on c.Id equals o.CustomerId into orderGroup
                 select new { c.Name, Count = orderGroup.Count() };
```

### Set Operations

```csharp
var set1 = new[] { 1, 2, 3, 4, 5 };
var set2 = new[] { 4, 5, 6, 7, 8 };

// Distinct (unique values)
var distinct = new[] { 1, 1, 2, 2, 3 }.Distinct();  // 1, 2, 3

// Intersection (common elements)
var intersection = set1.Intersect(set2);  // 4, 5

// Union (all unique from both)
var union = set1.Union(set2);  // 1, 2, 3, 4, 5, 6, 7, 8

// Except (in first, not in second)
var except = set1.Except(set2);  // 1, 2, 3

// With comparer
var distinctByName = products.Distinct(new ProductNameComparer());
```

### Partitioning

```csharp
var numbers = Enumerable.Range(1, 100);

// Take/Skip
var page1 = numbers.Take(20);           // 1-20
var page2 = numbers.Skip(20).Take(20);  // 21-40

// While conditions
var firstBig = numbers.TakeWhile(n => n < 50);  // 1-49
var skipSmall = numbers.SkipWhile(n => n < 50); // 50-100

// Chunk (.NET 6+)
var batches = numbers.Chunk(10);  // 10 arrays of 10 elements

// Pagination helper
public static IEnumerable<T> GetPage<T>(
    this IEnumerable<T> source, int pageNumber, int pageSize)
{
    return source.Skip((pageNumber - 1) * pageSize).Take(pageSize);
}
```

### Aggregation

```csharp
var nums = new[] { 1, 2, 3, 4, 5 };

// Built-in aggregates
int sum = nums.Sum();
double avg = nums.Average();
int min = nums.Min();
int max = nums.Max();
int count = nums.Count();

// With selector
decimal totalPrice = products.Sum(p => p.Price);
int expensiveCount = products.Count(p => p.Price > 100);

// Aggregate (fold/reduce)
int factorial = Enumerable.Range(1, 5)
    .Aggregate((acc, n) => acc * n);  // 120

// Aggregate with seed
var stats = numbers.Aggregate(
    new { Sum = 0, Count = 0, Min = int.MaxValue, Max = int.MinValue },
    (acc, n) => new
    {
        Sum = acc.Sum + n,
        Count = acc.Count + 1,
        Min = Math.Min(acc.Min, n),
        Max = Math.Max(acc.Max, n)
    },
    acc => new
    {
        acc.Sum,
        acc.Count,
        acc.Min,
        acc.Max,
        Average = (double)acc.Sum / acc.Count
    }
);
```

### Element Operations

```csharp
var list = new[] { 10, 20, 30, 40, 50 };

// First/Last (throws if empty)
int first = list.First();              // 10
int firstEven = list.First(n => n % 2 == 0);  // 10
int last = list.Last();                // 50

// OrDefault variants (null-safe)
int? empty = new int[0].FirstOrDefault();  // 0 (default)
int notFound = list.FirstOrDefault(n => n > 100);  // 0

// Single (throws if 0 or >1 match)
var single = list.Single(n => n == 30);  // 30
// list.Single() on empty → InvalidOperationException
// list.Single() on 2+ items → InvalidOperationException

// ElementAt (random access)
int third = list.ElementAt(2);         // 30
int? missing = list.ElementAtOrDefault(100);  // 0

// DefaultIfEmpty (ensures at least one element)
var emptyOrValue = new int[0]
    .DefaultIfEmpty(-1);  // Returns [-1]
```

---

## 8. Performance Considerations

### Materialization Strategies

```csharp
// Multiple enumeration trap
IEnumerable<int> query = numbers.Where(n => n > 5);

// Evaluated 3 times!
if (query.Any()) { }          // 1st enumeration
var first = query.First();     // 2nd enumeration
var list = query.ToList();     // 3rd enumeration

// Solution: Materialize once
var materialized = query.ToList();
if (materialized.Any()) { }   // Uses cached result
```

### List vs Dictionary Lookup

```csharp
// O(n) lookup per item
var query = from o in orders
            where customers.Any(c => c.Id == o.CustomerId)
            select o;

// O(1) lookup with hash
var customerDict = customers.ToDictionary(c => c.Id);
var optimized = orders.Where(o => customerDict.ContainsKey(o.CustomerId));
```

### String Concatenation

```csharp
// BAD: O(n²) string concatenation
var csv = names.Aggregate((a, b) => $"{a},{b}");

// GOOD: O(n) with StringBuilder
var csv = string.Join(",", names);
```

### Deferred Execution Benefits

```csharp
// Efficient: Stop processing early
var firstMatch = hugeList
    .Where(ExpensivePredicate)
    .FirstOrDefault();  // Stops at first match

// vs
var allMatches = hugeList
    .Where(ExpensivePredicate)
    .ToList();          // Processes everything!
```

---

## 9. LINQ Provider Architecture

### IEnumerable<T> vs IQueryable<T>

```csharp
// LINQ to Objects: In-memory, executes in process
IEnumerable<Customer> local = customers.AsEnumerable();
var localQuery = local.Where(c => c.Name.StartsWith("A"));  // In-memory filter

// IQueryable: Expression tree, translated to external query
IQueryable<Customer> remote = dbContext.Customers;
var remoteQuery = remote.Where(c => c.Name.StartsWith("A"));  // Translates to SQL LIKE

// Expression tree inspection
Expression<Func<Customer, bool>> predicate = c => c.Name.StartsWith("A");
// Can be translated to: WHERE Name LIKE 'A%'
```

### Provider Differences

```csharp
// LINQ to Objects: Case-sensitive Contains
var listResult = list.Where(x => x.Name.Contains("test"));

// Entity Framework: Database collation dependent
var dbResult = dbContext.Items.Where(x => x.Name.Contains("test"));
// May translate to: WHERE Name LIKE '%test%'
// Collation determines case sensitivity

// Some operations not translatable
var badQuery = dbContext.Customers
    .Where(c => CustomMethod(c.Name));  // Runtime error!

// Solution: Materialize then process
var workaround = dbContext.Customers
    .ToList()                            // Load to memory
    .Where(c => CustomMethod(c.Name));   // Then apply
```

---

## 10. Professional Patterns

### Repository Pattern with LINQ

```csharp
public interface IRepository<T> where T : class
{
    IQueryable<T> Query();  // Returns IQueryable for composition
    Task<T> GetByIdAsync(int id);
    Task AddAsync(T entity);
}

// Usage: Repository returns IQueryable, caller composes
public async Task<IEnumerable<Order>> GetPendingOrdersForCustomer(
    IRepository<Order> repo, int customerId)
{
    return await repo.Query()
        .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Pending)
        .OrderByDescending(o => o.CreatedDate)
        .Take(10)
        .ToListAsync();
}
```

### Specification Pattern

```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    List<OrderByExpression<T>> OrderBy { get; }
}

public class OverdueOrderSpecification : ISpecification<Order>
{
    public Expression<Func<Order, bool>> Criteria =>
        o => o.DueDate < DateTime.Now && o.Status != OrderStatus.Completed;

    public List<Expression<Func<Order, object>>> Includes =>
        new() { o => o.Customer, o => o.LineItems };

    public List<OrderByExpression<Order>> OrderBy =>
        new() { new(o => o.DueDate, true) };  // descending
}

// Usage
var spec = new OverdueOrderSpecification();
var overdue = await repo.Query()
    .Where(spec.Criteria)
    .Include(spec.Includes)
    .OrderBy(spec.OrderBy)
    .ToListAsync();
```

### Functional Composition

```csharp
public static class QueryExtensions
{
    public static IQueryable<T> ActiveOnly<T>(this IQueryable<T> source)
        where T : IActivatable =>
        source.Where(x => x.IsActive);

    public static IQueryable<T> CreatedInRange<T>(
        this IQueryable<T> source, DateTime from, DateTime to)
        where T : ICreatable =>
        source.Where(x => x.CreatedAt >= from && x.CreatedAt <= to);

    public static IQueryable<T> Page<T>(
        this IQueryable<T> source, int page, int size) =>
        source.Skip((page - 1) * size).Take(size);
}

// Composable queries
var result = await repo.Query()
    .ActiveOnly()
    .CreatedInRange(startDate, endDate)
    .OrderByDescending(x => x.CreatedAt)
    .Page(pageNumber, pageSize)
    .ToListAsync();
```

---

## Quick Reference Card

| Operator | Category | Deferred | Streaming | Syntax |
|----------|----------|----------|-----------|--------|
| Where | Filter | Yes | Yes | Both |
| Select | Project | Yes | Yes | Both |
| SelectMany | Project | Yes | Yes | Both |
| OrderBy | Sort | Yes | No | Both |
| ThenBy | Sort | Yes | No | Query |
| GroupBy | Group | Yes | No | Both |
| Join | Join | Yes | Yes | Both |
| GroupJoin | Join | Yes | Yes | Both |
| Take/Skip | Partition | Yes | Yes | Method |
| First/Last | Element | No | — | Method |
| Single | Element | No | — | Method |
| Count | Aggregate | No | No | Method |
| Any/All | Quantify | No | Yes | Method |
| Distinct | Set | Yes | No | Method |
| Union/Intersect | Set | Yes | No | Method |
| ToList/ToArray | Convert | No | No | Method |

---

*Professional LINQ is about understanding execution semantics, not just syntax. Master deferred execution, respect streaming boundaries, and choose the right syntax for the situation.*
