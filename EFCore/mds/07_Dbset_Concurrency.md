
## DbSet<TEntity>

`DbSet<TEntity>` represents a collection of entities of a specific type that can be queried from the database. It serves as the gateway for database operations (query, insert, update, delete) on entity collections.

### Generic Example

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
    public DbSet<Author> Authors { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer("Server=.;Database=BlogDb;Trusted_Connection=True;");
}

public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }
    public List<Post> Posts { get; set; }
}
```

### Common Operations

```csharp
using var context = new ApplicationDbContext();

// Query
var blogs = await context.Blogs.Where(b => b.Url.Contains("microsoft")).ToListAsync();

// Add
context.Blogs.Add(new Blog { Url = "https://newblog.com" });
await context.SaveChangesAsync();

// Update
var blog = await context.Blogs.FindAsync(1);
blog.Url = "https://updated.com";
await context.SaveChangesAsync();

// Remove
context.Blogs.Remove(blog);
await context.SaveChangesAsync();
```

---

## Concurrency Tokens / RowVersion

Concurrency tokens prevent simultaneous updates from overwriting each other. When a conflict is detected, EF Core throws a `DbUpdateConcurrencyException`.

### SQL Server RowVersion

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

### Fluent API Configuration

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>()
        .Property(p => p.RowVersion)
        .IsRowVersion();
}
```

### Handling Concurrency Conflicts

```csharp
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var clientValues = entry.CurrentValues;
    var databaseValues = await entry.GetDatabaseValuesAsync();
    
    // Resolve conflict: use database values, client values, or merge
    entry.OriginalValues.SetValues(databaseValues);
    await context.SaveChangesAsync();
}
```

---

## InverseProperty Attribute

`InverseProperty` establishes navigation property relationships when multiple relationships exist between the same entity types. It clarifies which navigation property on the other side corresponds to each relationship.

### Without InverseProperty (Ambiguous)

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    // Which Post.Author? Which Post.Reviewer?
    public List<Post> WrittenPosts { get; set; }
    public List<Post> ReviewedPosts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    
    public User Author { get; set; }
    public User Reviewer { get; set; }
}
```

### With InverseProperty (Resolved)

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    [InverseProperty("Author")]
    public List<Post> WrittenPosts { get; set; }
    
    [InverseProperty("Reviewer")]
    public List<Post> ReviewedPosts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    
    public int AuthorId { get; set; }
    public User Author { get; set; }
    
    public int? ReviewerId { get; set; }
    public User Reviewer { get; set; }
}
```
