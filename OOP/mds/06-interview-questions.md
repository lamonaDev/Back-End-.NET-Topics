# OOP Interview Questions for C#

## Conceptual Questions

### Q1: What are the four pillars of OOP?

**Answer:**

| Pillar | Definition | Purpose |
|--------|------------|---------|
| **Encapsulation** | Bundling data and methods that operate on that data into a single unit, while restricting access to some components | Data protection, controlled access |
| **Abstraction** | Hiding implementation details and showing only essential features | Simplified usage, reduced complexity |
| **Inheritance** | Acquiring properties and behaviors from a parent class | Code reuse, IS-A relationships |
| **Polymorphism** | Objects taking many forms; same call, different behaviors | Flexibility, extensibility |

---

### Q2: Explain the difference between abstraction and encapsulation.

**Answer:**

| Aspect | Abstraction | Encapsulation |
|--------|-------------|---------------|
| **Focus** | Hiding complexity and showing essentials | Hiding data and controlling access |
| **Level** | Design-level | Implementation-level |
| **Goal** | Simplify usage | Protect data |
| **How** | Abstract classes, interfaces | Access modifiers (private, public) |
| **Analogy** | Car dashboard (simple controls) | Car hood (hides engine) |

**Key Point:** They work together — encapsulation wraps data; abstraction wraps complexity.

---

### Q3: What is the difference between method overloading and method overriding?

**Answer:**

```csharp
// OVERLOADING (Compile-time polymorphism)
public class Calculator
{
    public int Add(int a, int b) => a + b;           // Signature 1
    public int Add(int a, int b, int c) => a + b + c; // Signature 2
    public double Add(double a, double b) => a + b;   // Signature 3
}
// Same name, different parameters
// Resolved at COMPILE time

// OVERRIDING (Runtime polymorphism)
public class Animal
{
    public virtual void MakeSound() => Console.WriteLine("Generic");
}

public class Dog : Animal
{
    public override void MakeSound() => Console.WriteLine("Woof!");
}

public class Cat : Animal
{
    public override void MakeSound() => Console.WriteLine("Meow!");
}

Animal a1 = new Dog();
Animal a2 = new Cat();
a1.MakeSound(); // "Woof!" - resolved at RUNTIME
a2.MakeSound(); // "Meow!" - resolved at RUNTIME
```

| Aspect | Overloading | Overriding |
|--------|-------------|------------|
| Class location | Same class | Base + Derived classes |
| Signature | Must be different | Must be same |
| Return type | Can be different | Must be same (or covariant) |
| Keywords | None | `virtual`, `override` |
| Resolution | Compile time | Runtime |
| Binding | Early | Late |

---

### Q4: Can you override a non-virtual method?

**Answer:** No. You can only **hide** it with the `new` keyword, but that breaks polymorphism:

```csharp
public class Animal
{
    public void MakeSound() => Console.WriteLine("Generic");
}

public class Dog : Animal
{
    public new void MakeSound() => Console.WriteLine("Woof!");
}

// Demonstrating the difference
Animal a = new Dog();
a.MakeSound();  // "Generic" - calls Animal's version!
                // Dog.MakeSound is HIDDEN when accessed through Animal reference

Dog d = new Dog();
d.MakeSound();  // "Woof!" - Dog's version
```

To enable true polymorphism, the base method must be marked `virtual` or `abstract`.

---

### Q5: What is the Liskov Substitution Principle (LSP)?

**Answer:**

> **Definition:** Objects of a superclass shall be replaceable with objects of its subclasses without breaking the application.

**Example of LSP Violation:**

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public virtual int Area => Width * Height;
}

public class Square : Rectangle  // Square IS-A Rectangle?
{
    // Violates LSP!
    public override int Width
    {
        set { base.Width = value; base.Height = value; }
    }
    public override int Height
    {
        set { base.Width = value; base.Height = value; }
    }
}

