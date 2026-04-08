# DbContext Deep Dive

## Overview

DbContext is the primary class in Entity Framework Core responsible for coordinating with the database. It serves as a unit of work, providing facilities for querying, change tracking, and persisting data. Understanding its lifecycle, configuration, and internal mechanisms is essential for building high-performance applications.

---

## Core Responsibilities

```
DbContext Responsibilities:
|
+-- Querying
|   +-- LINQ translation
|   +-- Query execution
|   +-- Result materialization
|
+-- Change Tracking
|   +-- Entity state management
|   +-- Property modification detection
|   +-- Relationship fixup
|
+-- Persistence
|   +-- Transaction management
|   +-- SQL generation
|   +-- Command execution
|
+-- Caching
    +-- Identity map (entity instances)
    +-- Query plan cache
    +-- Metadata cache
```

---

## Lifecycle Management

### Lifecycle States

```
DbContext Lifecycle:

[Transient Scope]                    [Scoped Scope (Web)]
      |                                      |
      v                                      v
  Instantiate                          Per-request creation
      |                                      |
      v                                      v
  Configure services                 HTTP Request begins
      |                                      |
      v                                      v
  Use for operations                     v
      |                              [DbContext created]
      v                                      |
  Dispose (explicit/using)                 v
                                       [Operations execute]
                                              |
                                              v
                                       [Request ends]
                                              |
                                              v
                                       [DbContext disposed]
```

### Proper Disposal Patterns

```csharp
// Using statement (recommended for transient operations)
public async Task<Product?> GetProductAsync(int id)
{
    using var context = new ApplicationDbContext(_options);
    return await context.Products.FindAsync(id);
}

// Dependency Injection (recommended for web applications)
public class ProductController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public ProductController(ApplicationDbContext context)
    {
        _context = context; // Scoped lifetime
    }

    // Context disposed automatically at request end
}

// Factory pattern for long-running operations
public class BackgroundProductProcessor
{
    private readonly IDbContextFactory<ApplicationDbContext> _factory;

    public BackgroundProductProcessor(IDbContextFactory<ApplicationDbContext> factory)
    {
        _factory = factory;
    }

    public async Task ProcessBatchAsync()
    {
        await using var context = await _factory.CreateDbContextAsync();
        // Use context for batch operation
    }
}
```

---

## Configuration Architecture

### Configuration Sources

```
Configuration Resolution Order:

1. Data Annotations (highest priority)
   [Table("Products")]
   public class Product { }

2. Fluent API (overrides data annotations)
   modelBuilder.Entity<Product>()
       .ToTable("Products", schema: "inventory");

3. Conventions (lowest priority)
   Table name = DbSet property name
   Primary key = Id or EntityNameId
```

### Comprehensive Configuration Example

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
        : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Apply all configurations from assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

        // Global query filters
        modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
        modelBuilder.Entity<Order>().HasQueryFilter(o => o.OrderDate > new DateTime(2020, 1, 1));

        // Concurrency tokens
        modelBuilder.Entity<Product>()
            .Property(p => p.RowVersion)
            .IsRowVersion();

        // Computed columns
        modelBuilder.Entity<Order>()
            .Property(o => o.TotalAmount)
            .HasComputedColumnSql("[Subtotal] + [TaxAmount]");

        // Shadow properties
        modelBuilder.Entity<Customer>()
            .Property<DateTime>("LastModified")
            .HasDefaultValueSql("GETUTCDATE()");

        // Indexes
        modelBuilder.Entity<Product>()
            .HasIndex(p => new { p.CategoryId, p.Price })
            .HasDatabaseName("IX_Products_Category_Price")
            .HasFillFactor(80);

        // Sequences
        modelBuilder.HasSequence<int>("ProductSeq", schema: "shared")
            .StartsAt(1000)
            .IncrementsBy(1);
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder
                .UseSqlServer(
                    "Server=...",
                    sqlOptions =>
                    {
                        sqlOptions.EnableRetryOnFailure(
                            maxRetryCount: 3,
                            maxRetryDelay: TimeSpan.FromSeconds(30),
                            errorNumbersToAdd: null);
                        
                        sqlOptions.CommandTimeout(30);
                        sqlOptions.MinBatchSize(1);
                        sqlOptions.MaxBatchSize(100);
                    })
                .EnableSensitiveDataLogging(false)
                .EnableDetailedErrors(false)
                .LogTo(Console.WriteLine, LogLevel.Warning);
        }
    }
}

