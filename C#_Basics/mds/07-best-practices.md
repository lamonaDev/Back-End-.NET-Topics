# C# Best Practices

## Table of Contents
1. [Early Exit & Guard Clauses](#early-exit--guard-clauses)
2. [Why Returning Null is Bad](#why-returning-null-is-bad)
3. [Parse vs Convert vs TryParse](#parse-vs-convert-vs-tryparse)

---

## Early Exit & Guard Clauses

### Overview
Early exit pattern reduces nesting and improves readability by handling edge cases first.

### The Problem: Deep Nesting
```csharp
// Hard to read - pyramid of doom
public void ProcessOrder(Order order)
{
    if (order != null)
    {
        if (order.Items != null)
        {
            if (order.Items.Count > 0)
            {
                if (order.IsPaid)
                {
                    // Finally do work
                    ProcessItems(order.Items);
                }
                else
                {
                    throw new InvalidOperationException("Order not paid");
                }
            }
            else
            {
                throw new ArgumentException("No items in order");
            }
        }
        else
        {
            throw new ArgumentException("Items collection is null");
        }
    }
    else
    {
        throw new ArgumentNullException(nameof(order));
    }
}
```

### The Solution: Guard Clauses
```csharp
// Clean with early exits
public void ProcessOrder(Order order)
{
    // Guard clauses - fail fast
    if (order == null)
        throw new ArgumentNullException(nameof(order));
    
    if (order.Items == null)
        throw new ArgumentException("Items collection is null", nameof(order));
    
    if (order.Items.Count == 0)
        throw new ArgumentException("No items in order", nameof(order));
    
    if (!order.IsPaid)
        throw new InvalidOperationException("Order not paid");
    
    // Main logic - now unindented and clear
    ProcessItems(order.Items);
}
```

**Memory Visualization:**
```
Execution flow with guard clauses:

Stack:
┌─────────────────────────────────────────────────┐
│ ProcessOrder called                           │
├─────────────────────────────────────────────────┤
│ Check order == null? ──→ Yes ──→ Throw       │
│ (early exit, stack unwinds)                     │
├─────────────────────────────────────────────────┤
│ Check order.Items == null? ──→ Yes ──→ Throw │
│ (early exit, stack unwinds)                     │
├─────────────────────────────────────────────────┤
│ Check Count == 0? ──→ Yes ──→ Throw           │
│ (early exit, stack unwinds)                     │
├─────────────────────────────────────────────────┤
│ All checks passed                               │
│ Execute ProcessItems                            │
│ Return normally                                 │
└─────────────────────────────────────────────────┘
```

### Guard Clause Patterns

**Null Checks:**
```csharp
public void SendEmail(string to, string subject, string body)
{
    if (string.IsNullOrWhiteSpace(to))
        throw new ArgumentException("Email required", nameof(to));
    
    if (string.IsNullOrWhiteSpace(subject))
        throw new ArgumentException("Subject required", nameof(subject));
    
    if (body == null)
        body = string.Empty;  // Or throw, depending on requirements
    
    // Safe to proceed
    _emailService.Send(to, subject, body);
}
```

**Range Checks:**
```csharp
public decimal CalculateDiscount(int age, decimal amount)
{
    if (age < 0 || age > 150)
        throw new ArgumentOutOfRangeException(nameof(age), "Invalid age");
    
    if (amount < 0)
        throw new ArgumentException("Amount cannot be negative", nameof(amount));
    
    if (amount == 0)
        return 0;  // Early return, no discount
    
    // Calculate discount
    return age > 65 ? amount * 0.10m : amount * 0.05m;
}
```

**Authorization Checks:**
```csharp
public async Task<Order> GetOrderAsync(int orderId, User user)
{
    if (user == null)
        throw new UnauthorizedAccessException();
    
    if (!user.HasPermission(Permission.ViewOrders))
        throw new ForbiddenException("Cannot view orders");
    
    var order = await _repository.GetAsync(orderId);
    
    if (order == null)
        return null;  // Not found is valid result
    
    if (order.UserId != user.Id && !user.IsAdmin)
        throw new ForbiddenException("Cannot view this order");
    
    return order;
}
```

### Benefits

| Aspect | Nested If | Guard Clauses |
|--------|-----------|---------------|
| **Cyclomatic complexity** | High | Lower |
| **Nesting depth** | Deep | Flat |
| **Error location** | Buried | Immediate |
| **Main logic visibility** | Hidden | Prominent |
| **Testability** | Harder | Easier |

**Real-World Example:**
```csharp
public class OrderHandler : IRequestHandler<CreateOrderRequest, Order>
{
    private readonly IOrderRepository _repository;
    private readonly IValidator<CreateOrderRequest> _validator;
    private readonly IEventPublisher _events;
    
    public async Task<Order> Handle(CreateOrderRequest request)
    {
        // Validation guards
        if (request == null)
            throw new ArgumentNullException(nameof(request));
        
        var validation = await _validator.ValidateAsync(request);
        if (!validation.IsValid)
            throw new ValidationException(validation.Errors);
        
        // Business rule guards
        if (request.Items.Count > 100)
            throw new OrderLimitExceededException("Max 100 items per order");
        
        if (request.Total > 10000m)
            throw new OrderLimitExceededException("Max $10,000 per order");
        
        // State guards
        var customer = await _repository.GetCustomerAsync(request.CustomerId);
        if (customer?.IsActive != true)
            throw new InvalidOperationException("Customer not active");
        
        if (customer.HasOutstandingBalance)
            throw new InvalidOperationException("Outstanding balance must be cleared");
        
        // Main processing - now clearly the "happy path"
        var order = Order.Create(request);
        await _repository.SaveAsync(order);
        await _events.PublishAsync(new OrderCreatedEvent(order));
        
        return order;
    }
}
```

---

## Why Returning Null is Bad

### Overview
Null returns create ambiguity and force defensive null checks throughout the codebase.

### The Null Problem
```csharp
// Ambiguous: null means what?
public User GetUser(int id)
{
    var user = _db.Users.Find(id);
    return user;  // null if not found
}

// Caller confusion
var user = GetUser(123);

if (user == null)
{
    // Did user not exist?
    // Was there a database error?
    // Was the ID invalid?
    // Do nothing? Throw? Create?
}

// Defensive programming burden
string DisplayName = user?.Name ?? "Unknown";
int Age = user?.Age ?? 0;  // Is 0 valid or missing?
```

**Memory Visualization:**
```
Null reference chain:

┌─────────────────────────────────────────────────┐
│ Service.GetUser(123)                            │
│   ↓                                             │
│ Repository.Find(123)                            │
│   ↓                                             │
│ Database: No record found                       │
│   ↓                                             │
│ Returns: null                                     │
│   ↓                                             │
│ Controller: Receives null                       │
│   ↓                                             │
│ ?. operator propagates null                     │
│ ?? provides default                             │
└─────────────────────────────────────────────────┘

Problem: Information lost - why was it null?
```

### Better Alternatives

**Option 1: Result Pattern**
```csharp
public Result<User> GetUser(int id)
{
    var user = _db.Users.Find(id);
    
    if (user == null)
        return Result<User>.Failure($"User {id} not found");
    
    return Result<User>.Success(user);
}

// Usage
var result = GetUser(123);
if (!result.IsSuccess)
{
    _logger.LogWarning(result.Error);  // Clear reason
    return NotFound(result.Error);
}
return Ok(result.Value);
```

**Option 2: Try Pattern**
```csharp
public bool TryGetUser(int id, out User user)
{
    user = _db.Users.Find(id);
    return user != null;
}

// Usage
if (!TryGetUser(123, out var user))
{
    return NotFound();
}
// user is guaranteed non-null here
```

**Option 3: Null Object Pattern**
```csharp
// Always return a valid object
public User GetUser(int id)
{
    return _db.Users.Find(id) ?? User.Guest;  // Guest is non-null default
}

public class User
{
    public static readonly User Guest = new User 
    { 
        Id = 0, 
        Name = "Guest",
        CanEdit = false,
        CanDelete = false
    };
    
    // Null-safe operations
    public virtual bool CanPerform(Action action) => false;
}

// Usage - no null checks needed
var user = GetUser(123);
if (!user.CanPerform(Action.Edit))
{
    return Forbid();
}
```

**Option 4: Optional/Maybe Pattern**
```csharp
public Maybe<User> GetUser(int id)
{
    var user = _db.Users.Find(id);
    return user != null 
        ? Maybe<User>.Some(user) 
        : Maybe<User>.None;
}

// Usage with LINQ-like operations
GetUser(123)
    .Map(u => u.Email)
    .Filter(email => email.Contains("@"))
    .Match(
        some: email => SendNotification(email),
        none: () => _logger.Log("No user or email")
    );
```

### When Null Might Be Okay

| Scenario | Alternative |
|----------|-------------|
| **Optional configuration** | Options pattern |
| **Caching** | `TryGetValue` pattern |
| **Async operations** | `Task<T?>` with nullable reference types |
| **Legacy interop** | Document with null annotations |

**Real-World Example:**
```csharp
// Before: Null returns
public Invoice? GetInvoice(int orderId)
{
    return _invoices.FirstOrDefault(i => i.OrderId == orderId);
}

// After: Clear semantics with Result
public Result<Invoice> GetInvoice(int orderId)
{
    var invoice = _invoices.FirstOrDefault(i => i.OrderId == orderId);
    
    if (invoice == null)
        return Result<Invoice>.Failure($"No invoice for order {orderId}");
    
    if (invoice.Status == InvoiceStatus.Void)
        return Result<Invoice>.Failure($"Invoice {invoice.Id} is void");
    
    return Result<Invoice>.Success(invoice);
}

// API Controller - clean handling
[HttpGet("orders/{orderId}/invoice")]
public IActionResult GetInvoice(int orderId)
{
    var result = _service.GetInvoice(orderId);
    
    return result.Match(
        onSuccess: invoice => Ok(invoice),
        onFailure: error => error.Contains("void") 
            ? Conflict(error) 
            : NotFound(error)
    );
}
```

---

## Parse vs Convert vs TryParse

### Overview
Three approaches to converting strings to other types with different error handling strategies.

### Comparison

| Method | Throws on Failure | Returns on Failure | Use When |
|--------|-----------------|-------------------|----------|
| `Parse` | Yes | N/A | Input guaranteed valid |
| `Convert` | Yes | N/A | General conversion |
| `TryParse` | No | false + default | Input might be invalid |

### Parse
```csharp
// Throws FormatException if invalid
int number = int.Parse("123");      // Success: 123
int error = int.Parse("abc");       // Throws FormatException
int overflow = int.Parse("99999999999999999999");  // OverflowException
int nullRef = int.Parse(null);      // ArgumentNullException

// With styles
decimal currency = decimal.Parse("$1,234.56", NumberStyles.Currency, CultureInfo.InvariantCulture);
```

### Convert
```csharp
// More flexible, still throws
int fromString = Convert.ToInt32("123");
int fromObject = Convert.ToInt32(someObject);  // Handles null → 0
int fromBool = Convert.ToInt32(true);          // Returns 1

// Returns 0 for null (not exception)
int nullSafe = Convert.ToInt32(null);  // Returns 0

// Still throws on invalid format
int error = Convert.ToInt32("abc");  // FormatException
```

### TryParse (Recommended)
```csharp
// Pattern matching style (C# 7+)
if (int.TryParse(input, out int result))
{
    Console.WriteLine($"Success: {result}");
}
else
{
    Console.WriteLine("Invalid number");
}

// Variable declaration inline
if (DateTime.TryParse(dateString, out var date))
{
    // Use date
}

// Using the result (C# 8+ nullable reference types)
if (int.TryParse(input, out int value))
    Process(value);  // value is definitely assigned
```

**Memory Visualization:**
```
Parse exception flow:

┌─────────────────────────────────────────────────┐
│ int.Parse("abc")                                │
│   ↓                                             │
│ FormatException created on heap                │
│ ┌─────────────────────────────────────┐          │
│ │ Exception object                    │          │
│ │ ├─ Message: "Input string was..." │          │
│ │ ├─ Stack trace                      │          │
│ │ └─ Inner exception: null           │          │
│ └─────────────────────────────────────┘          │
│   ↓                                             │
│ Stack unwinding (expensive)                     │
│   ↓                                             │
│ Catch block (if present)                        │
└─────────────────────────────────────────────────┘

TryParse flow:
┌─────────────────────────────────────────────────┐
│ int.TryParse("abc", out result)                 │
│   ↓                                             │
│ Returns: false                                   │
│ result: 0 (default)                              │
│ No exception, no allocation                     │
│ Minimal overhead                                │
└─────────────────────────────────────────────────┘
```

### Performance Comparison
```csharp
// Benchmark: Parse 1M invalid strings
| Method     | Mean     | Allocated |
|------------|----------|----------|
| Parse      | 500 ms   | 400 MB   |  // Exceptions are expensive!
| TryParse   | 10 ms    | 0 B      |  // No allocations

// Valid strings: similar performance
```

### Real-World Patterns

**Configuration Parsing:**
```csharp
public class ConfigurationParser
{
    public int GetIntValue(string key, int defaultValue)
    {
        string value = _config[key];
        
        // TryParse with fallback
        return int.TryParse(value, out int result) 
            ? result 
            : defaultValue;
    }
    
    public TimeSpan GetTimeout(string key)
    {
        string value = _config[key];
        
        // Multiple parsing attempts
        if (int.TryParse(value, out int ms))
            return TimeSpan.FromMilliseconds(ms);
        
        if (TimeSpan.TryParse(value, out TimeSpan ts))
            return ts;
        
        throw new ConfigurationException($"Invalid timeout format: {value}");
    }
}
```

**Input Validation:**
```csharp
public class UserInputValidator
{
    public ValidationResult ValidateAge(string input)
    {
        if (!int.TryParse(input, out int age))
            return ValidationResult.Error("Age must be a number");
        
        if (age < 0 || age > 150)
            return ValidationResult.Error("Age must be between 0 and 150");
        
        return ValidationResult.Success(age);
    }
    
    public ValidationResult ValidateDate(string input)
    {
        if (!DateTime.TryParseExact(
            input, 
            "yyyy-MM-dd", 
            CultureInfo.InvariantCulture,
            DateTimeStyles.None,
            out DateTime date))
        {
            return ValidationResult.Error("Date must be in YYYY-MM-DD format");
        }
        
        if (date > DateTime.Now)
            return ValidationResult.Error("Date cannot be in the future");
        
        return ValidationResult.Success(date);
    }
}
```

**Best Practice Summary:**
```csharp
// ❌ Don't: Parse with try-catch for control flow
try
{
    return int.Parse(input);
}
catch (FormatException)
{
    return 0;
}

// ✅ Do: TryParse for user input
if (int.TryParse(input, out int value))
    return value;
return 0;

// ✅ Do: Parse for known-good data
var id = int.Parse(knownValidId);  // From database, not user

// ✅ Do: Convert for flexibility with null handling
var count = Convert.ToInt32(maybeNullObject);
```

---

*Source: C# coding standards, defensive programming practices, and performance optimization guides.*
