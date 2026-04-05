# Abstraction in C#

## What is Abstraction?

Abstraction is the **process of hiding implementation details and showing only the essential features** of an object. While encapsulation hides data, abstraction hides **complexity** — allowing users to interact with simple interfaces without needing to understand the underlying complexity.

> **In simple terms:** Abstraction means "you don't need to know HOW it works, only WHAT it does."

---

## The Real-World Analogy

Think of a **car steering wheel**:

- You only need to know: **turn left, turn right, honk**
- You DON'T need to understand: rack-and-pinion gearing, hydraulic systems, power steering pumps, CAN bus protocols
- The car manufacturer can **upgrade the steering system** (electric power steering replacing hydraulic) without you needing to relearn how to drive

**The steering wheel "abstracts" away the complexity of the steering mechanism.**

---

## Abstraction vs Encapsulation

| Aspect | Abstraction | Encapsulation |
|--------|-------------|---------------|
| **Focus** | Hiding complexity and showing essentials | Hiding data and controlling access |
| **Level** | Design-level concept | Implementation-level concept |
| **Goal** | Simplify usage | Protect data |
| **How** | Abstract classes, interfaces | Access modifiers (private, public) |
| **Analogy** | Car dashboard (simple controls) | Car hood (hides engine) |

**They work together:** Encapsulation wraps data; Abstraction wraps complexity.

---

## How Abstraction Works in C#

C# provides abstraction through:
1. **Abstract Classes** - Partial implementation, cannot be instantiated
2. **Interfaces** - Pure contract, no implementation

---

## Abstract Classes

An abstract class provides a **base blueprint** with some common implementation that derived classes must complete.

```csharp
// Abstract base class - cannot be instantiated directly
public abstract class PaymentProcessor
{
    // Common field - shared by all payment processors
    protected readonly ILogger _logger;
    
    public PaymentProcessor(ILogger logger)
    {
        _logger = logger;
    }

    // Concrete method - shared implementation
    public PaymentResult ProcessPayment(decimal amount, string currency)
    {
        _logger.Log($"Processing {currency} {amount}");
        
        try
        {
            // Call the abstract method (implemented by child)
            var transactionId = ExecutePayment(amount, currency);
            
            _logger.Log($"Payment successful: {transactionId}");
            return PaymentResult.Success(transactionId);
        }
        catch (Exception ex)
        {
            _logger.LogError($"Payment failed: {ex.Message}");
            return PaymentResult.Failure(ex.Message);
        }
    }

    // Abstract method - MUST be implemented by child classes
    protected abstract string ExecutePayment(decimal amount, string currency);
    
    // Virtual method - CAN be overridden by child classes
    protected virtual void ValidateCurrency(string currency)
    {
        if (string.IsNullOrWhiteSpace(currency))
            throw new ArgumentException("Currency required");
    }
}

// Concrete implementations
public class StripeProcessor : PaymentProcessor
{
    private readonly string _apiKey;
    
    public StripeProcessor(string apiKey, ILogger logger) : base(logger)
    {
        _apiKey = apiKey;
    }

    // Must implement abstract method
    protected override string ExecutePayment(decimal amount, string currency)
    {
        // Stripe-specific implementation
        var stripe = new StripeClient(_apiKey);
        var charge = stripe.Charges.Create(new ChargeCreateOptions
        {
            Amount = (long)(amount * 100), // cents
            Currency = currency.ToLower(),
        });
        return charge.Id;
    }
}

public class PayPalProcessor : PaymentProcessor
{
    private readonly string _clientId;
    private readonly string _clientSecret;
    
    public PayPalProcessor(string clientId, string clientSecret, ILogger logger) 
        : base(logger)
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

// Usage - ABSTRACTION IN ACTION
public class CheckoutService
{
    private readonly PaymentProcessor _paymentProcessor;
    
    public CheckoutService(PaymentProcessor paymentProcessor)
    {
        _paymentProcessor = paymentProcessor; // Could be Stripe OR PayPal
    }

    public void CompleteOrder(Order order)
    {
        // Code doesn't care WHICH processor it is!
        // Just knows it CAN process payments
        var result = _paymentProcessor.ProcessPayment(
            order.Total, 
            order.Currency);
            
        if (result.IsSuccess)
        {
            order.MarkAsPaid(result.TransactionId);
        }
    }
}
```

---

## Memory Allocation for Abstract Classes

