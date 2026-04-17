# EF Core Advanced Features

---

## EF Core Interceptors

Interceptors enable cross-cutting concerns like auditing, logging, and security by intercepting database operations.

### Audit Columns Interceptor

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<DbDataReader> result)
    {
        // Log or modify command before execution
        Console.WriteLine($"Executing: {command.CommandText}");
        return base.ReaderExecuting(command, eventData, result);
    }

    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context == null) return await base.SavingChangesAsync(eventData, result, cancellationToken);

        var entries = context.ChangeTracker.Entries<IAuditable>();
        var now = DateTime.UtcNow;
        var user = GetCurrentUser(); // Your user resolution logic

        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
                entry.Entity.CreatedBy = user;
            }

            if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAt = now;
                entry.Entity.UpdatedBy = user;
            }
        }

        return await base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

### Auditable Entity Interface

```csharp
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
    DateTime? UpdatedAt { get; set; }
    string UpdatedBy { get; set; }
}

public class Product : IAuditable
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public string UpdatedBy { get; set; }
}
```

### Registration

```csharp
services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer(connectionString)
           .AddInterceptors(new AuditInterceptor());
});
```

### Interceptor Types

| Interceptor | Purpose |
|-------------|---------|
| `SaveChangesInterceptor` | Before/after SaveChanges |
| `CommandInterceptor` | SQL command execution |
| `ConnectionInterceptor` | Connection open/close |
| `TransactionInterceptor` | Transaction begin/commit/rollback |
| `MaterializationInterceptor` | Entity instance creation |
| `QueryInterceptor` | Query compilation |

---

## Connection Resiliency

Handle transient database failures with automatic retry.

```csharp
services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null);
    });
});
```

### Custom Execution Strategy

```csharp
var executionStrategy = context.Database.CreateExecutionStrategy();

await executionStrategy.ExecuteAsync(async () =>
{
    using var transaction = await context.Database.BeginTransactionAsync();
    
    try
    {
        context.Orders.Add(order);
        context.Inventory.Update(inventory);
        await context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
});
```

---

## Raw SQL Queries

Execute raw SQL when LINQ is insufficient.

### FromSqlRaw

```csharp
var blogs = await context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Url LIKE {0}", "%microsoft%")
    .ToListAsync();
```

### FromSqlInterpolated

```csharp
var searchTerm = "microsoft";
var blogs = await context.Blogs
    .FromSqlInterpolated($"SELECT * FROM Blogs WHERE Url LIKE '%{searchTerm}%'")
    .ToListAsync();
```

### With LINQ Composition

```csharp
var recentPosts = await context.Posts
    .FromSqlRaw("SELECT * FROM Posts WHERE Published = 1")
    .Where(p => p.PublishDate > DateTime.Now.AddDays(-7))
    .OrderByDescending(p => p.ViewCount)
    .ToListAsync();
```

---

## Database Migrations Best Practices

### Managing Migrations

```powershell
# Create migration
Add-Migration InitialCreate

# Apply to database
Update-Database

# Script migrations
Script-Migration -From InitialCreate -To AddAuditColumns

# Remove last migration (if not applied)
Remove-Migration

# Generate idempotent script
Script-Migration -Idempotent -Output "migrations.sql"
```

### Migration Safety

```csharp
public partial class AddIndexSafely : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Online index creation (SQL Server Enterprise)
        migrationBuilder.Sql(@"
            CREATE INDEX IX_Products_Name 
            ON Products(Name) 
            WITH (ONLINE = ON)");
    }
}
```

---

## Model Configuration Patterns

### IEntityTypeConfiguration

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).IsRequired().HasMaxLength(200);
        builder.Property(p => p.Price).HasPrecision(18, 2);
        builder.HasIndex(p => p.Name);
    }
}

// Registration
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new ProductConfiguration());
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(ProductConfiguration).Assembly);
}
```

### Owned Entities

```csharp
public class Order
{
    public int Id { get; set; }
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
}

[Owned]
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string ZipCode { get; set; }
}

// Generates: ShippingAddress_Street, ShippingAddress_City, etc.
```
