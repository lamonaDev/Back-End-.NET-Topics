# EF Core Query Performance

---

## AsNoTracking vs AsNoTrackingWithIdentityResolution

### AsNoTracking

Disables change tracking for read-only scenarios. Improves performance by skipping identity cache and snapshot creation.

```csharp
// Fastest for read-only scenarios where entities don't reference each other
var blogs = await context.Blogs
    .AsNoTracking()
    .ToListAsync();
```

**Trade-off:** Each query returns new instances. Related entities that should be identical become separate objects.

### AsNoTrackingWithIdentityResolution

Provides performance benefits of `AsNoTracking` while maintaining identity resolution for related entities within the query.

```csharp
// Ensures same Post instances for duplicate references
var blogsWithAuthors = await context.Blogs
    .Include(b => b.Posts)
    .ThenInclude(p => p.Author)
    .AsNoTrackingWithIdentityResolution()
    .ToListAsync();
```

**Use when:** Query returns entities that reference the same related entity multiple times and identity matters.

### Comparison

| Feature | Tracking | AsNoTracking | AsNoTrackingWithIdentityResolution |
|---------|----------|--------------|-----------------------------------|
| Change tracking | Yes | No | No |
| Identity resolution | Yes | No | Yes |
| Performance | Baseline | Fastest | Fast |
| Memory usage | Higher | Lowest | Low |

---

## SingleQuery vs SplitQuery

EF Core 5+ supports splitting queries for collection includes to avoid Cartesian explosion.

### Single Query (Default)

```csharp
// Generates single JOIN-heavy query
// Can cause data duplication with multiple collection includes
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Authors)
    .AsSingleQuery() // Explicit, though default
    .ToListAsync();
```

**SQL Generated:** Single query with JOINs. Result set size = blogs × posts × authors.

### Split Query

```csharp
// Generates multiple queries: one per collection include
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Authors)
    .AsSplitQuery()
    .ToListAsync();
```

**SQL Generated:** 
1. Query for blogs
2. Query for posts by blog IDs
3. Query for authors by blog IDs

### Global Configuration

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString, 
            o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
}
```

**Guidelines:**
- Use `SplitQuery` with multiple collection includes
- Use `SingleQuery` for reference-only includes or small datasets
- Split queries require multiple round-trips; balance against data duplication

---

## Bulk Insert Batching and SQL Parameters

EF Core batches multiple insert statements and uses parameterized SQL to optimize performance.

### How EF Core Sends Data

```csharp
// Multiple entities in single SaveChanges
var orders = Enumerable.Range(1, 1000)
    .Select(i => new Order { OrderNumber = $"ORD-{i}", Amount = i * 10 })
    .ToList();

context.Orders.AddRange(orders);
await context.SaveChangesAsync();
```

**Batching behavior:**
- Default batch size: 42 statements (SQL Server provider)
- Batches grouped by entity type
- Uses table-valued parameters for large batches when supported

### Parameterization

```csharp
// Always parameterized - prevents SQL injection
var userInput = "test";
var products = await context.Products
    .Where(p => p.Name == userInput)
    .ToListAsync();

// Generates: SELECT ... WHERE [Name] = @__userInput_0
```

### Configuration

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString, options =>
    {
        options.MaxBatchSize(100);
        options.MinBatchSize(1);
    });
}
```

---

## Compiled Queries

Compiled queries cache the translation from LINQ to SQL, reducing overhead for frequently executed queries.

### Basic Compiled Query

```csharp
private static readonly Func<ApplicationDbContext, int, IAsyncEnumerable<Blog>>
    _compiledQuery = EF.CompileAsyncQuery(
        (ApplicationDbContext context, int id) =>
            context.Blogs.Where(b => b.Id == id));

public async Task<Blog> GetBlogByIdAsync(int id)
{
    await foreach (var blog in _compiledQuery(context, id))
    {
        return blog;
    }
    return null;
}
```

### With Includes

```csharp
private static readonly Func<ApplicationDbContext, int, IAsyncEnumerable<Blog>>
    _compiledQueryWithInclude = EF.CompileAsyncQuery(
        (ApplicationDbContext context, int id) =>
            context.Blogs
                .AsNoTracking()
                .Include(b => b.Posts)
                .Where(b => b.Id == id));
```

### Limitations

- Cannot use `ToListAsync()`, `FirstOrDefaultAsync()` directly on compiled query
- Parameters must be compile-time constants or passed as arguments
- Cannot use anonymous types in projection without defining type
- Skip/Take values must be constants, not computed

**Performance gain:** 5-10% for simple queries, less for complex queries or when query plan caching is active.
