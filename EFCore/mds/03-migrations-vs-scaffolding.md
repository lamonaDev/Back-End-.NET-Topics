# Migrations vs Scaffolding in EF Core

## Overview

EF Core provides two primary approaches for aligning code models with database schemas: **Migrations** (code-first) and **Scaffolding** (database-first). Each approach serves different workflow requirements and team preferences.

---

## Migrations (Code-First)

Migrations enable developers to define the domain model in C# code and use EF Core to generate and apply database schema changes incrementally.

### Workflow

```
Step 1: Define/Modify Model Classes
         |
         v
Step 2: Create Migration (CLI/API)
         |
         +---> Generate Migration File (C# code)
         +---> Generate Designer File (snapshot)
         +---> Update Model Snapshot
         |
         v
Step 3: Apply Migration to Database
         |
         +---> Generate SQL scripts
         +---> Execute against database
         +---> Update __EFMigrationsHistory table
         |
         v
Step 4: Database Schema Updated
```

### Migration Components

```
Migrations/
├── 20240101120000_InitialCreate.cs          # Up/Down methods
├── 20240101120000_InitialCreate.Designer.cs # Model snapshot
├── 20240201150000_AddProductReviews.cs      # Subsequent migration
├── 20240201150000_AddProductReviews.Designer.cs
└── ApplicationDbContextModelSnapshot.cs     # Current model state
```

---

## Scaffolding (Database-First)

Scaffolding reverse-engineers an existing database schema into C# entity classes and a DbContext, ideal for working with legacy databases or DBA-managed schemas.

### Workflow

```
Step 1: Design Database Schema
         | (using SQL scripts, SSMS, DBeaver, etc.)
         v
Step 2: Run Scaffold Command (CLI)
         |
         +---> Introspect database schema
         +---> Map tables to entity classes
         +---> Generate navigation properties
         +---> Create DbContext configuration
         |
         v
Step 3: Generated Code
         |
         +---> Entities/ (POCO classes)
         +---> ApplicationDbContext.cs
         |
         v
Step 4: Integrate into Application
```

---

## Detailed Comparison

| Aspect | Migrations | Scaffolding |
|--------|------------|-------------|
| **Starting Point** | C# code | Database schema |
| **Control** | Developer controls model | DBA controls schema |
| **Version Control** | Full history tracked | Manual management |
| **Team Workflow** | Developers own schema | DBAs own schema |
| **Greenfield/Brownfield** | Greenfield preferred | Brownfield/legacy |
| **Customization** | Fluent API attributes | Partial classes, extensions |
| **Schema Evolution** | Incremental updates | Re-scaffold or manual sync |
| **Database Features** | Limited by provider | Full database feature access |

---

## Real-World Example: E-Commerce System

### Scenario A: New System (Migrations Approach)

Building an e-commerce platform from scratch with an agile development team.

#### Initial Model Definition

```csharp
// Domain model in C#
public class Product
{
    public int ProductId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    
    // Navigation property
    public ICollection<ProductCategory> ProductCategories { get; set; } = 
        new List<ProductCategory>();
}

public class Category
{
    public int CategoryId { get; set; }
    public string Name { get; set; } = string.Empty;
    public int? ParentCategoryId { get; set; }
    
    // Self-referencing relationship
    public Category? ParentCategory { get; set; }
    public ICollection<Category> SubCategories { get; set; } = 
        new List<Category>();
}

public class ProductCategory
{
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
}

// DbContext configuration
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
        : base(options) { }

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<ProductCategory>(entity =>
        {
            entity.HasKey(e => new { e.ProductId, e.CategoryId });
            
            entity.HasOne(e => e.Product)
                .WithMany(p => p.ProductCategories)
                .HasForeignKey(e => e.ProductId);
                
            entity.HasOne(e => e.Category)
                .WithMany()
                .HasForeignKey(e => e.CategoryId);
        });

        modelBuilder.Entity<Category>(entity =>
        {
            entity.HasOne(e => e.ParentCategory)
                .WithMany(e => e.SubCategories)
                .HasForeignKey(e => e.ParentCategoryId)
                .OnDelete(DeleteBehavior.Restrict);
        });
    }
}
```

#### Creating Initial Migration

```powershell
# Create initial migration
Add-Migration InitialCreate -OutputDir Migrations

# Review generated migration file
# Apply to development database
Update-Database

# Generate SQL script for production deployment
Script-Migration -Idempotent
```

#### Generated Migration File

