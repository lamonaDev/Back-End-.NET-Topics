# Encapsulation in C#

## What is Encapsulation?

Encapsulation is the **bundling of data (variables) and methods (functions) that operate on that data into a single unit**, while restricting direct access to some of the object's components. It is one of the four fundamental pillars of Object-Oriented Programming (OOP).

> **In simple terms:** Encapsulation means "hiding the internal details and showing only the essential features."

---

## The Real-World Analogy

Think of a **smartphone**:

- You interact with the **screen, buttons, and apps** (public interface)
- You have **no access** to the internal circuitry, battery chemistry, or OS kernel (private implementation)
- The manufacturer can **upgrade the battery** without you needing to change how you use the phone

**Your phone "encapsulates" its complexity behind a simple, stable interface.**

---

## Why Encapsulation Matters

| Benefit | Explanation |
|---------|-------------|
| **Data Protection** | Prevents accidental or unauthorized modification of data |
| **Controlled Access** | You decide HOW data can be changed (validation, logic) |
| **Flexibility** | Internal implementation can change without breaking external code |
| **Debugging** | Bugs are isolated to specific classes |
| **Security** | Sensitive data stays hidden from external manipulation |

---

## How Encapsulation Works in C#

### Access Modifiers

| Modifier | Access Level |
|----------|--------------|
| `public` | Accessible from anywhere |
| `private` | Accessible only within the same class |
| `protected` | Accessible within class and derived classes |
| `internal` | Accessible within the same assembly |
| `protected internal` | Accessible within assembly OR derived classes |
| `private protected` | Accessible within class AND derived classes (same assembly) |

### Example: Before and After Encapsulation

#### ❌ BAD: No Encapsulation

```csharp
public class BankAccount
{
    public decimal Balance;  // Anyone can modify directly!
}

// Usage - DANGEROUS!
var account = new BankAccount();
account.Balance = -999999;  // No validation! Account in invalid state
```

**Problems:**
- No validation on what values can be set
- External code can corrupt the object's state
- No way to track changes or add business logic
- Impossible to maintain invariants (rules that must always be true)

---

#### ✅ GOOD: With Encapsulation

```csharp
public class BankAccount
{
    // Private field - hidden from outside
    private decimal _balance;
    
    // Public property with controlled access
    public decimal Balance 
    { 
        get { return _balance; }  // Anyone can read
        private set { _balance = value; }  // Only this class can modify
    }

    // Controlled way to modify balance
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Deposit must be positive");
            
        _balance += amount;
        LogTransaction($"Deposited: {amount:C}");
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Withdrawal must be positive");
        if (amount > _balance)
            throw new InvalidOperationException("Insufficient funds");
            
        _balance -= amount;
        LogTransaction($"Withdrew: {amount:C}");
    }

    private void LogTransaction(string message)
    {
        // Internal logic - completely hidden
        Console.WriteLine($"[{DateTime.Now}] {message}");
    }
}

// Usage - SAFE!
var account = new BankAccount();
account.Deposit(1000);     // ✓ Validated, logged
account.Withdraw(500);     // ✓ Validated, logged
// account.Balance = -999; // ✗ COMPILE ERROR - Cannot set directly!
```

---

## Memory Allocation Visualization

### Memory Layout for Encapsulated Object

```
┌─────────────────────────────────────────────────────────────┐
│                    STACK (Method Call)                       │
├─────────────────────────────────────────────────────────────┤
│  Variable        │  Value                                     │
│  ────────────────┼─────────────────────────────────────────  │
│  account         │  0x7FFE1234  (reference to heap object)      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    HEAP (Object Storage)                     │
├─────────────────────────────────────────────────────────────┤
│  Address         │  Content                                   │
│  ────────────────┼─────────────────────────────────────────  │
│  0x7FFE1234      │  [Object Header]                           │
│  0x7FFE1238      │  _balance: 500.00m  (private field)         │
│  0x7FFE1240      │  [Method Table Pointer]                    │
└─────────────────────────────────────────────────────────────┘
```

**Key Insight:**
- `_balance` is stored in heap memory as part of the object
- No external code can access `0x7FFE1238` directly because it's marked `private`
- Access is ONLY possible through the `Balance` property or `Deposit`/`Withdraw` methods
- The CLR enforces these boundaries at compile time and runtime

---

## Advanced Encapsulation Patterns

### Pattern 1: Immutable Objects

Objects that cannot be changed after creation (thread-safe by design):

```csharp
public sealed class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(currency)) throw new ArgumentException("Currency required");
        
        Amount = amount;
        Currency = currency.Trim().ToUpper();
    }

    // Returns NEW object instead of modifying current
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
            
        return new Money(Amount + other.Amount, Currency);
    }
}

// Usage
var salary = new Money(5000, "USD");
var bonus = new Money(500, "USD");
var total = salary.Add(bonus);  // New object created, salary unchanged
```