// Separate configuration classes (cleaner organization)
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products", schema: "inventory");
        
        builder.HasKey(p => p.ProductId);
        
        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);
        
        builder.Property(p => p.SKU)
            .IsRequired()
            .HasMaxLength(50);
        
        builder.HasIndex(p => p.SKU)
            .IsUnique();
        
        builder.Property(p => p.Price)
            .HasPrecision(18, 2);
        
        builder.Property(p => p.Description)
            .HasMaxLength(2000);
        
        // Owned entity configuration
        builder.OwnsOne(p => p.Dimensions, dimensions =>
        {
            dimensions.Property(d => d.Length).HasColumnName("DimensionLength");
            dimensions.Property(d => d.Width).HasColumnName("DimensionWidth");
            dimensions.Property(d => d.Height).HasColumnName("DimensionHeight");
        });
        
        // Many-to-many relationship
        builder.HasMany(p => p.Categories)
            .WithMany(c => c.Products)
            .UsingEntity<ProductCategory>(
                j => j.HasOne(pc => pc.Category).WithMany(),
                j => j.HasOne(pc => pc.Product).WithMany(),
                j =>
                {
                    j.HasKey(pc => new { pc.ProductId, pc.CategoryId });
                    j.ToTable("ProductCategories");
                });
    }
}
```

---

## Change Tracking Mechanisms

### Change Tracker Architecture

```
DbContext.ChangeTracker:
|
+-- Entries (EntityEntry collection)
|   |
|   +-- EntityEntry
|   |   +-- Entity (object reference)
|   |   +-- State (Added/Modified/Deleted/Unchanged)
|   |   +-- OriginalValues (snapshot at query time)
|   |   +-- CurrentValues (current property values)
|   |   +-- Properties (PropertyEntry collection)
|   |
|   +-- NavigationEntry
|       +-- CurrentValue
|       +-- IsLoaded
|
+-- DetectChanges()
|   +-- Scans tracked entities
|   +-- Compares current vs original values
|   +-- Updates EntityState
|
+-- CascadeDeleteTiming
+-- DeleteOrphansTiming
```

### Change Tracking Modes

```csharp
public class ChangeTrackingExamples
{
    private readonly ApplicationDbContext _context;

    public async Task DemonstrateTrackingModes()
    {
        // 1. Full Change Tracking (default)
        var trackedProduct = await _context.Products
            .FirstAsync(p => p.ProductId == 1);
        
        trackedProduct.Price = 99.99m; // Automatically detected
        await _context.SaveChangesAsync(); // UPDATE generated

        // 2. No-Tracking (read-only scenarios)
        var readOnlyProducts = await _context.Products
            .AsNoTracking()
            .Where(p => p.CategoryId == 5)
            .ToListAsync();
        // Changes not detected, memory efficient

        // 3. No-Tracking with Identity Resolution
        var productsWithIdentity = await _context.Products
            .AsNoTrackingWithIdentityResolution()
            .Include(p => p.Category)
            .ToListAsync();
        // Same entity references unified, no change tracking

        // 4. Explicit tracking control
        _context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
        
        // Re-enable for specific query
        var tracked = await _context.Products
            .AsTracking()
            .FirstAsync();
    }

    public async Task ManualChangeTracking()
    {
        // Detached entity scenario
        var productDto = new ProductDto { Id = 1, Price = 49.99m };
        
        // Attach and mark property as modified
        var product = new Product { ProductId = productDto.Id };
        _context.Products.Attach(product);
        _context.Entry(product).Property(p => p.Price).IsModified = true;
        
        // Only Price column in UPDATE statement
        await _context.SaveChangesAsync();
    }