// This code breaks with Square:
void TestRectangle(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    Debug.Assert(rect.Area == 50);  // FAILS for Square (returns 100)!
}
```

**Solution:** Use composition instead:

```csharp
public interface IShape { int Area { get; } }
public class Rectangle : IShape { /* ... */ }
public class Square : IShape { /* ... */ }  // No inheritance
```

---

### Q6: Explain the four types of class relationships.

**Answer:**

| Relationship | Strength | UML | Lifetime | Example |
|--------------|----------|-----|----------|---------|
| **Dependency** | Weakest | `---▷` (dashed) | Method scope | Service uses Repository as parameter |
| **Association** | Weak | `───▷` (solid) | Independent | Order has Customer reference |
| **Aggregation** | Medium | `◇───` (white diamond) | Child outlives parent | Team has Players |
| **Composition** | Strong | `◆───` (black diamond) | Child dies with parent | House has Rooms |

---

### Q7: What is the difference between aggregation and composition?

**Answer:**

| Aspect | Aggregation | Composition |
|--------|-------------|-------------|
| **Ownership** | Weak | Strong |
| **Lifetime** | Child outlives parent | Child dies with parent |
| **Creation** | Child created outside | Child created inside |
| **Sharing** | Child can be shared | Child belongs exclusively to one parent |
| **UML** | White diamond (◇) | Black diamond (◆) |
| **Example** | University has Students | House has Rooms |

---

### Q8: Why doesn't C# support multiple inheritance?

**Answer:**

To avoid the **"Diamond Problem"** — ambiguity when a class inherits from two classes that share a common base:

```
     BaseClass
       /    \
  ClassA   ClassB
       \    /
      ClassC  ← Which method implementation if both override?
```

**C# Solution:** Interfaces provide similar capability without this issue:

```csharp
// Multiple interfaces allowed
public class MyClass : IComparable, IDisposable, ILogger
{
    // Must implement all interface members
}
```

---

## Code-Based Questions

### Q9: What is the output of this code?

```csharp
public class Base
{
    public virtual void Print() => Console.WriteLine("Base");
}

public class Derived : Base
{
    public override void Print() => Console.WriteLine("Derived");
}

Base b = new Derived();
b.Print();
```

**Answer:** "Derived"

**Explanation:** Even though `b` is declared as `Base`, it actually references a `Derived` object. Since `Print()` is virtual, the runtime uses the virtual method table (vtable) to dispatch to `Derived.Print()`.

---

### Q10: What is the output of this code?

```csharp
public class Base
{
    public void Print() => Console.WriteLine("Base");
}

public class Derived : Base
{
    public new void Print() => Console.WriteLine("Derived");
}

Base b = new Derived();
b.Print();
```

**Answer:** "Base"

**Explanation:** Without `virtual`/`override`, polymorphism doesn't work. The `new` keyword creates a completely separate method. Since `b` is typed as `Base`, it calls `Base.Print()`. The `Derived.Print()` is hidden.

---

### Q11: What happens when you mark a class as `sealed`?

**Answer:**

```csharp
public sealed class FinalClass { }
// class Derived : FinalClass { }  // ❌ COMPILE ERROR

public class Base
{
    public virtual void Method() { }
}

public class Derived : Base
{
    public sealed override void Method() { }
    // No further override possible in classes that inherit from Derived
}
```

**Use Cases:**
- Security: Prevent malicious subclassing
- Performance: Enable optimizations (virtual dispatch not needed)
- Design: Explicitly mark classes not intended for inheritance

---

### Q12: Write a class that properly encapsulates its data.

**Answer:**

```csharp
public class BankAccount
{
    // Private fields - hidden from outside
    private decimal _balance;
    private readonly List<string> _transactionHistory = new();

    // Public read-only calculated property
    public decimal Balance => _balance;
    
    // Public read-only view of history
    public IReadOnlyCollection<string> TransactionHistory => _transactionHistory.AsReadOnly();

    // Controlled modification through methods
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive", nameof(amount));
        
        _balance += amount;
        _transactionHistory.Add($"Deposited: {amount:C} at {DateTime.Now}");
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive", nameof(amount));
        if (amount > _balance)
            throw new InvalidOperationException("Insufficient funds");
        
        _balance -= amount;
        _transactionHistory.Add($"Withdrew: {amount:C} at {DateTime.Now}");
    }
}
```

---

### Q13: Design a payment system using polymorphism.

**Answer:**

```csharp
// Abstract base
public abstract class PaymentProcessor
{
    protected readonly ILogger _logger;
    
    public PaymentProcessor(ILogger logger) => _logger = logger;
    
    // Template method pattern
    public PaymentResult Process(decimal amount, string currency)
    {
        try
        {
            Validate(amount, currency);
            var transactionId = ExecutePayment(amount, currency);
            LogSuccess(transactionId);
            return PaymentResult.Success(transactionId);
        }
        catch (Exception ex)
        {
            LogFailure(ex);
            return PaymentResult.Failure(ex.Message);
        }
    }
    
    protected abstract string ExecutePayment(decimal amount, string currency);
    
