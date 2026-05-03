# Remove OnConfiguring — Correct DbContext Setup

## Overview

Proper DbContext configuration is critical for building testable, secure, and maintainable ASP.NET Core applications. The `OnConfiguring` method, while convenient for learning, creates significant problems in production environments.

## Why OnConfiguring is Risky

`OnConfiguring()` in your `DbContext` subclass bypasses ASP.NET Core's Dependency Injection and configuration system. This leads to:

- **Hardcoded connection strings**: Credentials live in source control forever
- **No environment switching**: Cannot switch between dev/staging/production without recompiling
- **Impossible unit testing**: Cannot inject mock or in-memory context easily
- **Security risks**: Exposed credentials in code repositories
- **Violates Dependency Inversion Principle**: Database configuration is tightly coupled

### The Problem Visualized

```
Wrong Approach (OnConfiguring):
┌─────────────────────────────────────────────────────────────┐
│  AppDbContext.cs                                            │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ protected override void OnConfiguring(...)           │ │
│  │ {                                                      │ │
│  │     optionsBuilder.UseSqlServer("Server=.;...");    │ │
│  │ }                                                      │ │
│  └───────────────────────────────────────────────────────┘ │
│  - Connection string hardcoded                              │
│  - Cannot change without code modification                 │
│  - Not testable                                            │
└─────────────────────────────────────────────────────────────┘

Correct Approach (DI):
┌─────────────────────────────────────────────────────────────┐
│  Program.cs                                                 │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ services.AddDbContext<AppDbContext>(options =>        │ │
│  │     options.UseSqlServer(configuration.GetConnection   │ │
│  │         ("DefaultConnection")));                      │ │
│  └───────────────────────────────────────────────────────┘ │
│  - Connection from appsettings.json                        │
│  - Environment-based switching                             │
│  - Testable with InMemory provider                         │
└─────────────────────────────────────────────────────────────┘
```

### Wrong Pattern — Before

```csharp
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // Hardcoded, never change this — it's wrong!
        optionsBuilder.UseSqlServer(
            "Server=.;Database=ShopDB;User=sa;Password=P@ssw0rd");
    }
}
```

Problems with this approach:
- Connection string is hardcoded in compiled code — lives in source control forever
- Cannot switch between dev/staging/production databases without recompiling
- Cannot inject mock/in-memory context for unit tests
- Violates the Dependency Inversion Principle

### Correct Pattern — After

#### appsettings.json — correct key location
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=ShopDB;Trusted_Connection=True;TrustServerCertificate=True"
  }
}
```

#### AppDbContext.cs — CORRECT (no OnConfiguring)

```csharp
public class AppDbContext : DbContext
{
    // Options injected via DI — clean, testable, configurable
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

#### Program.cs — register DbContext via DI

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions => sqlOptions.EnableRetryOnFailure(maxRetryCount: 3)
    )
);
```

## Connection Resiliency Options

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
    
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        // Enable retry on transient failures
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null);
        
        // Command timeout
        sqlOptions.CommandTimeout(30);
        
        // Enable sensitive data logging (development only!)
        sqlOptions.EnableSensitiveDataLogging();
    });
});
```

## Before / After Architecture

| Aspect | OnConfiguring (Wrong) | DI + appsettings (Correct) |
|--------|------------------------|----------------------------|
| Connection string location | Hardcoded in C# class | appsettings.json / env var / Key Vault |
| Environment switching | Requires recompile | Automatic via config system |
| Unit testing | Difficult — real DB required | Easy — inject InMemory provider |
| Security | Credentials in source control | Credentials outside repo |
| Cloud deployment | Must change code for each env | Override via Azure App Settings |
| Flexibility | None | Full |

## Testing with In-Memory Provider

```csharp
public class AppDbContextTests
{
    [Fact]
    public async Task CreateOrder_ShouldCreateOrderSuccessfully()
    {
        // Arrange
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;
            
        using var context = new AppDbContext(options);
        
        // Act
        context.Orders.Add(new Order { CustomerId = 1 });
        await context.SaveChangesAsync();
        
        // Assert
        Assert.Equal(1, context.Orders.Count());
    }
}
```

## Production-Ready DbContext Checklist

- **Do**: Place connection string under `ConnectionStrings:DefaultConnection` in appsettings
- **Do**: Use User Secrets for local dev connection strings
- **Do**: Override via environment variable in CI/CD: `ConnectionStrings__DefaultConnection`
- **Do**: Enable retry on failure (`EnableRetryOnFailure`) for transient DB errors in cloud
- **Do**: Use `TrustServerCertificate=True` only in development, not production
- **Do**: Configure appropriate command timeouts
- **Do**: Use Managed Identity in Azure for authentication
- **Don't**: Remove all `OnConfiguring` override if it only calls `UseSqlServer`
- **Don't**: Commit real connection strings to version control
- **Don't**: Use `sa` account or SQL auth in production — use Windows/Managed Identity auth

## Azure Managed Identity Setup

```csharp
// Program.cs - Azure App Service with Managed Identity
builder.Services.AddDbContext<AppDbContext>(options =>
{
    var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
    
    // Use Managed Identity in production
    if (builder.Environment.IsProduction())
    {
        options.UseSqlServer(connectionString, sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(maxRetryCount: 3);
        });
    }
    else
    {
        options.UseSqlServer(connectionString);
    }
});
```

## References

- [EF Core — DbContext Configuration and Initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- [EF Core — Connection Resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency)