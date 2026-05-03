# Interceptors in EF Core

## Overview

**EF Core Interceptors** are powerful hooks that allow you to intercept and modify EF Core operations at various points in the pipeline without changing your application code. They provide a clean way to implement cross-cutting concerns like auditing, soft delete, query logging, and connection management.

## What Are EF Core Interceptors?

Interceptors are hooks that let you intercept and optionally modify EF Core operations — `SaveChanges`, command execution, connection events, and more — **without touching your services or controllers**. They're registered at the `DbContext` level and run transparently.

### Why Use Interceptors?

```
Traditional Approach:                    Interceptor Approach:
┌─────────────────────────┐              ┌─────────────────────────┐
│ Service Layer          │              │ DbContext + Interceptor│
│ ┌─────────────────────┐ │              │ ┌─────────────────────┐ │
│ │ orderService.Add() │ │              │ │ SaveChanges()       │ │
│ │ + SetCreatedAt()   │ │              │ │   └─→ Audit Inter   │ │
│ │ + SetCreatedBy()   │ │              │ │         └─→ Set date │ │
│ │ + LogActivity()    │ │              │ │         └─→ Set user│ │
│ └─────────────────────┘ │              └─────────────────────┘ │
│ ProductService.Add()   │              No service changes needed!│
│ + SetCreatedAt()       │              │
│ + SetCreatedBy()      │              Benefits:                │
│ + LogActivity()       │              - Single place for logic │
└─────────────────────────┘              - No duplicate code       │
                                       - Easy to add/remove      │
                                       - Transparent to callers │
```

### Available Interceptor Types

| Interceptor Type | Intercepts | Common Use |
|------------------|------------|------------|
| ISaveChangesInterceptor | SaveChanges / SaveChangesAsync | Audit columns, soft delete |
| IDbCommandInterceptor | SQL command execution | Query logging, slow query alerts |
| IDbConnectionInterceptor | DB connection events | Connection monitoring |
| IDbTransactionInterceptor | Transaction lifecycle | Distributed transaction hooks |

## Interceptor A — Audit Columns (CreatedAt, UpdatedAt, CreatedBy, UpdatedBy)

Instead of manually setting audit fields in every service method, an interceptor handles this automatically and consistently. This ensures every entity change is audited regardless of which service method triggered it.

### The Audit Problem

```
Without Interceptor (Manual):
┌────────────────────────────────────────────────────────────────┐
│ OrderService.Create()     → SetCreatedAt(), SetCreatedBy()     │
│ ProductService.Create()   → SetCreatedAt(), SetCreatedBy()     │
│ CustomerService.Create()  → SetCreatedAt(), SetCreatedBy()     │
│                                                                │
│ Problems:                                                      │
│ - Easy to forget in one service                                │
│ - Inconsistent implementation                                 │
│ - Duplicate code everywhere                                   │
└────────────────────────────────────────────────────────────────┘

With Interceptor:
┌────────────────────────────────────────────────────────────────┐
│ AuditInterceptor.SavingChanges()                              │
│ → Automatically sets CreatedAt, CreatedBy for ALL entities    │
│                                                                │
│ Benefits:                                                      │
│ - Impossible to forget                                         │
│ - Consistent for all entities                                  │
│ - Single source of truth                                       │
└────────────────────────────────────────────────────────────────┘
```

### IAuditableEntity.cs — marker interface for auditable entities

```csharp
public interface IAuditableEntity
{
    DateTime CreatedAt { get; set; }
    DateTime UpdatedAt { get; set; }
    string? CreatedBy { get; set; }
    string? UpdatedBy { get; set; }
}
```

### AuditInterceptor.cs

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuditInterceptor(IHttpContextAccessor httpContextAccessor)
        => _httpContextAccessor = httpContextAccessor;

    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        ApplyAudit(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        ApplyAudit(eventData.Context);
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private void ApplyAudit(DbContext? context)
    {
        if (context is null) return;

        var currentUser = _httpContextAccessor.HttpContext?
            .User.Identity?.Name ?? "System";
        var now = DateTime.UtcNow;

        foreach (var entry in context.ChangeTracker.Entries<IAuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = now;
                entry.Entity.CreatedBy = currentUser;
                entry.Entity.UpdatedAt = now;
                entry.Entity.UpdatedBy = currentUser;
            }
            else if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAt = now;
                entry.Entity.UpdatedBy = currentUser;
                // Preserve CreatedAt/CreatedBy — don't let updates overwrite them
                entry.Property(e => e.CreatedAt).IsModified = false;
                entry.Property(e => e.CreatedBy).IsModified = false;
            }
        }
    }
}
```

## Interceptor B — Prevent Hard Delete (Soft Delete)

Soft delete patterns mark records as deleted rather than removing them from the database, allowing for data recovery and maintaining referential integrity.

### The Soft Delete Pattern

```
Traditional Delete:                    Soft Delete:
┌─────────────────────────┐            ┌─────────────────────────┐
│ DELETE FROM Products   │            │ UPDATE Products        │
│ WHERE Id = 1           │            │ SET IsDeleted = 1      │
└─────────────────────────┘            │ WHERE Id = 1           │
                                        └─────────────────────────┘
