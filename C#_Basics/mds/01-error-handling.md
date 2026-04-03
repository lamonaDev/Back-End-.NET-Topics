# C# Error Handling & Exceptions

## Table of Contents
1. [Throw vs Result Pattern](#throw-vs-result-pattern)
2. [InnerException](#innerexception)
3. [Custom Exceptions](#create-your-own-exception)
4. [InvalidOperationException](#invalidoperationexception)

---

## Throw vs Result Pattern

### Overview
Two approaches to handling errors: throwing exceptions vs returning result objects that indicate success or failure.

### Throw Pattern

**When to Use:**
- Exceptional circumstances (file not found, network failure)
- Programming errors (null reference, invalid cast)
- Unrecoverable conditions

**Code Example:**
```csharp
public class PaymentService
{
    public void ProcessPayment(decimal amount, string cardNumber)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive", nameof(amount));
        
        if (string.IsNullOrWhiteSpace(cardNumber))
            throw new ArgumentException("Card number is required", nameof(cardNumber));
        
        if (!IsValidCard(cardNumber))
            throw new InvalidOperationException("Invalid card number");
        
        // Process payment...
    }
}

// Usage
try
{
    paymentService.ProcessPayment(-100, "1234");
}
catch (ArgumentException ex)
{
    Console.WriteLine($"Validation error: {ex.Message}");
}
```

**Memory Visualization:**
```
Stack before throw:
┌─────────────────┐
│ ProcessPayment  │ ← Current frame
├─────────────────┤
│ Main            │
└─────────────────┘

Stack after throw:
┌─────────────────┐
│ Catch block     │ ← Unwound to handler
├─────────────────┤
│ Main            │
└─────────────────┘

Exception object (Heap):
┌─────────────────┐
│ Message         │ ──→ "Amount must be positive"
│ StackTrace      │ ──→ Call stack snapshot
│ InnerException  │ ──→ null
└─────────────────┘
```

### Result Pattern

**When to Use:**
- Expected failure cases (validation, business rules)
- API responses where failures are common
- When you want explicit error handling

**Code Example:**
```csharp
public record Result<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public string Error { get; }
    
    private Result(bool isSuccess, T value, string error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }
    
    public static Result<T> Success(T value) => 
        new Result<T>(true, value, null);
    
    public static Result<T> Failure(string error) => 
        new Result<T>(false, default, error);
}

public class OrderService
{
    public Result<Order> CreateOrder(string customerId, List<Item> items)
    {
        if (string.IsNullOrWhiteSpace(customerId))
            return Result<Order>.Failure("Customer ID is required");
        
        if (items == null || items.Count == 0)
            return Result<Order>.Failure("Order must contain at least one item");
        
        if (!HasSufficientInventory(items))
            return Result<Order>.Failure("Insufficient inventory");
        
        var order = new Order { CustomerId = customerId, Items = items };
        return Result<Order>.Success(order);
    }
}

// Usage - explicit handling
var result = orderService.CreateOrder("", new List<Item>());
if (!result.IsSuccess)
{
    Console.WriteLine($"Failed: {result.Error}");
    return;
}
Console.WriteLine($"Order created: {result.Value.Id}");
```

**Memory Visualization:**
```
Result object (Stack/Heap):
┌─────────────────────────┐
│ IsSuccess │ false       │
├─────────────────────────┤
│ Value     │ null        │
├─────────────────────────┤
│ Error     │ ──→ "Customer ID is required"
└─────────────────────────┘

No stack unwinding - control flow continues normally
```

### Comparison

| Aspect | Throw | Result Pattern |
|--------|-------|----------------|
| **Stack unwinding** | Yes | No |
| **Performance** | Slower (stack trace capture) | Faster |
| **Readability** | Try-catch blocks | Explicit checking |
| **Use case** | Exceptional | Expected failures |
| **Discoverability** | Stack trace | Error message |

**Real-World Example:**
```csharp
// API Controller - mixing both patterns
[HttpPost("orders")]
public IActionResult CreateOrder(CreateOrderRequest request)
{
    // Result pattern for validation
    var validationResult = ValidateRequest(request);
    if (!validationResult.IsSuccess)
        return BadRequest(validationResult.Error);
    
    try
    {
        // Throw for exceptional cases
        var order = _orderService.Save(validationResult.Value);
        return Ok(order);
    }
    catch (DatabaseException ex)
    {
        _logger.LogError(ex, "Database error");
        return StatusCode(500, "Internal server error");
    }
}
```

---

## InnerException

### Overview
Preserves the original exception when wrapping it in a higher-level exception.

### Why Use InnerException?
- Maintains full error chain
- Original stack trace preserved
- Context about where error originated

**Code Example:**
```csharp
public class OrderProcessingException : Exception
{
    public OrderProcessingException(string message, Exception inner) 
        : base(message, inner) { }
}

public class OrderProcessor
{
    public void Process(int orderId)
    {
        try
        {
            var order = _orderRepository.Get(orderId);    // May throw SqlException
            var payment = _paymentService.Charge(order);  // May throw PaymentException
            _notificationService.SendConfirmation(order); // May throw SmtpException
        }
        catch (SqlException ex)
        {
            throw new OrderProcessingException(
                $"Failed to retrieve order {orderId}", ex);
        }
        catch (PaymentException ex)
        {
            throw new OrderProcessingException(
                $"Payment failed for order {orderId}", ex);
        }
    }
}

// Usage
try
{
    processor.Process(123);
}
catch (OrderProcessingException ex)
{
    Console.WriteLine($"User-friendly: {ex.Message}");
    Console.WriteLine($"Root cause: {ex.InnerException?.Message}");
    Console.WriteLine($"Full stack:\n{ex.InnerException?.StackTrace}");
}
```

**Memory Visualization:**
```
Exception Chain (Heap):

OrderProcessingException
├─ Message: "Payment failed for order 123"
├─ StackTrace: (OrderProcessor.Process)
└─ InnerException ──→ PaymentException
                        ├─ Message: "Card declined: insufficient funds"
                        ├─ StackTrace: (PaymentService.Charge)
                        └─ InnerException ──→ BankApiException
                                                ├─ Message: "Error code: INSUFFICIENT_FUNDS"
                                                └─ StackTrace: (BankClient.Post)
```

**Real-World Example:**
```csharp
// Logging service preserves full chain
public void LogException(Exception ex, string context)
{
    var sb = new StringBuilder();
    sb.AppendLine($"Context: {context}");
    
    var current = ex;
    int depth = 0;
    while (current != null)
    {
        sb.AppendLine($"[{depth}] {current.GetType().Name}: {current.Message}");
        current = current.InnerException;
        depth++;
    }
    
    _logger.LogError(sb.ToString());
}

// Output:
// Context: OrderProcessing
// [0] OrderProcessingException: Payment failed for order 123
// [1] PaymentException: Card declined: insufficient funds
// [2] BankApiException: Error code: INSUFFICIENT_FUNDS
```

---

## Create Your Own Exception

### Why Create Custom Exceptions?
- Domain-specific error types
- Better error handling granularity
- Self-documenting code

**Code Example:**
```csharp
// Custom exception hierarchy
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    protected DomainException(string message, Exception inner) : base(message, inner) { }
}

public class InsufficientFundsException : DomainException
{
    public decimal Available { get; }
    public decimal Required { get; }
    
    public InsufficientFundsException(decimal available, decimal required)
        : base($"Insufficient funds: available {available:C}, required {required:C}")
    {
        Available = available;
        Required = required;
    }
}

public class ProductNotFoundException : DomainException
{
    public string ProductId { get; }
    
    public ProductNotFoundException(string productId)
        : base($"Product '{productId}' not found")
    {
        ProductId = productId;
    }
}

// Usage
public class ShoppingCart
{
    private readonly IProductRepository _products;
    
    public void AddItem(string productId, int quantity)
    {
        var product = _products.Get(productId);
        if (product == null)
            throw new ProductNotFoundException(productId);
        
        if (!product.IsInStock)
            throw new OutOfStockException(productId);
        
        // ...
    }
}

// Handling
catch (ProductNotFoundException ex)
{
    return NotFound($"Product {ex.ProductId} was removed from catalog");
}
catch (InsufficientFundsException ex)
{
    return BadRequest(new { 
        Message = ex.Message,
        Available = ex.Available,
        Required = ex.Required
    });
}
```

**Best Practices:**
1. Inherit from `Exception` or appropriate base
2. Provide constructors matching base class
3. Include additional properties for context
4. Use `[Serializable]` if needed
5. Follow naming convention: `[Name]Exception`

---

## InvalidOperationException

### Overview
Thrown when object state doesn't support an operation.

**When to Use:**
- Object not properly initialized
- Operation called at wrong time
- State machine in invalid state

**Code Example:**
```csharp
public class OrderProcessor
{
    private bool _isInitialized = false;
    private PaymentGateway _gateway;
    
    public void Initialize(PaymentGateway gateway)
    {
        _gateway = gateway ?? throw new ArgumentNullException(nameof(gateway));
        _isInitialized = true;
    }
    
    public void ProcessPayment(Order order)
    {
        if (!_isInitialized)
            throw new InvalidOperationException(
                "OrderProcessor must be initialized before processing payments. " +
                "Call Initialize() first.");
        
        if (order.Status != OrderStatus.Pending)
            throw new InvalidOperationException(
                $"Cannot process order with status {order.Status}. " +
                "Only Pending orders can be processed.");
        
        _gateway.Charge(order.Total);
    }
}

// Enumerator example
public class CustomEnumerator<T>
{
    private T[] _items;
    private int _index = -1;
    
    public T Current
    {
        get
        {
            if (_index < 0 || _index >= _items.Length)
                throw new InvalidOperationException(
                    "Enumeration has not started or has already finished. " +
                    "Call MoveNext() before accessing Current.");
            return _items[_index];
        }
    }
}

// LINQ example
var emptyList = new List<int>();
var first = emptyList.First(); // Throws InvalidOperationException: Sequence contains no elements
var firstOrDefault = emptyList.FirstOrDefault(); // Returns 0, no exception
```

**Comparison with ArgumentException:**

| Exception | When to Throw | Example |
|-----------|---------------|---------|
| **ArgumentException** | Invalid parameter value | `null` passed for required parameter |
| **ArgumentNullException** | `null` parameter not allowed | Same, specifically for null |
| **InvalidOperationException** | Object in wrong state | Method called before initialization |

**Real-World Example:**
```csharp
// Entity Framework - DbContext
public class Repository<T> where T : class
{
    private readonly DbContext _context;
    private DbSet<T> _dbSet;
    
    private DbSet<T> Set
    {
        get
        {
            if (_context == null)
                throw new InvalidOperationException(
                    "Repository is not initialized. Ensure DbContext is set.");
            return _dbSet ??= _context.Set<T>();
        }
    }
    
    public T GetById(int id)
    {
        return Set.Find(id) ?? throw new InvalidOperationException(
            $"Entity of type {typeof(T).Name} with id {id} was not found.");
    }
}
```

---

*Source: C# documentation, exception handling best practices, and .NET design guidelines.*
