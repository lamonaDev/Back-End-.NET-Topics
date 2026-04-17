# Advanced Mapping

---

## ValueConverter

Value converters transform values between the database and the application. Useful for storing complex types as primitive types or encrypting sensitive data.

### Enum to String Conversion

```csharp
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
}

public enum OrderStatus { Pending, Processing, Shipped, Delivered }

// Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .Property(o => o.Status)
        .HasConversion<string>();
}
```

### Custom ValueConverter

```csharp
public class EncryptedConverter : ValueConverter<string, string>
{
    public EncryptedConverter() : base(
        v => Encrypt(v),
        v => Decrypt(v))
    { }

    private static string Encrypt(string value) => Convert.ToBase64String(Encoding.UTF8.GetBytes(value));
    private static string Decrypt(string value) => Encoding.UTF8.GetString(Convert.FromBase64String(value));
}

// Usage
modelBuilder.Entity<User>()
    .Property(u => u.SocialSecurityNumber)
    .HasConversion<EncryptedConverter>();
```

### Collection Serialization

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Tags)
    .HasConversion(
        v => JsonSerializer.Serialize(v, JsonOptions.Default),
        v => JsonSerializer.Deserialize<List<string>>(v, JsonOptions.Default));
```

---

## IsTemporal (Temporal Tables)

Temporal tables automatically track historical data changes, enabling point-in-time queries without manual auditing.

### Configuration

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>()
        .ToTable("Customers", b => b.IsTemporal());
}
```

### Querying Temporal Data

```csharp
// As-of query: data at specific point in time
var customerAtTime = await context.Customers
    .TemporalAsOf(new DateTime(2024, 1, 15))
    .FirstOrDefaultAsync(c => c.Id == 1);

// All changes between dates
var changes = await context.Customers
    .TemporalBetween(
        new DateTime(2024, 1, 1), 
        new DateTime(2024, 1, 31))
    .Where(c => c.Id == 1)
    .ToListAsync();

// Full history
var history = await context.Customers
    .TemporalAll()
    .Where(c => c.Id == 1)
    .OrderBy(c => EF.Property<DateTime>(c, "PeriodStart"))
    .ToListAsync();
```

### Temporal Properties

```csharp
// Access system-versioning columns
var historyWithPeriod = await context.Customers
    .TemporalAll()
    .Select(c => new
    {
        c.Id,
        c.Name,
        PeriodStart = EF.Property<DateTime>(c, "PeriodStart"),
        PeriodEnd = EF.Property<DateTime>(c, "PeriodEnd")
    })
    .ToListAsync();
```

---

## HasQueryFilter (Global Query Filters)

Global query filters automatically apply predicates to all queries for an entity type. Ideal for soft-delete and multi-tenancy scenarios.

### Soft Delete Pattern

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
}

public class Product : ISoftDeletable
{
    public int Id { get; set; }
    public string Name { get; set; }
    public bool IsDeleted { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>()
        .HasQueryFilter(p => !p.IsDeleted);
}
```

### Multi-Tenancy

```csharp
public interface ITenantEntity
{
    string TenantId { get; set; }
}

public class Document : ITenantEntity
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string TenantId { get; set; }
}

public class ApplicationDbContext : DbContext
{
    private readonly string _currentTenantId;

    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options, ITenantProvider tenantProvider) 
        : base(options)
    {
        _currentTenantId = tenantProvider.GetTenantId();
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Document>()
            .HasQueryFilter(d => d.TenantId == _currentTenantId);
    }
}
```

### Bypassing Filters

```csharp
// Include deleted records
var allProducts = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();
```
