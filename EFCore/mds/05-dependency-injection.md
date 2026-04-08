# Dependency Injection and DbContext

## Overview

Entity Framework Core integrates seamlessly with .NET Dependency Injection (DI), enabling proper lifecycle management, testability, and configuration flexibility. Understanding the relationship between DI containers and DbContext is critical for building maintainable applications.

---

## DI Registration Patterns

### Registration Scopes

```
Service Lifetime Options:

Transient:
  New instance per injection
  + Simple, thread-safe
  - Higher memory usage
  - Connection pooling less effective

Scoped (Recommended for Web):
  One instance per HTTP request
  + Natural unit of work boundary
  + Efficient connection reuse
  - Not for background services

Singleton:
  One instance for application lifetime
  + Minimal memory overhead
  - Not supported for DbContext
  - Requires thread safety
```

### Standard Registration

```csharp
// Program.cs - ASP.NET Core
var builder = WebApplication.CreateBuilder(args);

// Standard scoped registration
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
});

// With additional configuration
builder.Services.AddDbContext<ApplicationDbContext>((serviceProvider, options) =>
{
    var connectionString = serviceProvider
        .GetRequiredService<IConfiguration>()
        .GetConnectionString("DefaultConnection");
    
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(3);
        sqlOptions.CommandTimeout(30);
    });
    
    // Use service provider for logging
    var loggerFactory = serviceProvider.GetRequiredService<ILoggerFactory>();
    options.UseLoggerFactory(loggerFactory);
});
```

---

## Real-World Example: Enterprise Application

### Scenario

Multi-layered application with separate projects for API, Services, Data, and Domain.

### Project Structure

```
Solution/
├── src/
│   ├── Domain/                    (Entities, interfaces)
│   │   └── Entities/
│   ├── Infrastructure.Data/       (EF Core, repositories)
│   │   ├── DbContext/
│   │   ├── Repositories/
│   │   └── Configurations/
│   ├── Application/               (Services, DTOs)
│   │   ├── Services/
│   │   └── Interfaces/
│   └── WebAPI/                    (Controllers, DI setup)
│       └── Program.cs
└── tests/
```

### Domain Layer (No EF Dependencies)

```csharp
// Domain/Entities/Product.cs
public class Product
{
    public int ProductId { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public decimal Price { get; private set; }
    
    // Domain behavior
    public void UpdatePrice(decimal newPrice)
    {
        if (newPrice <= 0)
            throw new DomainException("Price must be positive");
        
        Price = newPrice;
    }
}

// Domain/Interfaces/IRepository.cs
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
    Task SaveChangesAsync();
}

public interface IProductRepository : IRepository<Product>
{
    Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId);
    Task<Product?> GetBySkuAsync(string sku);
}
```

### Data Layer (EF Core Implementation)

```csharp
// Infrastructure.Data/DbContext/ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
        : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }
}

// Infrastructure.Data/Configurations/ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.ProductId);
        
        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);
        
        builder.Property(p => p.Price)
            .HasPrecision(18, 2);
        
        builder.HasIndex(p => p.Name);
    }
}

// Infrastructure.Data/Repositories/Repository.cs
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext Context;
    protected readonly DbSet<T> DbSet;

    public Repository(ApplicationDbContext context)
    {
        Context = context;
        DbSet = context.Set<T>();
    }

    public virtual async Task<T?> GetByIdAsync(int id)
    {
        return await DbSet.FindAsync(id);
    }

    public virtual async Task<IEnumerable<T>> GetAllAsync()
    {
        return await DbSet.ToListAsync();
    }

    public virtual async Task AddAsync(T entity)
    {
        await DbSet.AddAsync(entity);
    }

    public virtual void Update(T entity)
    {
        DbSet.Update(entity);
    }

    public virtual void Delete(T entity)
    {
        DbSet.Remove(entity);
    }

    public async Task SaveChangesAsync()
    {
        await Context.SaveChangesAsync();
    }
}

// Infrastructure.Data/Repositories/ProductRepository.cs
public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(ApplicationDbContext context) : base(context) { }

    public async Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId)
    {
        return await Context.Products
            .Where(p => p.CategoryId == categoryId)
            .ToListAsync();
    }

    public async Task<Product?> GetBySkuAsync(string sku)
    {
        return await Context.Products
            .FirstOrDefaultAsync(p => p.SKU == sku);
    }
}

// Infrastructure.Data/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureData(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // DbContext registration
        services.AddDbContext<ApplicationDbContext>(options =>
        {
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                sqlOptions =>
                {
                    sqlOptions.MigrationsAssembly(
                        typeof(ApplicationDbContext).Assembly.GetName().Name);
                    sqlOptions.EnableRetryOnFailure(3);
                });
        });

        // Repository registrations
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<ICategoryRepository, CategoryRepository>();
        services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

        return services;
    }
}
```

### Application Layer (Services)

