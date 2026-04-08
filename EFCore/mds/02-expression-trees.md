# Expression Trees in EF Core

## Overview

Expression trees are data structures that represent code as tree-shaped objects, where each node is an expression (method call, binary operation, constant, etc.). In EF Core, expression trees serve as the bridge between LINQ queries in C# and SQL statements executed against the database.

---

## Core Concepts

### What is an Expression Tree?

An expression tree is a hierarchical representation of lambda expressions that can be inspected, modified, and compiled at runtime.

```
Expression: customer => customer.Age > 21

               LambdaExpression
                     |
        +------------+------------+
        |                         |
   Parameter                GreaterThan
   (customer)                  |
                      +--------+--------+
                      |                 |
                 MemberAccess       Constant
                      |                 |
                  Property: Age       Value: 21
```

### Expression Tree Structure

```
Expression Type Hierarchy:

System.Linq.Expressions.Expression (abstract)
    |
    +-- LambdaExpression (represents lambda)
    |       +-- Body (the expression body)
    |       +-- Parameters (input parameters)
    |
    +-- BinaryExpression (==, !=, <, >, &&, ||)
    +-- MemberExpression (property/field access)
    +-- MethodCallExpression (method invocation)
    +-- ConstantExpression (literal values)
    +-- ParameterExpression (lambda parameters)
    +-- UnaryExpression (!, -, Convert)
```

---

## How EF Core Uses Expression Trees

### Translation Pipeline

```
C# LINQ Query
      |
      v
[Compiler] -> Expression Tree (IQueryable)
      |
      v
[EF Core Query Compiler]
      |
      +---> Expression Tree Visitor (inspects/modifies)
      |
      +---> Query Model (intermediate representation)
      |
      +---> SQL Generation
      |
      v
[Database Provider]
      |
      v
   SQL Command
      |
      v
   Database
```

### Query Translation Process

1. **Expression Building**: LINQ operators build expression trees
2. **Expression Visiting**: EF Core traverses the tree to understand intent
3. **Translation**: Database provider converts expressions to SQL
4. **Execution**: SQL sent to database, results materialized

---

## Real-World Example: Dynamic Query Builder

### Scenario

Build a product search API with multiple optional filters that must execute in the database for performance.

### Implementation

```csharp
public class ProductSearchService
{
    private readonly ApplicationDbContext _context;

    public ProductSearchService(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<PagedResult<Product>> SearchAsync(ProductSearchRequest request)
    {
        // Start with base query (IQueryable - no execution yet)
        IQueryable<Product> query = _context.Products.AsNoTracking();

        // Build expression tree dynamically
        var parameter = Expression.Parameter(typeof(Product), "p");
        Expression predicate = Expression.Constant(true); // Start with TRUE

        // Category filter
        if (!string.IsNullOrEmpty(request.Category))
        {
            var categoryProperty = Expression.Property(parameter, "Category");
            var categoryValue = Expression.Constant(request.Category);
            var categoryEqual = Expression.Equal(categoryProperty, categoryValue);
            predicate = Expression.AndAlso(predicate, categoryEqual);
        }

        // Price range filter
        if (request.MinPrice.HasValue)
        {
            var priceProperty = Expression.Property(parameter, "Price");
            var minPriceValue = Expression.Constant(request.MinPrice.Value);
            var priceGreaterThan = Expression.GreaterThanOrEqual(priceProperty, minPriceValue);
            predicate = Expression.AndAlso(predicate, priceGreaterThan);
        }

        if (request.MaxPrice.HasValue)
        {
            var priceProperty = Expression.Property(parameter, "Price");
            var maxPriceValue = Expression.Constant(request.MaxPrice.Value);
            var priceLessThan = Expression.LessThanOrEqual(priceProperty, maxPriceValue);
            predicate = Expression.AndAlso(predicate, priceLessThan);
        }

        // In-stock filter
        if (request.InStockOnly)
        {
            var stockProperty = Expression.Property(parameter, "StockQuantity");
            var zeroValue = Expression.Constant(0);
            var inStock = Expression.GreaterThan(stockProperty, zeroValue);
            predicate = Expression.AndAlso(predicate, inStock);
        }

        // Create lambda expression
        var lambda = Expression.Lambda<Func<Product, bool>>(predicate, parameter);

        // Apply to query
        query = query.Where(lambda);

        // Apply sorting
        if (!string.IsNullOrEmpty(request.SortBy))
        {
            query = ApplySorting(query, request.SortBy, request.SortDescending);
        }

        // Execute query (materialization point)
        var totalCount = await query.CountAsync();
        var items = await query
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .ToListAsync();

        return new PagedResult<Product>
        {
            Items = items,
            TotalCount = totalCount,
            Page = request.Page,
            PageSize = request.PageSize
        };
    }

    private IQueryable<Product> ApplySorting(
        IQueryable<Product> query, 
        string sortBy, 
        bool descending)
    {
        var parameter = Expression.Parameter(typeof(Product), "p");
        var property = Expression.Property(parameter, sortBy);
        var lambda = Expression.Lambda(property, parameter);

        // Use reflection to call OrderBy/OrderByDescending
        var methodName = descending ? "OrderByDescending" : "OrderBy";
        var method = typeof(Queryable)
            .GetMethods()
            .Where(m => m.Name == methodName && m.GetParameters().Length == 2)
            .First()
            .MakeGenericMethod(typeof(Product), property.Type);

        return (IQueryable<Product>)method.Invoke(null, new object[] { query, lambda });
    }
}

// Usage
public class ProductSearchRequest
{
    public string? Category { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public bool InStockOnly { get; set; }
    public string? SortBy { get; set; }
    public bool SortDescending { get; set; }
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}
```