```
┌─────────────────────────────────────────────────────────────────┐
│                    HEAP MEMORY LAYOUT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  PaymentProcessor (Abstract) - Method Table              │    │
│  │  ┌─────────────────────────────────────────────────────┐│    │
│  │  │ Virtual Method Table (vtable)                       ││    │
│  │  │ ├─ ProcessPayment() → Concrete implementation       ││    │
│  │  │ ├─ ExecutePayment() → ABSTRACT - no address         ││    │
│  │  │ └─ ValidateCurrency() → Default implementation      ││    │
│  │  └─────────────────────────────────────────────────────┘│    │
│  └─────────────────────────────────────────────────────────┘    │
│                              ▲                                   │
│                              │ Inherits vtable layout            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  StripeProcessor - Method Table                          │    │
│  │  ┌─────────────────────────────────────────────────────┐│    │
│  │  │ Virtual Method Table                                ││    │
│  │  │ ├─ ProcessPayment() → Inherited from base          ││    │
│  │  │ ├─ ExecutePayment() → Stripe's implementation ◄──  ││    │
│  │  │ └─ ValidateCurrency() → Inherited from base          ││    │
│  │  └─────────────────────────────────────────────────────┘│    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  StripeProcessor Instance @ 0x7FFE1000                   │    │
│  │  ├─ [Object Header]                                       │    │
│  │  ├─ [Method Table Pointer] → StripeProcessor vtable      │    │
│  │  ├─ _logger: 0x7FFE2000                                  │    │
│  │  ├─ _apiKey: "sk_live_..."                                │    │
│  │  └─ [Other fields]                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Key Points:                                                     │
│  • Abstract class has vtable entries for ALL methods              │
│  • Abstract methods have NULL entries until overridden            │
│  • Cannot instantiate PaymentProcessor directly (no vtable entry) │
│  • Concrete classes fill in ALL abstract method slots             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interfaces

An interface defines a **pure contract** with no implementation. Classes implement the interface and provide ALL the implementation.

```csharp
// Interface - pure contract
public interface INotificationService
{
    string Channel { get; }  // Property contract
    void Send(string recipient, string message);  // Method contract
}

// Multiple implementations
public class EmailService : INotificationService
{
    public string Channel => "Email";
    
    private readonly SmtpClient _smtpClient;
    
    public EmailService(SmtpClient smtpClient)
    {
        _smtpClient = smtpClient;
    }

    public void Send(string recipient, string message)
    {
        var mail = new MailMessage("noreply@company.com", recipient)
        {
            Subject = "Notification",
            Body = message
        };
        _smtpClient.Send(mail);
    }
}

public class SmsService : INotificationService
{
    public string Channel => "SMS";
    
    private readonly ITwilioClient _twilio;
    
    public SmsService(ITwilioClient twilio)
    {
        _twilio = twilio;
    }

    public void Send(string recipient, string message)
    {
        _twilio.SendMessage(
            from: "+1234567890",
            to: recipient,
            body: message
        );
    }
}

public class PushNotificationService : INotificationService
{
    public string Channel => "Push";
    
    private readonly IFirebaseMessaging _firebase;
    
    public void Send(string recipient, string message)
    {
        _firebase.SendAsync(new Message
        {
            Token = recipient, // Device token
            Notification = new Notification { Body = message }
        });
    }
}

// High-level code depends on ABSTRACTION (interface), not concrete classes
public class OrderConfirmationService
{
    private readonly List<INotificationService> _notificationServices;
    
    public OrderConfirmationService(IEnumerable<INotificationService> services)
    {
        _notificationServices = services.ToList();
    }

    public void ConfirmOrder(Order order)
    {
        var message = $"Order #{order.Id} confirmed. Total: {order.Total:C}";
        
        // Same code works for Email, SMS, or Push!
        // Doesn't care HOW each service sends - just THAT it can send
        foreach (var service in _notificationServices)
        {
            service.Send(order.CustomerContact, message);
        }
    }
}

// Composition - combine multiple notification methods
var notificationServices = new List<INotificationService>
{
    new EmailService(smtpClient),
    new SmsService(twilioClient),
    new PushNotificationService(firebase)
};

var confirmationService = new OrderConfirmationService(notificationServices);
```

---

## Memory Allocation for Interfaces

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERFACE DISPATCH                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  HEAP:                                                           │
│  ┌────────────────────────┐                                       │
│  │ EmailService Instance  │                                       │
│  │ ├─ [Header]            │                                       │
│  │ ├─ [MT Ptr] ───────────┼──┐                                    │
│  │ ├─ _smtpClient         │  │                                    │
│  │ └─ ...                 │  │                                    │
│  └────────────────────────┘  │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │  EmailService Method Table                              │      │
│  │  ├─ [Interface Map]                                     │      │
│  │  │   └─ INotificationService                             │      │
│  │  │       ├─ Channel → offset 0x08                      │      │
│  │  │       └─ Send → offset 0x10                          │      │
│  │  └─ [Class Methods]                                     │      │
│  └─────────────────────────────────────────────────────────┘      │
│                                                                  │
│  COMPILED IL CODE:                                               │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │  callvirt instance void INotificationService::Send()      │      │
│  └─────────────────────────────────────────────────────────┘      │
│                                                                  │
│  RUNTIME DISPATCH:                                               │
│  1. Load object reference                                        │
│  2. Follow MT Ptr to Method Table                              │
│  3. Look up interface map for INotificationService             │
│  4. Find method offset in interface map                         │
│  5. Jump to actual implementation                               │
│                                                                  │
│  Result: Same IL works for EmailService, SmsService, etc.       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Abstract Class vs Interface

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| **Implementation** | Can have concrete methods | C# 8.0+ can have default implementations |
| **Instantiation** | Cannot instantiate | Cannot instantiate |
| **Multiple inheritance** | Single inheritance only | Multiple interfaces allowed |
| **Fields** | Can have fields | No instance fields (static allowed) |
| **Constructors** | Yes | No |
| **Access modifiers** | Any | Default public, C# 8.0+ allows private |
| **Use when** | "IS-A" relationship with shared code | "CAN-DO" capability contract |

---

## Practical Example: Repository Pattern

```csharp
// Repository interface - abstraction over data access
public interface IRepository<T> where T : class
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(int id);
    void SaveChanges();
}