    protected virtual void Validate(decimal amount, string currency)
    {
        if (amount <= 0) throw new ArgumentException("Invalid amount");
        if (string.IsNullOrEmpty(currency)) throw new ArgumentException("Currency required");
    }
    
    private void LogSuccess(string id) => _logger.Log($"Payment {id} succeeded");
    private void LogFailure(Exception ex) => _logger.LogError($"Payment failed: {ex.Message}");
}

// Concrete implementations
public class StripeProcessor : PaymentProcessor
{
    private readonly string _apiKey;
    
    public StripeProcessor(string apiKey, ILogger logger) : base(logger) => _apiKey = apiKey;
    
    protected override string ExecutePayment(decimal amount, string currency)
    {
        // Stripe-specific implementation
        var stripe = new StripeClient(_apiKey);
        var charge = stripe.Charges.Create(new ChargeCreateOptions
        {
            Amount = (long)(amount * 100),
            Currency = currency.ToLower(),
        });
        return charge.Id;
    }
}

public class PayPalProcessor : PaymentProcessor
{
    private readonly string _clientId;
    private readonly string _clientSecret;
    
    public PayPalProcessor(string clientId, string clientSecret, ILogger logger) : base(logger)
    {
        _clientId = clientId;
        _clientSecret = clientSecret;
    }
    
    protected override string ExecutePayment(decimal amount, string currency)
    {
        // PayPal-specific implementation
        var apiContext = new APIContext(new OAuthTokenCredential(
            _clientId, _clientSecret).GetAccessToken());
        
        var payment = Payment.Create(apiContext, new Payment
        {
            intent = "sale",
            transactions = new List<Transaction>
            {
                new Transaction
                {
                    amount = new Amount { currency = currency, total = amount.ToString() }
                }
            }
        });
        return payment.id;
    }
}

// Usage - polymorphism in action
public class CheckoutService
{
    private readonly PaymentProcessor _processor;
    
    public CheckoutService(PaymentProcessor processor) => _processor = processor;
    
    public void CompleteOrder(Order order)
    {
        var result = _processor.Process(order.Total, order.Currency);
        if (result.IsSuccess)
            order.MarkAsPaid(result.TransactionId);
    }
}
```

---

## Memory Allocation Questions

### Q14: How does virtual method dispatch work at the memory level?

**Answer:**

```
1. Each class has a Virtual Method Table (vtable) in memory
2. Each object has a hidden pointer to its class's vtable
3. When calling a virtual method:
   - CLR reads the object header to get vtable pointer
   - Looks up the method slot in the vtable
   - Jumps to that address

EXAMPLE:
┌─────────────────────────────────────────────────────────┐
│ Dog vtable                                              │
│ ├─ Slot 0: MakeSound → Dog.MakeSound (0x7FFE5000)      │
│ └─ Slot 1: Eat → Animal.Eat (inherited)               │
└─────────────────────────────────────────────────────────┘
                            │
                            ▼
Animal a = new Dog();   // Object at 0x7FFE1000
a.MakeSound();          // Runtime dispatch:
                         // 1. Read 0x7FFE1000 → get vtable ptr (Dog vtable)
                         // 2. Slot 0 → 0x7FFE5000
                         // 3. Jump to Dog.MakeSound
```

---

### Q15: Explain the difference between `call` and `callvirt` IL instructions.

**Answer:**

| Instruction | Usage | Performance |
|-------------|-------|-------------|
| `call` | Non-virtual methods, static methods | Fast (direct address) |
| `callvirt` | Virtual methods, instance methods | Slower (vtable lookup) |

```csharp
public class Example
{
    public void Normal() { }      // call
    public virtual void Virtual() { }  // callvirt
}

// IL for nonVirtualObj.Normal():
// call instance void Example::Normal()

// IL for virtualObj.Virtual():
// callvirt instance void Example::Virtual()
```

**Note:** Even for non-virtual instance methods, C# uses `callvirt` to enforce null checking.

---

## Best Practices Questions

### Q16: When should you prefer composition over inheritance?

**Answer:**

Prefer composition when:
1. **Relationship is HAS-A, not IS-A**
2. **Want to avoid fragile base class problem**
3. **Need flexibility at runtime** (can swap components)
4. **Class needs multiple capabilities** (no multiple inheritance in C#)

```csharp
// INHERITANCE (brittle)
public class LoggerStack : Stack<string>  // Stack IS-NOT-A Logger!
{
    // Inherits all Stack methods - dangerous!
}