### Generated SQL Example

```sql
-- Input: Category="Electronics", MinPrice=100, InStockOnly=true

SELECT [p].[ProductId], [p].[Name], [p].[Category], [p].[Price], [p].[StockQuantity]
FROM [Products] AS [p]
WHERE ([p].[Category] = @__category_0) 
  AND ([p].[Price] >= @__minPrice_1) 
  AND ([p].[StockQuantity] > 0)
ORDER BY [p].[Price] DESC
OFFSET @__p_2 ROWS FETCH NEXT @__p_3 ROWS ONLY
```

---

## Expression Tree Visualization

### Simple Filter Expression

```csharp
// Source: p => p.Price > 100 && p.Category == "Electronics"

LambdaExpression
├── Parameters
│   └── p (ParameterExpression: Product)
└── Body
    └── AndAlso (BinaryExpression)
        ├── Left
        │   └── GreaterThan (BinaryExpression)
        │       ├── Left
        │       │   └── MemberAccess: p.Price
        │       └── Right
        │           └── Constant: 100
        └── Right
            └── Equal (BinaryExpression)
                ├── Left
                │   └── MemberAccess: p.Category
                └── Right
                    └── Constant: "Electronics"
```

### Method Call Expression

```csharp
// Source: p => p.Name.StartsWith("A") && p.CreatedDate > date

LambdaExpression
├── Parameters
│   └── p (ParameterExpression: Product)
└── Body
    └── AndAlso (BinaryExpression)
        ├── Left
        │   └── MethodCallExpression: StartsWith
        │       ├── Object
        │       │   └── MemberAccess: p.Name
        │       └── Arguments
        │           └── Constant: "A"
        └── Right
            └── GreaterThan (BinaryExpression)
                ├── Left
                │   └── MemberAccess: p.CreatedDate
                └── Right
                    └── Constant: 2024-01-01
```

---

## Custom Expression Builder

### Specification Pattern Implementation