// SQL Server implementation
public class SqlRepository<T> : IRepository<T> where T : class
{
    private readonly DbContext _context;
    private readonly DbSet<T> _dbSet;

    public SqlRepository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public T GetById(int id) => _dbSet.Find(id);
    public IEnumerable<T> GetAll() => _dbSet.ToList();
    public void Add(T entity) => _dbSet.Add(entity);
    public void Update(T entity) => _dbSet.Update(entity);
    public void Delete(int id)
    {
        var entity = _dbSet.Find(id);
        if (entity != null) _dbSet.Remove(entity);
    }
    public void SaveChanges() => _context.SaveChanges();
}

// In-Memory implementation for testing
public class InMemoryRepository<T> : IRepository<T> where T : class
{
    private readonly Dictionary<int, T> _data = new();
    private int _nextId = 1;

    public T GetById(int id) => _data.TryGetValue(id, out var entity) ? entity : null;
    public IEnumerable<T> GetAll() => _data.Values;
    public void Add(T entity)
    {
        var id = _nextId++;
        typeof(T).GetProperty("Id")?.SetValue(entity, id);
        _data[id] = entity;
    }
    public void Update(T entity)
    {
        var id = (int)typeof(T).GetProperty("Id").GetValue(entity);
        _data[id] = entity;
    }
    public void Delete(int id) => _data.Remove(id);
    public void SaveChanges() { /* No-op for in-memory */ }
}

// Business logic depends only on abstraction
public class CustomerService
{
    private readonly IRepository<Customer> _customerRepository;
    
    public CustomerService(IRepository<Customer> customerRepository)
    {
        _customerRepository = customerRepository;
    }

    public Customer GetCustomer(int id) => _customerRepository.GetById(id);
    public void CreateCustomer(Customer customer)
    {
        _customerRepository.Add(customer);
        _customerRepository.SaveChanges();
    }
}

// Dependency Injection configuration
// Production:
services.AddScoped<IRepository<Customer>, SqlRepository<Customer>>();

// Testing:
services.AddScoped<IRepository<Customer>, InMemoryRepository<Customer>>();
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    ABSTRACTION PRINCIPLE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────────┐                       │
│                    │   USER/CALLER       │                       │
│                    │   ─────────────     │                       │
│                    │   "Process Payment" │                       │
│                    │   "Send Notification"│                      │
│                    └──────────┬──────────┘                       │
│                               │                                  │
│                    ┌──────────▼──────────┐                       │
│                    │   ABSTRACT LAYER    │                       │
│                    │   ───────────────── │                       │
│                    │   • Abstract Class   │                       │
│                    │   • Interface        │                       │
│                    │   ─────────────────  │                       │
│                    │   Hides:             │                       │
│                    │   • Which processor  │                       │
│                    │   • How it works     │                       │
│                    │   • Implementation   │                       │
│                    └──────────┬──────────┘                       │
│                               │                                  │
│          ┌────────────────────┼────────────────────┐             │
│          │                    │                    │             │
│  ┌───────▼──────┐   ┌────────▼────────┐   ┌───────▼──────┐       │
│  │   Stripe     │   │    PayPal      │   │   Braintree  │       │
│  │   Processor  │   │    Processor   │   │   Processor  │       │
│  └──────────────┘   └─────────────────┘   └──────────────┘       │
│                                                                  │
│  Benefit: Add new payment processor WITHOUT changing caller code │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: What's the difference between abstraction and encapsulation?**
> Encapsulation hides data (private fields); Abstraction hides implementation (showing only interface).

**Q: When should you use an abstract class vs interface?**
> Use abstract class for "IS-A" with shared code; use interface for "CAN-DO" capabilities, especially when multiple inheritance is needed.

**Q: Can an abstract class have a constructor?**
> Yes, and it's called when concrete subclasses are instantiated to initialize base fields.

**Q: What's the purpose of virtual vs abstract methods?**
> Virtual methods have default implementations that CAN be overridden; abstract methods MUST be implemented by subclasses.

**Q: How does abstraction support the Open/Closed Principle?**
> Code is open for extension (new implementations) but closed for modification (caller code doesn't change).
