# LINQ and Expression Trees

---

## Expression Trees Basics

Expression trees represent code as data structures that can be analyzed, transformed, and compiled at runtime. EF Core uses them to translate LINQ queries to SQL.

### Expression Tree Structure

```csharp
// Lambda expression: x => x.Price > 100
// Represented as Expression<Func<Product, bool>>

Expression<Func<Product, bool>> predicate = p => p.Price > 100;

// Expression tree components:
// - LambdaExpression: Root (parameters + body)
// - ParameterExpression: 'p'
// - BinaryExpression: '>' (GreaterThan)
// - MemberExpression: 'p.Price'
// - ConstantExpression: '100'
```

### Building Expressions Programmatically

```csharp
public static Expression<Func<T, bool>> BuildGreaterThan<T>(string propertyName, object value)
{
    var param = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(param, propertyName);
    var constant = Expression.Constant(value);
    var comparison = Expression.GreaterThan(property, constant);
    
    return Expression.Lambda<Func<T, bool>>(comparison, param);
}

// Usage
var predicate = BuildGreaterThan<Product>("Price", 100);
var expensiveProducts = await context.Products.Where(predicate).ToListAsync();
```

### How EF Core Translates

```csharp
// 1. LINQ Query
var query = context.Products.Where(p => p.Price > 100);

// 2. Expression Tree
// LambdaExpression
//   Parameters: [p]
//   Body: BinaryExpression(GreaterThan)
//     Left: MemberExpression(p.Price)
//     Right: ConstantExpression(100)

// 3. EF Core visits tree and generates: WHERE [Price] > 100
```

### Combining Expressions

```csharp
public static Expression<Func<T, bool>> AndAlso<T>(
    Expression<Func<T, bool>> left,
    Expression<Func<T, bool>> right)
{
    var param = Expression.Parameter(typeof(T), "x");
    var body = Expression.AndAlso(
        Expression.Invoke(left, param),
        Expression.Invoke(right, param));
    
    return Expression.Lambda<Func<T, bool>>(body, param);
}
```

---

## EF.Functions

Database-specific functions that EF Core translates to SQL.

### Like

```csharp
var results = await context.Customers
    .Where(c => EF.Functions.Like(c.Name, "A%")) // Starts with A
    .ToListAsync();

// Generated: WHERE [Name] LIKE 'A%'
```

### DateDiff

```csharp
var recentOrders = await context.Orders
    .Where(o => EF.Functions.DateDiffDay(o.OrderDate, DateTime.Now) < 30)
    .ToListAsync();

// Generated: DATEDIFF(day, [OrderDate], GETDATE()) < 30
```

### Contains (Full-Text Search)

```csharp
var matches = await context.Articles
    .Where(a => EF.Functions.Contains(a.Content, "search term"))
    .ToListAsync();

// Requires full-text index
```

### Collate

```csharp
var caseInsensitive = await context.Products
    .Where(p => EF.Functions.Collate(p.Name, "SQL_Latin1_General_CP1_CI_AI") == "apple")
    .ToListAsync();
```

### FreeText

```csharp
var results = await context.Documents
    .Where(d => EF.Functions.FreeText(d.Body, "machine learning"))
    .ToListAsync();
```

---

## Regular Expressions in C# (Not Translated to SQL)

.NET Regex operations cannot be translated to SQL because databases have different regex dialects or no support.

### What Fails

```csharp
// NOT TRANSLATED: Throws InvalidOperationException
var matches = await context.Customers
    .Where(c => Regex.IsMatch(c.Email, @"^[^@]+@[^@]+$"))
    .ToListAsync();
```

### Workarounds

```csharp
// Option 1: Use Like for simple patterns
var emails = await context.Customers
    .Where(c => EF.Functions.Like(c.Email, "%@%.%"))
    .ToListAsync();

// Option 2: Client evaluation (pull to memory)
var allCustomers = await context.Customers.ToListAsync();
var matches = allCustomers
    .Where(c => Regex.IsMatch(c.Email, @"^[^@]+@[^@]+$"))
    .ToList();

// Option 3: Use database-specific function via EF.Functions
// (requires custom function mapping)
```

### EF.Functions vs Regex

| Use Case | Solution |
|----------|----------|
| Simple wildcards (`%`, `_`) | `EF.Functions.Like` |
| Character classes `[a-z]` | Not supported; use Like or client eval |
| Anchors (`^`, `$`) | Like with `%` patterns |
| Complex patterns | Client evaluation or stored procedure |

### Custom Function Mapping

```csharp
// Map database regex function
[DbFunction("REGEXP", IsBuiltIn = true)]
public static bool RegexMatch(string input, string pattern) => throw new NotSupportedException();

// Usage
var results = await context.Customers
    .Where(c => RegexMatch(c.Email, "^[A-Za-z]+@[A-Za-z]+\\.[A-Za-z]+$"))
    .ToListAsync();
```