```csharp
// Abstract specification using expression trees
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
    {
        var predicate = ToExpression().Compile();
        return predicate(entity);
    }

    // AND operator
    public Specification<T> And(Specification<T> other)
    {
        return new AndSpecification<T>(this, other);
    }

    // OR operator
    public Specification<T> Or(Specification<T> other)
    {
        return new OrSpecification<T>(this, other);
    }

    // NOT operator
    public Specification<T> Not()
    {
        return new NotSpecification<T>(this);
    }
}

// Concrete specifications
public class PriceRangeSpecification : Specification<Product>
{
    private readonly decimal _minPrice;
    private readonly decimal _maxPrice;

    public PriceRangeSpecification(decimal minPrice, decimal maxPrice)
    {
        _minPrice = minPrice;
        _maxPrice = maxPrice;
    }

    public override Expression<Func<Product, bool>> ToExpression()
    {
        return p => p.Price >= _minPrice && p.Price <= _maxPrice;
    }
}

public class InStockSpecification : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
    {
        return p => p.StockQuantity > 0;
    }
}

// Composite specification implementations
public class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left;
    private readonly Specification<T> _right;

    public AndSpecification(Specification<T> left, Specification<T> right)
    {
        _left = left;
        _right = right;
    }

    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();

        // Combine expressions using visitor pattern
        var parameter = Expression.Parameter(typeof(T), "x");
        var leftVisitor = new ReplaceParameterVisitor(leftExpr.Parameters[0], parameter);
        var rightVisitor = new ReplaceParameterVisitor(rightExpr.Parameters[0], parameter);

        var leftBody = leftVisitor.Visit(leftExpr.Body);
        var rightBody = rightVisitor.Visit(rightExpr.Body);

        var combined = Expression.AndAlso(leftBody, rightBody);
        return Expression.Lambda<Func<T, bool>>(combined, parameter);
    }
}

// Parameter replacement visitor
public class ReplaceParameterVisitor : ExpressionVisitor
{
    private readonly ParameterExpression _oldParameter;
    private readonly ParameterExpression _newParameter;

    public ReplaceParameterVisitor(
        ParameterExpression oldParameter, 
        ParameterExpression newParameter)
    {
        _oldParameter = oldParameter;
        _newParameter = newParameter;
    }

    protected override Expression VisitParameter(ParameterExpression node)
    {
        return node == _oldParameter ? _newParameter : base.VisitParameter(node);
    }
}

// Usage with EF Core
public class ProductRepository
{
    private readonly ApplicationDbContext _context;

    public async Task<IEnumerable<Product>> FindAsync(Specification<Product> spec)
    {
        return await _context.Products
            .Where(spec.ToExpression())
            .ToListAsync();
    }
}

// Business logic layer
public class ProductService
{
    private readonly ProductRepository _repository;

    public async Task<IEnumerable<Product>> GetAvailableProductsInPriceRange(
        decimal min, 
        decimal max)
    {
        var priceSpec = new PriceRangeSpecification(min, max);
        var stockSpec = new InStockSpecification();
        
        // Combine specifications
        var combinedSpec = priceSpec.And(stockSpec);
        
        return await _repository.FindAsync(combinedSpec);
    }
}
```

---

## Common Pitfalls

### Client Evaluation vs Server Evaluation

```csharp
// BAD: Forces client evaluation (materializes entire table)
var products = await _context.Products
    .Where(p => CustomMethod(p.Price)) // Cannot translate to SQL
    .ToListAsync();

// GOOD: Expression is translatable to SQL
var products = await _context.Products
    .Where(p => p.Price > 100 && p.Price < 500)
    .ToListAsync();

// WORKAROUND: Split query if client evaluation needed
var query = _context.Products
    .Where(p => p.Category == "Electronics") // Server-side
    .AsEnumerable() // Materialize here
    .Where(p => CustomMethod(p.Price)); // Client-side
```

### Null Checks in Expressions

```csharp
// Problem: Null propagation not supported in EF Core 6
var query = _context.Products
    .Where(p => p.Category?.Name == "Electronics"); // May fail

// Solution: Explicit null check
var query = _context.Products
    .Where(p => p.Category != null && p.Category.Name == "Electronics");

// EF Core 8+ supports complex null handling
```

---

## Performance Considerations

| Aspect | Impact | Recommendation |
|--------|--------|----------------|
| Expression Compilation | One-time cost | Cache compiled delegates for reuse |
| Tree Traversal | O(n) where n = nodes | Keep expressions simple |
| Translation Complexity | Increases with operators | Limit nested expressions |
| Parameterization | Automatic | Use variables, not literals |

---

## Summary

Expression trees are fundamental to EF Core's operation, enabling compile-time type safety for database queries. Understanding their structure and behavior allows developers to build dynamic query capabilities, implement sophisticated filtering patterns, and debug translation issues effectively.