```csharp
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Categories",
            columns: table => new
            {
                CategoryId = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(type: "nvarchar(max)", nullable: false),
                ParentCategoryId = table.Column<int>(type: "int", nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Categories", x => x.CategoryId);
                table.ForeignKey(
                    name: "FK_Categories_Categories_ParentCategoryId",
                    column: x => x.ParentCategoryId,
                    principalTable: "Categories",
                    principalColumn: "CategoryId",
                    onDelete: ReferentialAction.Restrict);
            });

        migrationBuilder.CreateTable(
            name: "Products",
            columns: table => new
            {
                ProductId = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                Name = table.Column<string>(type: "nvarchar(max)", nullable: false),
                Description = table.Column<string>(type: "nvarchar(max)", nullable: false),
                Price = table.Column<decimal>(type: "decimal(18,2)", nullable: false),
                StockQuantity = table.Column<int>(type: "int", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Products", x => x.ProductId);
            });

        migrationBuilder.CreateTable(
            name: "ProductCategories",
            columns: table => new
            {
                ProductId = table.Column<int>(type: "int", nullable: false),
                CategoryId = table.Column<int>(type: "int", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_ProductCategories", x => new { x.ProductId, x.CategoryId });
                table.ForeignKey(
                    name: "FK_ProductCategories_Categories_CategoryId",
                    column: x => x.CategoryId,
                    principalTable: "Categories",
                    principalColumn: "CategoryId",
                    onDelete: ReferentialAction.Cascade);
                table.ForeignKey(
                    name: "FK_ProductCategories_Products_ProductId",
                    column: x => x.ProductId,
                    principalTable: "Products",
                    principalColumn: "ProductId",
                    onDelete: ReferentialAction.Cascade);
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "ProductCategories");
        migrationBuilder.DropTable(name: "Categories");
        migrationBuilder.DropTable(name: "Products");
    }
}
```

#### Subsequent Migration (Adding Reviews)

```csharp
// Add new entity
public class ProductReview
{
    public int ReviewId { get; set; }
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
    public string ReviewerName { get; set; } = string.Empty;
    public int Rating { get; set; } // 1-5
    public string Comment { get; set; } = string.Empty;
    public DateTime ReviewDate { get; set; }
    public bool IsApproved { get; set; }
}

// Update DbContext
public DbSet<ProductReview> ProductReviews { get; set; }

// Configure in OnModelCreating
modelBuilder.Entity<ProductReview>(entity =>
{
    entity.Property(e => e.Rating)
        .HasDefaultValue(1)
        .HasAnnotation("MinValue", 1)
        .HasAnnotation("MaxValue", 5);
        
    entity.Property(e => e.IsApproved)
        .HasDefaultValue(false);
        
    entity.HasIndex(e => e.ProductId)
        .HasDatabaseName("IX_ProductReviews_ProductId");
        
    entity.HasIndex(e => new { e.Rating, e.IsApproved })
        .HasDatabaseName("IX_ProductReviews_Rating_Approved");
});
```

```powershell
# Create migration for new feature
Add-Migration AddProductReviews

# Apply to database
Update-Database
```

---

### Scenario B: Legacy Database Integration (Scaffolding Approach)

Integrating with an existing enterprise database managed by a DBA team.

#### Database Schema

```sql
-- Existing tables managed by DBA
CREATE TABLE dbo.Customer (
    CustomerID INT IDENTITY(1,1) PRIMARY KEY,
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    Email NVARCHAR(100) NOT NULL UNIQUE,
    Phone NVARCHAR(20),
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

CREATE TABLE dbo.CustomerOrder (
    OrderID INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID INT NOT NULL FOREIGN KEY REFERENCES dbo.Customer(CustomerID),
    OrderDate DATETIME2 DEFAULT GETDATE(),
    TotalAmount DECIMAL(18,2) NOT NULL,
    Status NVARCHAR(20) DEFAULT 'Pending'
);

CREATE TABLE dbo.OrderLineItem (
    LineItemID INT IDENTITY(1,1) PRIMARY KEY,
    OrderID INT NOT NULL FOREIGN KEY REFERENCES dbo.CustomerOrder(OrderID),
    ProductCode NVARCHAR(50) NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL
);
```

#### Scaffold Command

```powershell
# Scaffold from existing database
Scaffold-DbContext "Server=dbserver;Database=LegacyDB;User Id=appuser;Password=***;" `
    Microsoft.EntityFrameworkCore.SqlServer `
    -OutputDir Entities `
    -ContextDir . `
    -Context ApplicationDbContext `
    -Tables Customer,CustomerOrder,OrderLineItem `
    -DataAnnotations `
    -Force
```

#### Generated Entities

```csharp
// Customer.cs
[Table("Customer", Schema = "dbo")]
public partial class Customer
{
    [Key]
    [Column("CustomerID")]
    public int CustomerId { get; set; }

    [Required]
    [Column("FirstName")]
    [StringLength(50)]
    public string FirstName { get; set; } = null!;

    [Required]
    [Column("LastName")]
    [StringLength(50)]
    public string LastName { get; set; } = null!;

    [Required]
    [Column("Email")]
    [StringLength(100)]
    public string Email { get; set; } = null!;

    [Column("Phone")]
    [StringLength(20)]
    public string? Phone { get; set; }

    [Column("CreatedDate")]
    public DateTime CreatedDate { get; set; }

