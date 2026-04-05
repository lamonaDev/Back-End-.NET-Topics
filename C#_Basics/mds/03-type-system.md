# C# Type System Deep Dive

## Table of Contents
1. [Readonly with Value vs Reference Types](#readonly-with-value-vs-reference-types)
2. [Immutable vs Mutable Types](#immutable-vs-mutable-types)
3. [Method Tables / VTables](#method-tables--vtables)
4. [Private Fields as readonly](#why-prefer-private-fields-to-be-readonly)

---

## Readonly with Value vs Reference Types

### Overview
The `readonly` keyword has different semantics for value types (structs) versus reference types (classes).

### Value Types (struct)

**Behavior:**
- The field value cannot be reassigned
- The value itself is stored inline
- For structs: prevents replacing the entire struct

**Code Example:**
```csharp
public struct Point
{
    public int X { get; set; }
    public int Y { get; set; }
}

public class ValueTypeReadonlyDemo
{
    // Value type - the struct cannot be replaced
    private readonly Point _origin;
    
    public ValueTypeReadonlyDemo()
    {
        _origin = new Point { X = 0, Y = 0 };
    }
    
    public void TryModify()
    {
        // ❌ Error: readonly field cannot be assigned
        // _origin = new Point { X = 10, Y = 10 };
        
        // ⚠️ But properties CAN be modified! (surprising behavior)
        _origin.X = 10;  // This compiles but...
        // Actually: CS1650 in older C#, but newer versions allow it
        // Best practice: make struct properties readonly too
    }
}

// Better: fully immutable struct
public readonly struct ImmutablePoint
{
    public int X { get; }
    public int Y { get; }
    
    public ImmutablePoint(int x, int y)
    {
        X = x;
        Y = y;
    }
}
```

**Memory Visualization:**
```
Class with readonly struct field:

┌─────────────────────────────────────────────┐
│ Instance on Heap:                           │
│ ┌────────────────────────┐                  │
│ │ Object header          │                  │
│ │ Method table ptr       │                  │
│ │ _origin (Point struct) │                  │
│ │   ├── X: 0 (inline)    │                  │
│ │   └── Y: 0 (inline)    │                  │
│ └────────────────────────┘                  │
└─────────────────────────────────────────────┘

readonly prevents: _origin = new Point()  // Error
But if mutable: _origin.X = 10 works    // Warning!
```

### Reference Types (class)

**Behavior:**
- The reference cannot be reassigned
- The object being referenced CAN be modified
- Only the pointer is protected

**Code Example:**
```csharp
public class User
{
    public string Name { get; set; }
    public List<string> Roles { get; set; }
}

public class ReferenceTypeReadonlyDemo
{
    // Reference type - reference cannot change
    private readonly User _adminUser;
    
    public ReferenceTypeReadonlyDemo()
    {
        _adminUser = new User { Name = "Admin" };
        _adminUser.Roles = new List<string> { "Admin", "SuperUser" };
    }
    
    public void Modify()
    {
        // ❌ Error: cannot reassign readonly field
        // _adminUser = new User { Name = "Other" };
        
        // ✅ Allowed: modifying the object itself
        _adminUser.Name = "SuperAdmin";
        _adminUser.Roles.Add("PowerUser");
        
        // The object is mutable even though reference is readonly
    }
}
```

**Memory Visualization:**
```
Class with readonly reference field:

Stack (reference):
┌───────────────────────────┐
│ _adminUser reference      │ ──→ cannot be changed
└───────────────────────────┘
              │
              ↓
Heap (object):
┌───────────────────────────┐
│ User object               │
│ ├─ Name ──→ "SuperAdmin" │  ← CAN modify this
│ └─ Roles ──→ List        │  ← CAN modify this
└───────────────────────────┘

readonly prevents: _adminUser = new User()  // Error
But: _adminUser.Name = "New" works          // Allowed!
```

### Summary Comparison

| Aspect | Value Type (readonly) | Reference Type (readonly) |
|--------|----------------------|---------------------------|
| **Prevents** | Reassignment | Reference reassignment |
| **Object mutation** | Depends on struct design | Always possible |
| **Thread safety** | Partial (if immutable struct) | Not guaranteed |
| **Memory** | Inline (part of container) | Reference stored inline |

**Real-World Example:**
```csharp
public class OrderService
{
    // Reference: injected dependency, shouldn't be replaced
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;
    
    // Value: configuration value, fixed after construction
    private readonly TimeSpan _processingTimeout;
    
    public OrderService(
        IOrderRepository repository,
        ILogger<OrderService> logger)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _processingTimeout = TimeSpan.FromMinutes(5);
    }
    
    // These fields are stable throughout object lifetime
    // but the repository's internal state can change
}
```

---

## Immutable vs Mutable Types

### Overview
Immutable types cannot be changed after creation; mutable types can be modified in place.

### Immutable Types

**Examples:**
- `string`
- `DateTime`
- `TimeSpan`
- `decimal`
- `Uri`
- `Tuple` (C# 7+)

**Code Example:**
```csharp
// String immutability
customerName = "John";
customerName = customerName.ToUpper();  // Creates NEW string
// customerName is now "JOHN", original "John" is discarded

// DateTime immutability
var meeting = new DateTime(2024, 6, 15, 10, 0, 0);
var newMeeting = meeting.AddHours(2);  // Returns NEW DateTime
// meeting is still 10:00, newMeeting is 12:00

// Creating custom immutable types
public sealed class ImmutablePerson
{
    public string Name { get; }
    public int Age { get; }
    public IReadOnlyList<string> Tags { get; }
    
    public ImmutablePerson(string name, int age, IEnumerable<string> tags)
    {
        Name = name;
        Age = age;
        Tags = tags?.ToList().AsReadOnly() ?? 
               new List<string>().AsReadOnly();
    }
    
    // Return new instance instead of modifying
    public ImmutablePerson WithAge(int newAge) => 
        new ImmutablePerson(Name, newAge, Tags);
    
    public ImmutablePerson AddTag(string tag)
    {
        var newTags = Tags.ToList();
        newTags.Add(tag);
        return new ImmutablePerson(Name, Age, newTags);
    }
}
```

**Memory Visualization:**
```
Immutable operations create new instances:

Original: "Hello"
┌─────────────────────────────────────────────────┐
│ Heap: "Hello" (interned)                        │
└─────────────────────────────────────────────────┘

After ToUpper(): "HELLO"
┌─────────────────────────────────────────────────┐
│ Heap: "Hello" (still exists if referenced)      │
│ Heap: "HELLO" (new string)                    │
│                                                 │
│ Reference updated to point to "HELLO"           │
└─────────────────────────────────────────────────┘

Note: Original may be GC'd if no other references
```

### Mutable Types

**Examples:**
- `List<T>`
- `Dictionary<K,V>`
- `StringBuilder`
- `Array`
- Custom classes with setters

**Code Example:**
```csharp
// List mutability
var items = new List<string> { "A", "B" };
items.Add("C");        // Modifies in place
items[0] = "Z";        // Modifies in place
items.Remove("B");     // Modifies in place
// items is now ["Z", "C"]

// StringBuilder mutability
var sb = new StringBuilder("Hello");
sb.Append(" World");   // Modifies internal buffer
sb.Replace("World", "C#");  // Modifies in place
// sb now contains "Hello C#"

// Custom mutable type
public class MutableOrder
{
    public string Status { get; set; }
    public List<OrderItem> Items { get; set; }
    
    public void AddItem(OrderItem item) => Items.Add(item);
    public void UpdateStatus(string status) => Status = status;
}
```

**Memory Visualization:**
```
Mutable operations modify in place:

List before:
┌─────────────────────────────────────────────────┐
│ items (reference) ──→ List object               │
│                         ┌──────────────┐       │
│                         │ ["A", "B"]   │       │
│                         └──────────────┘       │
└─────────────────────────────────────────────────┘

After items.Add("C"):
┌─────────────────────────────────────────────────┐
│ items (same reference) ──→ List object            │
│                              ┌─────────────────┐ │
│                              │ ["A", "B", "C"] │ │
│                              └─────────────────┘ │
│ Same object, modified content                   │
└─────────────────────────────────────────────────┘
```

### Trade-offs

| Aspect | Immutable | Mutable |
|--------|-----------|---------|
| **Thread safety** | Safe by default | Requires synchronization |
| **Reasoning** | Easier to understand | Harder to track changes |
| **Performance** | More allocations | Less GC pressure |
| **Memory** | Duplication | Shared state risks |
| **Chaining** | Fluent APIs | Less elegant |

**Real-World Example:**
```csharp
// Immutable approach (preferred for domain models)
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }
}

// Usage
var price = new Money(100, "USD");
var tax = new Money(10, "USD");
var total = price.Add(tax);  // New Money instance, price unchanged
```

---

## Method Tables / VTables

### Overview
Mechanism enabling polymorphism through virtual method dispatch at runtime.

### What is a VTable?
A per-class table of function pointers that maps virtual methods to their implementations.

**Memory Layout:**
```
Object Memory Layout:

┌─────────────────────────────────────────────────┐
│ Object Header (8-16 bytes)                      │
│   - Sync block index                            │
│   - Hash code cache                             │
├─────────────────────────────────────────────────┤
│ Method Table Pointer                            │ ──→ Points to type info
├─────────────────────────────────────────────────┤
│ Field 1                                         │
│ Field 2                                         │
│ ...                                             │
└─────────────────────────────────────────────────┘

Method Table Structure:
┌─────────────────────────────────────────────────┐
│ Method Table Header                             │
├─────────────────────────────────────────────────┤
│ Base Method Table Pointer                       │
├─────────────────────────────────────────────────┤
│ Virtual Method Slot 0 ──→ BaseClass.MethodA   │
│ Virtual Method Slot 1 ──→ BaseClass.MethodB     │
│ Virtual Method Slot 2 ──→ DerivedClass.MethodC  │ ← Overridden
│ Virtual Method Slot 3 ──→ DerivedClass.MethodD │ ← New
└─────────────────────────────────────────────────┘
```

### How Virtual Dispatch Works

**Code Example:**
```csharp
public class Animal
{
    public virtual void Speak() => Console.WriteLine("...");
    public void Sleep() => Console.WriteLine("Zzz");
}

public class Dog : Animal
{
    public override void Speak() => Console.WriteLine("Woof!");
    public void Fetch() => Console.WriteLine("Fetching");
}

public class Cat : Animal
{
    public override void Speak() => Console.WriteLine("Meow!");
}

// Virtual dispatch
Animal myPet = GetPet();  // Could be Dog or Cat at runtime
myPet.Speak();  // Resolved via VTable at runtime

// Non-virtual method
myPet.Sleep();  // Direct call, no VTable lookup needed
```

**Dispatch Process:**
```
Virtual Call: myPet.Speak()

1. Get method table pointer from object
   obj → [Header][MethodTablePtr][Fields...]
                │
                ↓
2. Follow pointer to method table
   MethodTable → [Slot 0: Animal.Speak][Slot 1: Sleep]...
   
3. If Dog instance, slot 0 points to Dog.Speak
   If Cat instance, slot 0 points to Cat.Speak
   
4. Jump to actual implementation

Non-Virtual Call: myPet.Sleep()

1. Compiler knows exact method at compile time
2. Direct call to Animal.Sleep
3. No VTable lookup needed (faster)
```

### Performance Considerations

| Aspect | Virtual | Non-Virtual |
|--------|---------|-------------|
| **Call overhead** | VTable lookup | Direct jump |
| **Inlinability** | Harder | Easier |
| **Flexibility** | Polymorphism | Fixed behavior |
| **Cache** | May miss | Better prediction |

**Real-World Example:**
```csharp
// Design for extensibility with virtual methods
public abstract class PaymentProcessor
{
    // Virtual: allows customization
    public virtual async Task<PaymentResult> ProcessAsync(PaymentRequest request)
    {
        await ValidateAsync(request);
        var result = await ExecuteAsync(request);
        await LogAsync(result);
        return result;
    }
    
    // Abstract: must implement
    protected abstract Task<PaymentResult> ExecuteAsync(PaymentRequest request);
    
    // Sealed: performance-critical, no override needed
    protected sealed Task ValidateAsync(PaymentRequest request)
    {
        // Validation logic
        return Task.CompletedTask;
    }
}

public class StripeProcessor : PaymentProcessor
{
    protected override async Task<PaymentResult> ExecuteAsync(PaymentRequest request)
    {
        // Stripe-specific implementation
        return await _stripeClient.Charges.CreateAsync(...);
    }
}
```

---

## Why Prefer Private Fields to Be readonly

### Benefits

#### 1. Prevents Accidental Mutation
```csharp
// Without readonly
public class OrderProcessor
{
    private IOrderRepository _repository;  // Can be reassigned accidentally
    
    public OrderProcessor(IOrderRepository repository)
    {
        _repository = repository;
    }
    
    public void Initialize()  // Called later
    {
        _repository = new CachedRepository(_repository);  // Allowed
        // What if someone else holds reference to old repository?
    }
}

// With readonly
public class OrderProcessorSafe
{
    private readonly IOrderRepository _repository;  // Cannot be reassigned
    private readonly ILogger<OrderProcessor> _logger;
    
    public OrderProcessorSafe(
        IOrderRepository repository,
        ILogger<OrderProcessor> logger)
    {
        _repository = repository;
        _logger = logger;
    }
    
    // Impossible to accidentally reassign in methods
}
```

#### 2. Thread Safety
```csharp
public class Configuration
{
    // Thread-safe: values cannot change after construction
    private readonly string _apiKey;
    private readonly TimeSpan _timeout;
    
    public Configuration(string apiKey, TimeSpan timeout)
    {
        _apiKey = apiKey;
        _timeout = timeout;
    }
    
    // Safe to share across threads
    public string ApiKey => _apiKey;
}
```

#### 3. Self-Documenting Code
```csharp
// Reader knows: these don't change after construction
public class HttpClientWrapper
{
    private readonly HttpClient _httpClient;
    private readonly JsonSerializerOptions _jsonOptions;
    private readonly string _baseUrl;
    
    // Immediately clear: all set in constructor, stable afterwards
}
```

#### 4. Encourages Good Design
```csharp
// Forces dependency injection pattern
public class Service
{
    // Must be set in constructor
    private readonly IDependency _dependency;
    
    public Service(IDependency dependency)
    {
        _dependency = dependency ?? throw new ArgumentNullException(nameof(dependency));
    }
    
    // No default constructor workaround
    // No property injection needed
    // Clear dependency graph
}
```

### When NOT to Use readonly

| Scenario | Reason |
|----------|--------|
| **Lazy initialization** | Need to assign on first use |
| **Object pooling** | Objects reused with new state |
| **Serialization** | Deserializers may need to set fields |
| **Caching** | Cache needs to be replaceable |

**Real-World Example:**
```csharp
public class DocumentService
{
    // Dependencies - immutable after construction
    private readonly IDocumentRepository _repository;
    private readonly ILogger<DocumentService> _logger;
    private readonly IFileStorage _storage;
    
    // Configuration - immutable
    private readonly TimeSpan _uploadTimeout;
    private readonly long _maxFileSize;
    
    // Mutable: cache that can be refreshed
    private List<DocumentType>? _cachedTypes;
    private DateTime _cacheExpiry;
    
    public DocumentService(
        IDocumentRepository repository,
        ILogger<DocumentService> logger,
        IFileStorage storage,
        IOptions<DocumentOptions> options)
    {
        _repository = repository;
        _logger = logger;
        _storage = storage;
        _uploadTimeout = options.Value.UploadTimeout;
        _maxFileSize = options.Value.MaxFileSize;
    }
    
    public async Task<IReadOnlyList<DocumentType>> GetTypesAsync()
    {
        if (_cachedTypes == null || DateTime.Now > _cacheExpiry)
        {
            _cachedTypes = await _repository.GetTypesAsync();
            _cacheExpiry = DateTime.Now.AddHours(1);
        }
        return _cachedTypes;
    }
}
```

---

*Source: C# language specification, CLR internals documentation, and object-oriented design principles.*