Result: Record is GONE forever!        Result: Record hidden,
                                        but data preserved!
```

### ISoftDeletable.cs

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}
```

### SoftDeleteInterceptor.cs

```csharp
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        ConvertDeleteToSoftDelete(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        ConvertDeleteToSoftDelete(eventData.Context);
        return base.SavingChangesAsync(eventData, result, ct);
    }

    private static void ConvertDeleteToSoftDelete(DbContext? context)
    {
        if (context is null) return;

        var deletedEntries = context.ChangeTracker
            .Entries<ISoftDeletable>()
            .Where(e => e.State == EntityState.Deleted);

        foreach (var entry in deletedEntries)
        {
            entry.State = EntityState.Modified; // cancel the DELETE
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = DateTime.UtcNow;
        }
    }
}
```

### Global Query Filter for Soft Delete

Add this to your DbContext to automatically filter soft-deleted records:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Apply to all entities implementing ISoftDeletable
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        if (typeof(ISoftDeletable).IsAssignableFrom(entityType.ClrType))
        {
            modelBuilder.Entity(entityType.ClrType)
                .HasQueryFilter(e => !((ISoftDeletable)e).IsDeleted);
        }
    }
}
```

## Interceptor C — Command Logging

Monitor and log all SQL commands executed by EF Core:

```csharp
public class CommandLoggingInterceptor : DbCommandInterceptor
{
    private readonly ILogger<CommandLoggingInterceptor> _logger;

    public CommandLoggingInterceptor(ILogger<CommandLoggingInterceptor> logger)
        => _logger = logger;

    public override void CommandExecuting(CommandCommandEventData eventData)
    {
        _logger.LogDebug("Executing SQL: {Command}", eventData.Command.CommandText);
    }

    public override void CommandExecuted(CommandExecutedEventData eventData)
    {
        if (eventData.Duration > TimeSpan.FromSeconds(1))
        {
            _logger.LogWarning(
                "Slow query detected: {Duration}ms - {Command}",
                eventData.Duration.TotalMilliseconds,
                eventData.Command.CommandText);
        }
    }
}
```

## Register Interceptors in Program.cs

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<AuditInterceptor>();
builder.Services.AddSingleton<SoftDeleteInterceptor>();

builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(connectionString);
    options.AddInterceptors(
        sp.GetRequiredService<AuditInterceptor>(),
        sp.GetRequiredService<SoftDeleteInterceptor>());
});
```

## Edge Cases

### 1. Bulk Operations

**Bulk operations** (`ExecuteUpdateAsync`, `ExecuteDeleteAsync`) bypass `SaveChanges` and interceptors entirely — handle separately:

```csharp
// This bypasses soft delete interceptor!
await _context.Products
    .Where(p => p.IsExpired)
    .ExecuteDeleteAsync();

// Must manually handle:
await _context.Products
    .Where(p => p.IsExpired)
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.IsDeleted, true)
        .SetProperty(p => p.DeletedAt, DateTime.UtcNow));
```

### 2. Raw SQL

**Raw SQL** (`ExecuteSqlRaw`) also bypasses interceptors — wrap in explicit logic or avoid for entities with soft delete:

```csharp
// This bypasses interceptors!
await _context.Database
    .ExecuteSqlRawAsync("DELETE FROM Products WHERE Id = {0}", productId);
```

### 3. Background Jobs

**Background jobs** must supply a user identity (e.g., `"System"`) since `IHttpContextAccessor` has no HTTP context outside requests:

```csharp
// In background service
var httpContextAccessor = serviceProvider.GetRequiredService<IHttpContextAccessor>();
httpContextAccessor.HttpContext = new DefaultHttpContext
{
    User = new ClaimsPrincipal(new ClaimsIdentity(
        new[] { new Claim(ClaimTypes.Name, "BackgroundJob") }))
};
```

## Performance Considerations

- Interceptors run on every `SaveChanges` call
- Keep interceptor logic efficient
- Avoid expensive operations (external API calls, file I/O)
- Use `IAsyncEnumerable` for large datasets

## Testing Interceptors

```csharp
[Fact]
public void SaveChanges_ShouldSetCreatedAt()
{
    // Arrange
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseInMemoryDatabase(databaseName: "TestDb")
        .Options;
        
    using var context = new AppDbContext(options);
    
    // Manually set HttpContext for test
    var httpContextAccessor = context.GetService<IHttpContextAccessor>();
    var httpContext = new DefaultHttpContext();
    httpContext.User = new ClaimsPrincipal(new ClaimsIdentity(
        new[] { new Claim(ClaimTypes.Name, "TestUser") }));
    httpContextAccessor.HttpContext = httpContext;
    
    // Act
    context.Products.Add(new Product { Name = "Test" });
    await context.SaveChangesAsync();
    
    // Assert
    var product = context.Products.First();
    Assert.NotEqual(default, product.CreatedAt);
    Assert.Equal("TestUser", product.CreatedBy);
}
```

## References

- [Microsoft Docs — EF Core Interceptors](https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors)
- [EF Core — Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)