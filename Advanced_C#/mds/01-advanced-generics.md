# Advanced Generics in C#

## Table of Contents
1. [Generic Constraints Deep Dive](#generic-constraints-deep-dive)
2. [Covariance and Contravariance](#covariance-and-contravariance)
3. [Multiple Generic Type Parameters](#multiple-generic-type-parameters)
4. [Generic Methods](#generic-methods)
5. [Generic Inheritance](#generic-inheritance)

---

## Generic Constraints Deep Dive

### What are Generic Constraints?

Generic constraints tell the compiler what capabilities a type parameter `T` must have. Without constraints, `T` could be any type, and the compiler doesn't know what operations are valid on it.

### Types of Constraints

| Constraint | Syntax | Purpose |
|------------|--------|---------|
| **class** | `where T : class` | T must be a reference type (not value type) |
| **struct** | `where T : struct` | T must be a value type (non-nullable) |
| **new()** | `where T : new()` | T must have a parameterless constructor |
| **Base Class** | `where T : BaseClass` | T must inherit from BaseClass |
| **Interface** | `where T : IInterface` | T must implement IInterface |
| **Multiple** | `where T : class, IInterface, new()` | T must satisfy ALL constraints |

### Real-World Example: Generic Repository

```csharp
// Without constraints - we can't access any properties of T
public class BadRepository<T>
{
    public T GetById(int id)
    {
        // ❌ ERROR: 'T' does not contain a definition for 'Id'
        return _data.FirstOrDefault(x => x.Id == id);
    }
}

// With constraints - we can access BaseEntity properties
public class BaseEntity<TKey>
{
    public TKey Id { get; set; }
}

public class GenericRepository<T, TKey> where T : BaseEntity<TKey>
{
    private readonly List<T> _data = new();
    
    public T? GetById(TKey id)
    {
        // ✓ Works! We know T has an Id property
        return _data.FirstOrDefault(x => x.Id?.Equals(id) == true);
    }
    
    public void Add(T entity)
    {
        if (entity.Id == null)
            throw new ArgumentException("Id cannot be null");
        _data.Add(entity);
    }
}

// Usage
public class User : BaseEntity<int>
{
    public string Name { get; set; }
}

public class Order : BaseEntity<Guid>
{
    public decimal Total { get; set; }
}

var userRepo = new GenericRepository<User, int>();
var orderRepo = new GenericRepository<Order, Guid>();
```

### Memory Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│                    GENERIC CONSTRAINTS AT COMPILE TIME             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Compiler sees: GenericRepository<User, int>                   │
│                                                                  │
│   Constraint Check: User : BaseEntity<int>? ✓                  │
│                                                                  │
│   Code Generation:                                               │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  class GenericRepository__User_int                      │    │
│   │  {                                                      │    │
│   │    private List<User> _data;                           │    │
│   │                                                         │    │
│   │    public User? GetById(int id)                        │    │
│   │    {                                                    │    │
│   │      return _data.FirstOrDefault(x => x.Id == id);     │    │
│   │    }                                                    │    │
│   │  }                                                      │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Note: Generic types are JIT-compiled to specific types         │
│   at runtime for value types, or use shared code for             │
│   reference types.                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Covariance and Contravariance

### What is Variance?

Variance describes how generic types with different type arguments relate to each other based on the inheritance of those type arguments.

| Term | Keyword | Meaning | Example |
|------|---------|---------|---------|
| **Covariance** | `out T` | You can use a MORE derived type | `IEnumerable<Cat>` → `IEnumerable<Animal>` |
| **Contravariance** | `in T` | You can use a LESS derived type | `IComparer<Animal>` → `IComparer<Cat>` |
| **Invariance** | `T` | No conversion allowed | `IList<Cat>` cannot be `IList<Animal>` |

### Covariance (`out T`) - Producers

```csharp
// IEnumerable<out T> is covariant
public interface IReadable<out T>
{
    T GetById(int id);           // T is ONLY returned (output)
    IEnumerable<T> GetAll();     // T is ONLY returned (output)
    // ❌ void Add(T item);      // Cannot have T as parameter!
}

// This is SAFE because we're only READING from the collection
IReadable<Dog> dogRepo = new DogRepository();
IReadable<Animal> animalRepo = dogRepo;  // ✓ Covariance allows this

// We can only GET animals from it - safe!
Animal a = animalRepo.GetById(1);  // Actually returns a Dog
```

### Contravariance (`in T`) - Consumers

```csharp
// IComparer<in T> is contravariant
public interface IWriter<in T>
{
    void Write(T item);          // T is ONLY input
    void Update(T item);         // T is ONLY input
    // ❌ T Read();              // Cannot return T!
}

// This is SAFE because we're only WRITING to the collection
IWriter<Animal> animalWriter = new AnimalWriter();
IWriter<Dog> dogWriter = animalWriter;  // ✓ Contravariance allows this

// We can only WRITE dogs to it - safe because Animal writer can handle Dogs!
dogWriter.Write(new Dog());  // Animal writer can handle Dog
```

### Memory/IL Perspective

```
┌─────────────────────────────────────────────────────────────────┐
│                    VARIANCE AT THE IL LEVEL                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Covariant interface:                                           │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  .class interface public abstract auto ansi              │    │
│   │  {                                                       │    │
│   │    .custom instance void [mscorlib]System.Runtime.CompilerServices│
│   │    .CompilerGeneratedAttribute::.ctor() = ( 01 00 00 00 ) │    │
│   │                                                          │    │
│   │    // +1 for covariant                                   │    │
│   │    .param [0]                                            │    │
│   │    .custom instance void [mscorlib]System.Runtime.CompilerServices│
   │   │    .CovariantAttribute::.ctor() = ( 01 00 00 00 )      │    │
│   │  }                                                       │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   The compiler enforces variance rules:                          │
│   - out T: T can only appear in return positions                  │
│   - in T: T can only appear in parameter positions                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Real-World Use Case: Report Service

```csharp
public interface IReadable<out T> where T : BaseEntity
{
    T? GetById(int id);
    IEnumerable<T> GetAll();
}

public class ReportService
{
    // Works with ANY repository of entities!
    public void GenerateReport(IReadable<BaseEntity> repository)
    {
        var entities = repository.GetAll();
        foreach (var entity in entities)
        {
            Console.WriteLine($"Entity {entity.Id}");
        }
    }
}

// Can pass any specific repository to the general report service
var userRepo = new UserRepository();  // IReadable<User>
var reportService = new ReportService();
reportService.GenerateReport(userRepo);  // ✓ Covariance makes this work!
```

---

## Multiple Generic Type Parameters

### Why Multiple Type Parameters?

When you need more than one type to vary independently, use multiple type parameters.

```csharp
// Single type parameter - inflexible for ID types
public class Repository<T> where T : BaseEntity  // Forces int IDs
{
    public T GetById(int id) { }  // ID type is fixed!
}

// Multiple type parameters - flexible
public class Repository<TEntity, TKey> 
    where TEntity : BaseEntity<TKey>
{
    public TEntity GetById(TKey id) { }  // ID type is flexible!
}

// Usage
var userRepo = new Repository<User, int>();      // Int IDs
var orderRepo = new Repository<Order, Guid>();   // Guid IDs
var catRepo = new Repository<Category, string>(); // String IDs
```

### Constraint Combinations

```csharp
public class AuditableRepository<T, TKey>
    where T : class, BaseEntity<TKey>, IAuditable, new()
    //    ↑     ↑            ↑            ↑           ↑
    //    |     |            |            |           └─ Has parameterless ctor
    //    |     |            |            └─ Implements IAuditable
    //    |     |            └─ Inherits BaseEntity<TKey>
    //    |     └─ Is reference type (not struct)
    //    └─ Can compare with null
{
    public T CreateEmpty() => new T();  // ✓ new() guarantees this works
    
    public void Add(T entity)
    {
        if (entity is IAuditable auditable)
        {
            auditable.CreatedAt = DateTime.UtcNow;  // ✓ IAuditable guarantees this
        }
        // ...
    }
}
```

---

## Generic Methods

### Method-Level Type Parameters

A method can declare its own type parameters, independent of the class:

```csharp
public class Repository<T>
{
    // T comes from the class
    public T GetById(int id) { }
    
    // TOrderKey is specific to this method
    public IEnumerable<T> GetAllSorted<TOrderKey>(
        Func<T, TOrderKey> keySelector, 
        bool ascending = true)
    {
        return ascending 
            ? _data.OrderBy(keySelector)
            : _data.OrderByDescending(keySelector);
    }
}

// Usage - compiler infers TOrderKey automatically
var users = repo.GetAllSorted(u => u.Name);        // TOrderKey = string
var byDate = repo.GetAllSorted(u => u.CreatedAt);  // TOrderKey = DateTime
var byAge = repo.GetAllSorted(u => u.Age);         // TOrderKey = int
```

### Memory Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                    GENERIC METHOD INSTANTIATION                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Class: Repository<User>                                        │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │  GetAllSorted<string>(Func<User,string>)               │      │
│   │  GetAllSorted<DateTime>(Func<User,DateTime>)           │      │
│   │  GetAllSorted<int>(Func<User,int>)                     │      │
│   │                                                         │      │
│   │  Each method variant is JIT-compiled separately          │      │
│   │  for value types, shared for reference types            │      │
│   └─────────────────────────────────────────────────────────┘      │
│                                                                  │
│   Generic method dispatch:                                       │
│   1. Compiler infers TOrderKey from lambda return type           │
│   2. If inference fails, can specify explicitly:                │
│      repo.GetAllSorted<DateTime>(u => u.CreatedAt)            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Generic Inheritance

### Single-Level Inheritance

```csharp
public class GenericRepository<T, TKey> where T : BaseEntity<TKey>
{
    public virtual void Add(T entity) { }
    public virtual void Delete(TKey id) { }
}

public class ProductRepository : GenericRepository<Product, int>
{
    // Inherits all methods, types are fixed
}
```

### Multi-Level Generic Inheritance

```csharp
// Level 1: Generic base for all entities
public class GenericRepository<T, TKey> 
    where T : BaseEntity<TKey>
{
    protected List<T> _data = new();
    
    public virtual void Add(T entity)
    {
        _data.Add(entity);
    }
    
    public virtual void Delete(TKey id)
    {
        var entity = _data.FirstOrDefault(x => x.Id.Equals(id));
        if (entity != null)
            _data.Remove(entity);
    }
}

// Level 2: Middle layer for auditable entities
public class AuditableRepository<T, TKey> 
    : GenericRepository<T, TKey>
    where T : BaseEntity<TKey>, IAuditable
{
    public override void Add(T entity)
    {
        entity.CreatedAt = DateTime.UtcNow;
        base.Add(entity);
    }
    
    public override void Update(T entity)
    {
        entity.UpdatedAt = DateTime.UtcNow;
        base.Update(entity);
    }
}

// Level 3: Concrete repository
public class ProductRepository : AuditableRepository<Product, int>
{
    // Gets automatic timestamp handling from AuditableRepository
    // Gets CRUD operations from GenericRepository
    // Only adds Product-specific methods
    
    public IEnumerable<Product> GetByPriceRange(decimal min, decimal max)
    {
        return _data.Where(p => p.Price >= min && p.Price <= max);
    }
}
```

### Inheritance Chain Visualization

```
┌─────────────────────────────────────────────────────────────────┐
│                    GENERIC INHERITANCE CHAIN                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   GenericRepository<T, TKey>                                     │
│   ├─ Constraint: T : BaseEntity<TKey>                           │
│   ├─ Methods: Add, GetById, Delete, Update                    │
│   └─ Virtual methods can be overridden                          │
│              │                                                   │
│              │ Inherits, adds IAuditable constraint             │
│              ▼                                                   │
│   AuditableRepository<T, TKey>                                   │
│   ├─ Constraint: T : BaseEntity<TKey>, IAuditable              │
│   ├─ Overrides Add (adds timestamp)                            │
│   ├─ Overrides Update (adds timestamp)                           │
│   └─ Calls base for actual storage                              │
│              │                                                   │
│              │ Inherits, fixes T=Product, TKey=int               │
│              ▼                                                   │
│   ProductRepository                                              │
│   ├─ No constraints needed (fixed types)                       │
│   ├─ Gets: Add (with timestamp), Update (with timestamp)        │
│   └─ Adds: GetByPriceRange, GetByCategory                       │
│                                                                  │
│   UserRepository                                                 │
│   ├─ Inherits directly from GenericRepository                   │
│   └─ Does NOT get timestamp handling                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: Why use `where T : class` constraint?**
> Ensures T is a reference type, allowing null comparisons and ensuring `T?` nullable annotations work correctly.

**Q: When would you need multiple type parameters?**
> When the entity type and its key type need to vary independently (e.g., `Repository<Order, Guid>` vs `Repository<User, int>`).

**Q: What's the difference between `out T` and `in T`?**
> `out T` (covariant): T only appears as return type - safe to treat collection of derived as collection of base. `in T` (contravariant): T only appears as parameter - safe to treat handler of base as handler of derived.

**Q: Can you have both class and interface constraints?**
> Yes: `where T : BaseClass, IInterface1, IInterface2, new()`. Class must come first if present.

**Q: How does generic method type inference work?**
> Compiler examines argument types and infers method type parameters from them. Can be explicit if inference fails: `method<Type>(arg)`.
