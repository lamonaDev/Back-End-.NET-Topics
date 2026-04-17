# EF Core Query Translation and Debugging

---

## Client vs Server Evaluation

EF Core evaluates queryable operations on the database server when possible. Operations that cannot be translated execute in memory (client evaluation).

### Server Evaluation (Preferred)

```csharp
// Translated to SQL WHERE clause
var products = await context.Products
    .Where(p => p.Price > 100)
    .ToListAsync();

// Translated to SQL ORDER BY
var sorted = await context.Products
    .OrderBy(p => p.Name)
    .ToListAsync();
```

### Client Evaluation (Caution Required)

```csharp
// Method cannot be translated to SQL
public static bool IsPremium(decimal price) => price > 1000;

// Client evaluation warning or exception (EF Core 3+)
var products = await context.Products
    .Where(p => IsPremium(p.Price)) // Throws: could not be translated
    .ToListAsync();

// Solution: Pull to memory first (use sparingly)
var products = await context.Products
    .ToListAsync() // Execute query
    .Where(p => IsPremium(p.Price)); // Filter in memory
```

### Strategies to Avoid Client Evaluation

```csharp
// Inline the logic for translation
var products = await context.Products
    .Where(p => p.Price > 1000)
    .ToListAsync();

// Use EF.Functions
var products = await context.Products
    .Where(p => EF.Functions.Like(p.Name, "A%"))
    .ToListAsync();
```

---

## LINQ Queries That EF Core Cannot Translate

### Common Untranslatable Patterns

| Pattern | Why It Fails | Alternative |
|---------|--------------|-------------|
| Custom methods in filter | Method body unknown | Inline logic or client eval |
| `ToString()` with formatting | Complex formatting logic | Simple concat or client eval |
| `DateTime.ToString("format")` | Format string varies by culture | Use date parts (Year, Month, Day) |
| `List<T>.Contains()` with complex objects | Object equality not mapped | Use primitive key comparison |
| `String.Split()` | No SQL equivalent | Parse before query or client eval |
| `Enum.HasFlag()` | Bitwise operations limited | Compare specific values |
| `Regex.IsMatch()` | Regex not supported in SQL | Use `EF.Functions.Like` or client eval |
| Constructor with logic | New expression limitation | Use projection with object initializer |

### Examples

```csharp
// FAILS: Custom method
bool IsMatch(string name) => name.StartsWith("A") && name.Length > 5;
context.Products.Where(p => IsMatch(p.Name)); // Throws

// WORKS: Inline
context.Products.Where(p => p.Name.StartsWith("A") && p.Name.Length > 5);

// FAILS: Complex DateTime formatting
context.Orders.Where(o => o.CreatedAt.ToString("yyyy-MM") == "2024-01"); // Throws

// WORKS: Use date parts
context.Orders.Where(o => o.CreatedAt.Year == 2024 && o.CreatedAt.Month == 1);
```

---

## ToQueryString

Retrieves the SQL that EF Core would execute without running the query. Useful for debugging and logging.

```csharp
var query = context.Blogs
    .Where(b => b.Posts.Count > 5)
    .OrderBy(b => b.Url);

string sql = query.ToQueryString();
Console.WriteLine(sql);
```

**Output:**
```sql
SELECT [b].[Id], [b].[Url]
FROM [Blogs] AS [b]
WHERE (
    SELECT COUNT(*)
    FROM [Posts] AS [p]
    WHERE [p].[BlogId] = [b].[Id]) > 5
ORDER BY [b].[Url]
```

### With Parameters

```csharp
var id = 5;
var query = context.Posts.Where(p => p.BlogId == id);

// Shows parameterized SQL
Console.WriteLine(query.ToQueryString());
```

---

## TagWith

Adds comments to generated SQL for correlation with application code.

### Basic Usage

```csharp
var expensiveProducts = await context.Products
    .TagWith("Query: Expensive products for report")
    .Where(p => p.Price > 1000)
    .ToListAsync();
```

**Generated SQL:**
```sql
-- Query: Expensive products for report

SELECT [p].[Id], [p].[Name], [p].[Price]
FROM [Products] AS [p]
WHERE [p].[Price] > 1000
```

### With Caller Information

```csharp
public static class QueryExtensions
{
    public static IQueryable<T> TagWithSource<T>(this IQueryable<T> source,
        [CallerFilePath] string file = "",
        [CallerLineNumber] int line = 0)
    {
        return source.TagWith($"Source: {file}:{line}");
    }
}

// Usage
var data = await context.Products
    .TagWithSource()
    .Where(p => p.InStock)
    .ToListAsync();
```

---

## EnableDetailedErrors

Provides detailed exception messages in development, showing specific entity and property causing issues.

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    if (Environment.IsDevelopment())
    {
        optionsBuilder.EnableDetailedErrors();
    }
}
```

**Without:** `Database operation expected to affect 1 row(s) but actually affected 0 row(s).`

**With:** Shows entity type, key values, and property causing the concurrency conflict.

---

## Logging in .NET (LogLevel and Filtering)

Configure EF Core query logging through standard .NET logging.

### appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.EntityFrameworkCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

### Programmatic Configuration

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging(); // Include parameter values (dev only)
}
```

### Log Levels

| Level | Use Case |
|-------|----------|
| Debug | EF Core internals, query compilation |
| Information | SQL commands executed |
| Warning | Warnings (client evaluation, query optimization hints) |
| Error | Database errors |
| Critical | Unrecoverable errors |

### Filtering by Category

```csharp
optionsBuilder.LogTo(
    Console.WriteLine,
    new[] { 
        DbLoggerCategory.Database.Command.Name,
        DbLoggerCategory.Query.Name 
    },
    LogLevel.Information);
```