// COMPOSITION (flexible)
public class Logger
{
    private readonly ILogSink _sink;  // Can be Console, File, Database, etc.
    
    public Logger(ILogSink sink) => _sink = sink;
    
    public void Log(string message) => _sink.Write(message);
}
```

---

### Q17: What is the Open/Closed Principle and how does OOP support it?

**Answer:**

> **Open for extension, closed for modification**

**OOP Support through Polymorphism:**

```csharp
// Existing code - DOESN'T CHANGE
public class NotificationService
{
    private readonly List<INotificationChannel> _channels;
    
    public NotificationService(IEnumerable<INotificationChannel> channels)
    {
        _channels = channels.ToList();
    }
    
    public void Notify(string message)
    {
        foreach (var channel in _channels)
            channel.Send(message);
    }
}

// EXTENSION: Add new channel WITHOUT modifying NotificationService
public class SlackChannel : INotificationChannel
{
    public void Send(string message) { /* Slack implementation */ }
}

// Just inject new channel:
var service = new NotificationService(new[]
{
    new EmailChannel(),
    new SmsChannel(),
    new SlackChannel()  // NEW!
});
```

---

### Q18: What are the SOLID principles?

**Answer:**

| Principle | Definition | OOP Mechanism |
|-----------|------------|---------------|
| **S**ingle Responsibility | A class should have one reason to change | Class design |
| **O**pen/Closed | Open for extension, closed for modification | Polymorphism, Interfaces |
| **L**iskov Substitution | Subtypes must be substitutable | Inheritance rules |
| **I**nterface Segregation | Clients shouldn't depend on methods they don't use | Interface design |
| **D**ependency Inversion | Depend on abstractions, not concretions | Interfaces, DI |

---

## Scenario-Based Questions

### Q19: Design a document management system with versioning.

**Answer:**

```csharp
// Core abstraction
public interface IDocumentStorage
{
    void Save(Document document);
    Document Load(Guid id);
    IEnumerable<Document> GetVersions(Guid documentId);
}

// Composition over inheritance
public class Document
{
    public Guid Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public int Version { get; set; }
    public DateTime CreatedAt { get; set; }
    public Guid? PreviousVersionId { get; set; }
}

public class DocumentManager
{
    private readonly IDocumentStorage _storage;
    private readonly IVersioningStrategy _versioning;
    
    public DocumentManager(IDocumentStorage storage, IVersioningStrategy versioning)
    {
        _storage = storage;
        _versioning = versioning;
    }
    
    public void SaveNewVersion(Document document)
    {
        var existing = _storage.Load(document.Id);
        if (existing != null)
        {
            document.Version = existing.Version + 1;
            document.PreviousVersionId = existing.Id;
        }
        
        _storage.Save(document);
    }
    
    public Document GetVersion(Guid documentId, int version)
    {
        return _storage.GetVersions(documentId)
            .FirstOrDefault(v => v.Version == version);
    }
}
```

---

### Q20: How would you refactor a class with too many responsibilities?

**Answer:**

```csharp
// BEFORE: God class with multiple responsibilities
public class OrderService
{
    public void CreateOrder() { /* ... */ }
    public void ProcessPayment() { /* ... */ }
    public void SendEmail() { /* ... */ }
    public void UpdateInventory() { /* ... */ }
    public void GenerateInvoice() { /* ... */ }
}

// AFTER: Separated by responsibility
public class OrderService
{
    private readonly IPaymentService _payment;
    private readonly INotificationService _notification;
    private readonly IInventoryService _inventory;
    private readonly IInvoiceService _invoice;
    
    public OrderService(
        IPaymentService payment,
        INotificationService notification,
        IInventoryService inventory,
        IInvoiceService invoice)
    {
        _payment = payment;
        _notification = notification;
        _inventory = inventory;
        _invoice = invoice;
    }
    
    public void CreateOrder(OrderRequest request)
    {
        var order = CreateOrderEntity(request);
        _payment.Process(order);
        _inventory.Reserve(order.Items);
        _invoice.Generate(order);
        _notification.SendOrderConfirmation(order);
    }
}
```

---

## Summary Table

| Question Category | Key Focus |
|-------------------|-----------|
| **Conceptual** | Understanding definitions, differences, principles |
| **Code-based** | Syntax, behavior, output prediction |
| **Memory** | vtable, dispatch, IL instructions |
| **Best Practices** | SOLID, patterns, refactoring |
| **Scenarios** | Design, real-world applications |

*Practice explaining these concepts with code examples and analogies for interview success.*