    [InverseProperty("Customer")]
    public virtual ICollection<CustomerOrder> CustomerOrders { get; set; } = new List<CustomerOrder>();
}

// CustomerOrder.cs
[Table("CustomerOrder", Schema = "dbo")]
public partial class CustomerOrder
{
    [Key]
    [Column("OrderID")]
    public int OrderId { get; set; }

    [Column("CustomerID")]
    public int CustomerId { get; set; }

    [Column("OrderDate")]
    public DateTime OrderDate { get; set; }

    [Column("TotalAmount", TypeName = "decimal(18, 2)")]
    public decimal TotalAmount { get; set; }

    [Column("Status")]
    [StringLength(20)]
    public string Status { get; set; } = null!;

    [ForeignKey("CustomerId")]
    [InverseProperty("CustomerOrders")]
    public virtual Customer Customer { get; set; } = null!;

    [InverseProperty("Order")]
    public virtual ICollection<OrderLineItem> OrderLineItems { get; set; } = new List<OrderLineItem>();
}
```

#### Extending Scaffoled Code

```csharp
// Partial class for business logic (preserved on re-scaffold)
public partial class Customer
{
    // Computed property (not mapped)
    [NotMapped]
    public string FullName => $"{FirstName} {LastName}";

    // Business method
    public bool CanPlaceOrder() => 
        !string.IsNullOrEmpty(Email) && 
        CustomerOrders.Count(o => o.Status == "Pending") < 5;
}

// Custom configuration (preserved on re-scaffold)
public partial class ApplicationDbContext
{
    partial void OnModelCreatingPartial(ModelBuilder modelBuilder)
    {
        // Add query filters
        modelBuilder.Entity<CustomerOrder>()
            .HasQueryFilter(o => o.Status != "Deleted");
    }
}
```

---

## Migration Strategies

### Team Development Workflow

```
Developer A                    Shared Branch                   Developer B
    |                                |                              |
    |-- Add Feature: Orders          |                              |
    |-- Add-Migration AddOrders     |                              |
    |-- Commit & Push --------------->|                              |
    |                                |-- Pull                       |
    |                                |-- Merge conflict?             |
    |                                |   (Multiple migrations OK)   |
    |                                |-- Apply migrations           |
    |                                |                              |
    |                                |<------------------ Add Feature: Reviews
    |                                |                              |
    |                                |-- Add-Migration AddReviews  |
    |                                |<------------------ Commit    |
    |-- Pull ---------------------->|                              |
    |-- Apply migrations            |                              |
```

### Handling Merge Conflicts

```powershell
# Scenario: Two developers created migrations simultaneously

# Developer A's migration: 20240101120000_AddOrders
# Developer B's migration: 20240101130000_AddReviews

# Both are valid - EF Core applies in timestamp order
# Database will have both migrations in __EFMigrationsHistory

# If conflicts arise in ModelSnapshot, regenerate:
Remove-Migration  # Remove problematic migration
Add-Migration     # Recreate with merged model
```

---

## Best Practices

### Migrations

1. **Review Generated SQL Before Production**
   ```powershell
   Script-Migration -From PreviousMigration -To LatestMigration
   ```

2. **Keep Migrations Small and Focused**
   - One logical change per migration
   - Easier to rollback if issues arise

3. **Use Data Seeding for Reference Data**
   ```csharp
   migrationBuilder.InsertData(
       table: "Categories",
       columns: new[] { "CategoryId", "Name" },
       values: new object[,]
       {
           { 1, "Electronics" },
           { 2, "Clothing" },
           { 3, "Books" }
       });
   ```

4. **Never Modify Applied Migrations**
   - Create new migrations for corrections
   - In development: can remove/recreate if not applied

### Scaffolding

1. **Use Partial Classes for Extensions**
   - Preserved when re-scaffolding
   - Separates generated and custom code

2. **Configure Connection String Externally**
   ```csharp
   // Don't hardcode in scaffolded DbContext
   protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
   {
       if (!optionsBuilder.IsConfigured)
       {
           // Only for design-time scaffolding
           optionsBuilder.UseSqlServer("...");
       }
   }
   ```

3. **Version Control Scaffolding Configuration**
   - Document exact scaffold command used
   - Include in README or script file

---

## Decision Framework

```
Is this a new project?
|
+-- YES
|   |
|   +-- Do you have DBA-controlled schema changes?
|       |
|       +-- YES -> Hybrid: Scaffold initial, then migrations
|       +-- NO  -> Migrations (Code-First)
|
+-- NO (Existing database)
    |
    +-- Will schema be managed by DBAs ongoing?
        |
        +-- YES -> Scaffolding (re-scaffold on changes)
        +-- NO  -> Scaffolding initial, then migrations
```

---

## Summary

Migrations provide version-controlled, incremental schema evolution ideal for agile development teams. Scaffolding enables rapid integration with existing databases managed by DBAs. The choice depends on team structure, database ownership, and project lifecycle requirements.