    public void AccessTrackedEntities()
    {
        // Inspect change tracker
        foreach (var entry in _context.ChangeTracker.Entries())
        {
            Console.WriteLine($"Entity: {entry.Entity.GetType().Name}");
            Console.WriteLine($"State: {entry.State}");
            
            foreach (var property in entry.Properties)
            {
                if (property.IsModified)
                {
                    Console.WriteLine($"  {property.Metadata.Name}: " +
                        $"{property.OriginalValue} -> {property.CurrentValue}");
                }
            }
        }
    }
}
```

---

## Real-World Example: Multi-Tenant SaaS Application

### Scenario

Building a multi-tenant SaaS platform where each tenant's data is isolated via TenantId column.

### Implementation

```csharp
// Base entity with tenant support
public abstract class TenantEntity
{
    public Guid TenantId { get; set; }
    public Tenant Tenant { get; set; } = null!;
}

// Domain entities
public class Project : TenantEntity
{
    public int ProjectId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public ProjectStatus Status { get; set; }
    public ICollection<TaskItem> Tasks { get; set; } = new List<TaskItem>();
}

public class TaskItem : TenantEntity
{
    public int TaskId { get; set; }
    public int ProjectId { get; set; }
    public Project Project { get; set; } = null!;
    public string Title { get; set; } = string.Empty;
    public TaskPriority Priority { get; set; }
    public DateTime? DueDate { get; set; }
}

// Scoped tenant context
public interface ITenantContext
{
    Guid CurrentTenantId { get; }
}

public class TenantContext : ITenantContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public TenantContext(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public Guid CurrentTenantId
    {
        get
        {
            var tenantId = _httpContextAccessor.HttpContext?
                .User.FindFirst("tenant_id")?.Value;
            
            return tenantId != null ? Guid.Parse(tenantId) : Guid.Empty;
        }
    }
}

// Multi-tenant DbContext
public class SaaSDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public SaaSDbContext(
        DbContextOptions<SaaSDbContext> options,
        ITenantContext tenantContext) : base(options)
    {
        _tenantContext = tenantContext;
    }

    public DbSet<Tenant> Tenants { get; set; }
    public DbSet<Project> Projects { get; set; }
    public DbSet<TaskItem> Tasks { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Apply tenant filter to all tenant entities
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(TenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                var method = typeof(SaaSDbContext)
                    .GetMethod(nameof(SetTenantFilter), 
                        BindingFlags.NonPublic | BindingFlags.Static)!
                    .MakeGenericMethod(entityType.ClrType);
                
                method.Invoke(null, new object[] { modelBuilder });
            }
        }

        // Tenant entity configuration
        modelBuilder.Entity<Tenant>(entity =>
        {
            entity.HasKey(t => t.TenantId);
            entity.HasIndex(t => t.Subdomain).IsUnique();
        });

        // Project configuration
        modelBuilder.Entity<Project>(entity =>
        {
            entity.HasKey(p => p.ProjectId);
            entity.HasIndex(p => new { p.TenantId, p.Name });
            
            entity.HasOne(p => p.Tenant)
                .WithMany()
                .HasForeignKey(p => p.TenantId);
        });

        // Task configuration
        modelBuilder.Entity<TaskItem>(entity =>
        {
            entity.HasKey(t => t.TaskId);
            entity.HasIndex(t => new { t.TenantId, t.DueDate });
            
            entity.HasOne(t => t.Tenant)
                .WithMany()
                .HasForeignKey(t => t.TenantId);
                
            entity.HasOne(t => t.Project)
                .WithMany(p => p.Tasks)
                .HasForeignKey(t => new { t.TenantId, t.ProjectId })
                .HasPrincipalKey(p => new { p.TenantId, p.ProjectId });
        });
    }

    private static void SetTenantFilter<TEntity>(ModelBuilder builder) 
        where TEntity : TenantEntity
    {
        builder.Entity<TEntity>().HasQueryFilter(e => e.TenantId == TenantId);
    }

    // Property to access current tenant in filter
    private static Guid TenantId => _currentTenantId;
    private static Guid _currentTenantId;

    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        _currentTenantId = _tenantContext.CurrentTenantId;
        
        // Set tenant ID for added entities
        foreach (var entry in ChangeTracker.Entries<TenantEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.TenantId = _tenantContext.CurrentTenantId;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}