```csharp
// Application/Services/ProductService.cs
public class ProductService : IProductService
{
    private readonly IProductRepository _productRepository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IEventBus _eventBus;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IProductRepository productRepository,
        IUnitOfWork unitOfWork,
        IEventBus eventBus,
        ILogger<ProductService> logger)
    {
        _productRepository = productRepository;
        _unitOfWork = unitOfWork;
        _eventBus = eventBus;
        _logger = logger;
    }

    public async Task<ProductDto> GetProductAsync(int id)
    {
        var product = await _productRepository.GetByIdAsync(id);
        
        if (product == null)
            throw new NotFoundException($"Product {id} not found");
        
        return new ProductDto(product);
    }

    public async Task<ProductDto> CreateProductAsync(CreateProductRequest request)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price,
            SKU = request.SKU
        };

        await _productRepository.AddAsync(product);
        await _unitOfWork.SaveChangesAsync();

        _logger.LogInformation("Product {ProductId} created", product.ProductId);
        
        await _eventBus.PublishAsync(new ProductCreatedEvent(product.ProductId));

        return new ProductDto(product);
    }

    public async Task UpdateProductPriceAsync(int id, decimal newPrice)
    {
        var product = await _productRepository.GetByIdAsync(id);
        
        if (product == null)
            throw new NotFoundException($"Product {id} not found");

        product.UpdatePrice(newPrice);
        
        _productRepository.Update(product);
        await _unitOfWork.SaveChangesAsync();
    }
}

// Application/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<ICategoryService, CategoryService>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        
        // Validation, AutoMapper, etc.
        services.AddAutoMapper(typeof(DependencyInjection).Assembly);
        
        return services;
    }
}
```

### Unit of Work Pattern

```csharp
// Application/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    ICategoryRepository Categories { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task<IDbContextTransaction> BeginTransactionAsync();
}

// Infrastructure.Data/UnitOfWork.cs
public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction? _currentTransaction;

    public IProductRepository Products { get; }
    public ICategoryRepository Categories { get; }

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
        Products = new ProductRepository(context);
        Categories = new CategoryRepository(context);
    }

    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task<IDbContextTransaction> BeginTransactionAsync()
    {
        _currentTransaction = await _context.Database
            .BeginTransactionAsync();
        return _currentTransaction;
    }

    public async Task CommitTransactionAsync()
    {
        if (_currentTransaction != null)
        {
            await _currentTransaction.CommitAsync();
            await _currentTransaction.DisposeAsync();
            _currentTransaction = null;
        }
    }

    public void Dispose()
    {
        _currentTransaction?.Dispose();
        _context.Dispose();
    }
}
```

### Web API Layer (DI Configuration)

```csharp
// WebAPI/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Configuration
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables();

// Layer registrations (order matters)
builder.Services.AddInfrastructureData(builder.Configuration);
builder.Services.AddApplication();

// API-specific services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Validation
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductRequestValidator>();

// Build application
var app = builder.Build();

// Middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

// Ensure database migration on startup
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await context.Database.MigrateAsync();
}

app.Run();
```

---

## Advanced DI Scenarios

### Multiple DbContexts

```csharp
// Registration
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));

builder.Services.AddDbContext<AuditDbContext>(options =>
    options.UseSqlServer(connectionString));

builder.Services.AddDbContext<ReportingDbContext>(options =>
    options.UseSqlServer(reportingConnectionString));

// Usage
public class MultiContextService
{
    private readonly ApplicationDbContext _appContext;
    private readonly AuditDbContext _auditContext;
    private readonly ReportingDbContext _reportingContext;

    public MultiContextService(
        ApplicationDbContext appContext,
        AuditDbContext auditContext,
        ReportingDbContext reportingContext)
    {
        _appContext = appContext;
        _auditContext = auditContext;
        _reportingContext = reportingContext;
    }
}
```

### Factory Pattern for Background Services

```csharp
// For scoped DbContext in singleton services
builder.Services.AddDbContextFactory<ApplicationDbContext>(options =>
{
    options.UseSqlServer(connectionString);
}, lifetime: ServiceLifetime.Scoped);

// Background service
public class ProductSyncService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<ProductSyncService> _logger;

    public ProductSyncService(
        IServiceProvider serviceProvider,
        ILogger<ProductSyncService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var context = scope.ServiceProvider
                    .GetRequiredService<ApplicationDbContext>();
                
                // Or use factory
                // var factory = scope.ServiceProvider
                //     .GetRequiredService<IDbContextFactory<ApplicationDbContext>>();
                // await using var context = await factory.CreateDbContextAsync();
                
                await SyncProductsAsync(context);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Sync failed");
            }

            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }

    private async Task SyncProductsAsync(ApplicationDbContext context)
    {
        var productsToSync = await context.Products
            .Where(p => p.LastSync < DateTime.UtcNow.AddHours(-1))
            .ToListAsync();
        
        // Sync logic...
    }
}
```

---

## Testing with DI

### Integration Test Setup

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real database
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
            
            if (descriptor != null)
                services.Remove(descriptor);

            // Add in-memory database for testing
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });

            // Seed test data
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            context.Database.EnsureCreated();
            SeedTestData(context);
        });
    }

    private void SeedTestData(ApplicationDbContext context)
    {
        context.Products.AddRange(
            new Product { ProductId = 1, Name = "Test Product", Price = 99.99m },
            new Product { ProductId = 2, Name = "Another Product", Price = 49.99m }
        );
        context.SaveChanges();
    }
}

// Test class
public class ProductsControllerTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public ProductsControllerTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetProduct_ReturnsProduct()
    {
        var response = await _client.GetAsync("/api/products/1");
        response.EnsureSuccessStatusCode();
        
        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        Assert.NotNull(product);
        Assert.Equal("Test Product", product.Name);
    }
}
```

---

## Summary

Dependency Injection integration with EF Core enables clean architecture, testability, and proper lifecycle management. Key practices include registering DbContext with scoped lifetime, abstracting data access through repository interfaces, implementing Unit of Work for transaction management, and using factories for background services.