**Memory Flow:**
```
salary → [5000, "USD"]  (immutable)
bonus  → [500, "USD"]   (immutable)
total  → [5500, "USD"]  (new object, salary and bonus untouched)
```

---

### Pattern 2: Information Hiding with Calculated Properties

```csharp
public class Employee
{
    // Private backing fields
    private decimal _hourlyRate;
    private int _hoursWorked;
    
    // Public read-only calculated property
    public decimal WeeklySalary => _hourlyRate * _hoursWorked;
    
    // Controlled modification
    public void RecordHours(int hours)
    {
        if (hours < 0 || hours > 168)
            throw new ArgumentException("Invalid hours (0-168)");
        _hoursWorked += hours;
    }

    public void SetRate(decimal rate)
    {
        if (rate < 15) throw new ArgumentException("Minimum wage is $15/hr");
        _hourlyRate = rate;
    }
}
```

---

### Pattern 3: Encapsulation in Domain-Driven Design

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();
    private OrderStatus _status = OrderStatus.Pending;
    
    // Read-only public view of items
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    public OrderStatus Status => _status;
    public decimal Total => _items.Sum(i => i.Subtotal);

    // Domain behavior methods - only way to modify state
    public void AddItem(Product product, int quantity)
    {
        if (_status != OrderStatus.Pending)
            throw new InvalidOperationException("Can only modify pending orders");
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive");
        
        _items.Add(new OrderItem(product, quantity));
    }

    public void Confirm()
    {
        if (!_items.Any())
            throw new InvalidOperationException("Cannot confirm empty order");
            
        _status = OrderStatus.Confirmed;
    }

    public void Cancel()
    {
        if (_status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel shipped order");
            
        _status = OrderStatus.Cancelled;
    }
}
```

---

## Encapsulation vs Abstraction

| Aspect | Encapsulation | Abstraction |
|--------|---------------|-------------|
| **Focus** | Hiding data and implementation details | Showing essential features, hiding complexity |
| **Goal** | Data protection and controlled access | Simplified interface for complex systems |
| **How** | Access modifiers (private, public) | Abstract classes, interfaces |
| **Example** | Private fields with public getters/setters | `Car` abstracting engine, transmission, etc. |

**Relationship:** Encapsulation is HOW you hide; Abstraction is WHAT you show.

---

## Common Mistakes

### ❌ Breaking Encapsulation

```csharp
// DON'T DO THIS
public class Customer
{
    public List<Order> Orders { get; set; }  // Exposes internal list!
}

// External code can now bypass business logic:
customer.Orders.Clear();  // Bypassed any validation!
customer.Orders.Add(order);  // Direct manipulation
```

### ✅ Proper Encapsulation

```csharp
public class Customer
{
    private readonly List<Order> _orders = new();
    
    // Read-only view
    public IReadOnlyCollection<Order> Orders => _orders.AsReadOnly();
    
    // Controlled modification
    public void PlaceOrder(Order order)
    {
        if (!order.IsValid) throw new InvalidOperationException("Invalid order");
        if (_orders.Any(o => o.Id == order.Id)) throw new DuplicateOrderException();
        
        _orders.Add(order);
        NotifyOrderPlaced(order);
    }
}
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    ENCAPSULATION PRINCIPLE                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────┐            │
│   │          EXTERNAL CODE                      │            │
│   │                                             │            │
│   │   account.Deposit(100);     ✓ ALLOWED       │            │
│   │   account.Withdraw(50);     ✓ ALLOWED       │            │
│   │   account.Balance = -999;   ✗ DENIED        │            │
│   └─────────────────┬───────────────────────────┘            │
│                     │                                        │
│           ┌─────────▼──────────┐                            │
│           │   PUBLIC INTERFACE  │                            │
│           │   • Properties      │                            │
│           │   • Methods         │                            │
│           └─────────┬──────────┘                            │
│                     │                                        │
│           ┌─────────▼──────────┐                            │
│           │  PRIVATE FIELDS &   │                            │
│           │  IMPLEMENTATION     │                            │
│           │   • _balance        │                            │
│           │   • _transactions   │                            │
│           │   • Log()           │                            │
│           └─────────────────────┘                            │
│                                                              │
│   Rule: External code can only interact through the public   │
│   interface. Implementation details are hidden and protected.  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: Why use properties instead of public fields?**
> Properties allow you to change the implementation later (add validation, caching, lazy loading) without breaking external code.

**Q: What's the difference between encapsulation and data hiding?**
> Data hiding is the technique (using private fields). Encapsulation is the broader principle of bundling data with behavior and controlling access.

**Q: When should you use private protected vs protected?**
> `private protected` restricts access to derived classes in the same assembly, providing tighter encapsulation in large codebases.

**Q: How does encapsulation support the Single Responsibility Principle?**
> By hiding internal state, a class controls WHO can modify it and HOW, preventing other classes from making it responsible for their concerns.