// Repository implementation
public class ProjectRepository
{
    private readonly SaaSDbContext _context;

    public ProjectRepository(SaaSDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Project>> GetActiveProjectsAsync()
    {
        // Tenant filter applied automatically
        return await _context.Projects
            .Where(p => p.Status == ProjectStatus.Active)
            .Include(p => p.Tasks.Where(t => !t.IsCompleted))
            .OrderBy(p => p.Name)
            .ToListAsync();
    }

    public async Task<Project?> GetProjectWithTasksAsync(int projectId)
    {
        // Tenant isolation enforced by global filter
        return await _context.Projects
            .Include(p => p.Tasks)
            .FirstOrDefaultAsync(p => p.ProjectId == projectId);
    }

    public async Task AddProjectAsync(Project project)
    {
        // TenantId set automatically in SaveChanges
        _context.Projects.Add(project);
        await _context.SaveChangesAsync();
    }
}

// Usage in controller
public class ProjectsController : ControllerBase
{
    private readonly ProjectRepository _repository;

    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProjectDto>>> GetProjects()
    {
        var projects = await _repository.GetActiveProjectsAsync();
        return Ok(projects.Select(p => new ProjectDto(p)));
    }

    [HttpPost]
    public async Task<ActionResult<ProjectDto>> CreateProject(CreateProjectRequest request)
    {
        var project = new Project 
        { 
            Name = request.Name,
            Description = request.Description 
        };
        
        await _repository.AddProjectAsync(project);
        
        return CreatedAtAction(
            nameof(GetProject), 
            new { id = project.ProjectId }, 
            new ProjectDto(project));
    }
}
```

### Generated SQL with Tenant Filter

```sql
-- Query: context.Projects.Where(p => p.Status == 'Active')
-- Automatically includes tenant filter

SELECT [p].[ProjectId], [p].[Description], [p].[Name], [p].[Status], [p].[TenantId]
FROM [Projects] AS [p]
WHERE ([p].[TenantId] = @__tenantId_0) AND ([p].[Status] = 1)
ORDER BY [p].[Name]
```

---

## Performance Optimization

### Context Pooling

```csharp
// Program.cs - Enable context pooling
builder.Services.AddDbContextPool<ApplicationDbContext>(options =>
{
    options.UseSqlServer(connectionString);
}, poolSize: 1024);

// Pooled context resets state between uses
// Avoids context creation overhead
```

### Split Queries

```csharp
// Avoid cartesian explosion with Include
var products = await context.Products
    .AsSplitQuery() // Separate query per Include
    .Include(p => p.Category)
    .Include(p => p.Reviews)
    .Include(p => p.Supplier)
    .ToListAsync();

// Generates multiple SQL statements instead of large JOIN
```

### Compiled Queries

```csharp
// Pre-compile query for reuse
private static readonly Func<ApplicationDbContext, int, Task<Product?>> GetProductById =
    EF.CompileAsyncQuery(
        (ApplicationDbContext context, int productId) =>
            context.Products.FirstOrDefault(p => p.ProductId == productId));

// Usage
public async Task<Product?> GetProduct(int id)
{
    return await GetProductById(_context, id);
}
```

---

## Summary

DbContext is the central abstraction in EF Core, managing database connections, query translation, change tracking, and persistence. Proper configuration of lifecycle, change tracking mode, and query behavior is essential for building scalable applications. Understanding its internal mechanisms enables optimization and troubleshooting of complex scenarios.
