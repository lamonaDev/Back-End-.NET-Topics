# Mocking with Moq

## Table of Contents
1. [Understanding Mocking](#understanding-mocking)
2. [Moq Fundamentals](#moq-fundamentals)
3. [Setting Up Mock Behavior](#setting-up-mock-behavior)
4. [Verifying Interactions](#verifying-interactions)
5. [Advanced Mocking Patterns](#advanced-mocking-patterns)
6. [Mock vs Stub vs Fake](#mock-vs-stub-vs-fake)
7. [Real-World Patterns](#real-world-patterns)
8. [Best Practices](#best-practices)

---

## Understanding Mocking

### What is Mocking?

Mocking is the practice of creating **test doubles** that simulate the behavior of real dependencies. Mocks allow you to test a unit in isolation without relying on external systems (databases, APIs, file systems).

### Test Doubles Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                    TEST DOUBLES HIERARCHY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Fake                                                     │    │
│   │  └── Working implementation (in-memory database)        │    │
│   │      Used for integration testing                        │    │
│   └─────────────────────────────────────────────────────────┘    │
│                          │                                       │
│   ┌──────────────────────┴──────────────────────────────┐       │
│   │  Stub                                                │       │
│   │  └── Provides canned answers                        │       │
│   │      "Returns 42 when GetCustomer(1) is called"       │       │
│   │      No verification of behavior                     │       │
│   └──────────────────────────────────────────────────────┘       │
│                          │                                       │
│   ┌──────────────────────┴──────────────────────────────┐       │
│   │  Mock                                              │       │
│   │  └── Stub + Verification                            │       │
│   │      "Verify that SendEmail was called exactly once"│      │
│   │      Tests interactions and behavior                 │       │
│   └──────────────────────────────────────────────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why Mock?

```csharp
// ❌ Without mocking - slow, unreliable, complex
[Fact]
public void CreateOrder_WithRealDatabase()
{
    // Requires database to be running
    // Test data setup is complex
    // Tests are slow
    // May affect real data!
    var service = new OrderService(new RealDatabaseConnection());
    // ...
}

// ✅ With mocking - fast, isolated, reliable
[Fact]
public void CreateOrder_WithMockDatabase()
{
    var mockRepo = new Mock<IOrderRepository>();
    var service = new OrderService(mockRepo.Object);
    // ...
}
```

---

## Moq Fundamentals

### Installation and Setup

```bash
dotnet add package Moq
```

```csharp
using Moq;

public class OrderServiceTests
{
    [Fact]
    public void CreateOrder_SavesToRepository()
    {
        // Arrange
        var mockRepo = new Mock<IOrderRepository>();
        var mockEmail = new Mock<IEmailService>();
        
        var service = new OrderService(
            mockRepo.Object,    // Get the mocked interface
            mockEmail.Object);
        
        var order = new Order { CustomerId = 1, Total = 100 };
        
        // Act
        service.CreateOrder(order);
        
        // Assert
        mockRepo.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
    }
}
```

### Creating Mocks

```csharp
// Mock from interface (preferred)
var mock = new Mock<IOrderRepository>();

// Mock from abstract class
var mock = new Mock<BaseService>();

// With mock behavior (strict vs loose)
var strictMock = new Mock<IOrderRepository>(MockBehavior.Strict);
// Strict: all methods must have setups, throws if unexpected call

var looseMock = new Mock<IOrderRepository>(MockBehavior.Loose);
// Loose: returns default values for unconfigured calls (default)
```

---

## Setting Up Mock Behavior

### Return Values

```csharp
public class SetupExamples
{
    [Fact]
    public void Setup_Returns()
    {
        var mock = new Mock<IOrderRepository>();
        
        // Return fixed value
        mock.Setup(r => r.GetById(1)).Returns(new Order { Id = 1, Total = 100 });
        
        // Return null
        mock.Setup(r => r.GetById(999)).Returns((Order?)null);
        
        // Return different values on consecutive calls
        mock.Setup(r => r.GetById(1))
            .Returns(new Order { Id = 1 })
            .Returns(new Order { Id = 2 })  // Second call
            .Returns(new Order { Id = 3 }); // Third call
        
        // Return computed value
        mock.Setup(r => r.GetById(It.IsAny<int>()))
            .Returns((int id) => new Order { Id = id });
    }
    
    [Fact]
    public void Setup_Async()
    {
        var mock = new Mock<IOrderRepository>();
        
        // Setup async method
        mock.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(new Order { Id = 1 });
        
        // Setup async to throw
        mock.Setup(r => r.GetByIdAsync(999))
            .ThrowsAsync(new NotFoundException());
    }
}
```

### Conditional Setups

```csharp
[Fact]
public void Setup_Conditional()
{
    var mock = new Mock<IOrderRepository>();
    
    // Match any value
    mock.Setup(r => r.GetById(It.IsAny<int>()))
        .Returns(new Order());
    
    // Match specific range
    mock.Setup(r => r.GetById(It.IsInRange(1, 100, Range.Inclusive)))
        .Returns(new Order { Status = OrderStatus.Active });
    
    // Match with predicate
    mock.Setup(r => r.GetById(It.Is<int>(id => id > 0)))
        .Returns(new Order { Status = OrderStatus.Active });
    
    // Match specific string
    mock.Setup(r => r.GetByEmail("test@example.com"))
        .Returns(new Order());
    
    // Match any string
    mock.Setup(r => r.GetByEmail(It.IsAny<string>()))
        .Returns(new Order());
    
    // Match regex
    mock.Setup(r => r.GetByEmail(It.IsRegex(@"^.+@.+\\..+$")))
        .Returns(new Order());
}
```

### Callbacks and Side Effects

```csharp
[Fact]
public void Setup_Callback()
{
    var savedOrders = new List<Order>();
    var mock = new Mock<IOrderRepository>();
    
    // Capture arguments
    mock.Setup(r => r.Save(It.IsAny<Order>()))
        .Callback<Order>(order => savedOrders.Add(order));
    
    // Execute before returning
    mock.Setup(r => r.GetById(1))
        .Callback(() => Console.WriteLine("Getting order..."))
        .Returns(new Order());
}
```

---

## Verifying Interactions

### Basic Verification

```csharp
public class VerificationExamples
{
    [Fact]
    public void Verify_Simple()
    {
        var mock = new Mock<IEmailService>();
        var service = new OrderService(Mock.Of<IOrderRepository>(), mock.Object);
        
        service.CreateOrder(new Order { CustomerId = 1 });
        
        // Verify method was called
        mock.Verify(e => e.SendOrderConfirmation(It.IsAny<Order>()));
        
        // Verify call count
        mock.Verify(e => e.SendOrderConfirmation(It.IsAny<Order>()), Times.Once);
        mock.Verify(e => e.SendOrderConfirmation(It.IsAny<Order>()), Times.Exactly(2));
        mock.Verify(e => e.SendOrderConfirmation(It.IsAny<Order>()), Times.Never);
    }
    
    [Fact]
    public void Verify_Arguments()
    {
        var mock = new Mock<IOrderRepository>();
        var service = new OrderService(mock.Object, Mock.Of<IEmailService>());
        
        service.CreateOrder(new Order { CustomerId = 42, Total = 100 });
        
        // Verify specific argument values
        mock.Verify(r => r.Save(It.Is<Order>(o => 
            o.CustomerId == 42 && o.Total == 100)));
    }
    
    [Fact]
    public void Verify_NoOtherCalls()
    {
        var mock = new Mock<IOrderRepository>();
        
        // Do something with mock
        var _ = mock.Object.GetById(1);
        
        // Verify GetById was called
        mock.Verify(r => r.GetById(1), Times.Once);
        
        // Verify no other methods were called
        mock.VerifyNoOtherCalls();
    }
}
```

### Times Options

```csharp
mock.Verify(m => m.Method(), Times.Never);        // Never called
mock.Verify(m => m.Method(), Times.Once);       // Exactly once
mock.Verify(m => m.Method(), Times.Exactly(5));   // Exactly 5 times
mock.Verify(m => m.Method(), Times.AtLeast(2));   // 2 or more times
mock.Verify(m => m.Method(), Times.AtMost(3));    // 3 or fewer times
mock.Verify(m => m.Method(), Times.AtLeastOnce);  // 1 or more
mock.Verify(m => m.Method(), Times.AtMostOnce);    // 0 or 1 times
mock.Verify(m => m.Method(), Times.Between(2, 4, Range.Inclusive)); // 2-4 times
```

---

## Advanced Mocking Patterns

### Mock Properties

```csharp
[Fact]
public void Mock_Properties()
{
        var mock = new Mock<IConfiguration>();
    
    // Setup property getter
    mock.SetupGet(c => c["ConnectionString"]).Returns("Server=...");
    
    // Setup property setter
    mock.SetupSet(c => c["Key"] = "value");
    
    // Verify property was set
    mock.VerifySet(c => c["Key"] = "value");
}
```

### Sequential Calls

```csharp
[Fact]
public void Setup_Sequence()
{
    var mock = new Mock<IOrderRepository>();
    
    // Setup sequence of calls
    mock.SetupSequence(r => r.GetById(1))
        .Returns(new Order { Status = OrderStatus.Pending })
        .Returns(new Order { Status = OrderStatus.Shipped })
        .Throws(new NotFoundException());
    
    // First call
    var order1 = mock.Object.GetById(1); // Pending
    
    // Second call
    var order2 = mock.Object.GetById(1); // Shipped
    
    // Third call
    Assert.Throws<NotFoundException>(() => mock.Object.GetById(1));
}
```

### Mock Events

```csharp
public class EventExamples
{
    [Fact]
    public void Mock_Events()
    {
        var mock = new Mock<IOrderService>();
        OrderEventArgs? capturedArgs = null;
        
        // Subscribe to mocked event
        mock.Object.OrderCreated += (s, e) => capturedArgs = e;
        
        // Raise event
        mock.Raise(
            s => s.OrderCreated += null,
            new OrderEventArgs { OrderId = 1 });
        
        capturedArgs?.OrderId.ShouldBe(1);
    }
}
```

### Async Mocks

```csharp
public class AsyncExamples
{
    [Fact]
    public async Task Mock_Async()
    {
        var mock = new Mock<IOrderRepository>();
        
        // Setup async method
        mock.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(new Order { Id = 1 });
        
        // Setup async with delay
        mock.Setup(r => r.GetByIdAsync(2))
            .Returns(async () =>
            {
                await Task.Delay(100);
                return new Order { Id = 2 };
            });
        
        // Act
        var order = await mock.Object.GetByIdAsync(1);
        
        // Assert
        order.Id.ShouldBe(1);
    }
}
```

---

## Mock vs Stub vs Fake

### Definitions

```csharp
// STUB - Provides canned answers
var stub = new Mock<IEmailService>();
stub.Setup(e => e.Send(It.IsAny<Email>())).Returns(true);
// Only configures returns - no verification

// MOCK - Verifies behavior
var mock = new Mock<IEmailService>();
service.ProcessOrder(order);
mock.Verify(e => e.SendOrderConfirmation(order), Times.Once);
// Verifies interaction happened

// FAKE - Working implementation
public class FakeEmailService : IEmailService
{
    public List<Email> SentEmails { get; } = new();
    
    public bool Send(Email email)
    {
        SentEmails.Add(email);
        return true;
    }
}
```

### When to Use Each

| Type | Use When | Example |
|------|----------|---------|
| Stub | Need to isolate unit under test | Repository returning test data |
| Mock | Need to verify interactions | Verify email was sent |
| Fake | Need working implementation | In-memory database |

---

## Real-World Patterns

### Pattern 1: Service with Multiple Dependencies

```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;
    private readonly Mock<ILogger> _mockLogger;
    private readonly OrderService _service;

    public OrderServiceTests()
    {
        _mockRepo = new Mock<IOrderRepository>();
        _mockEmail = new Mock<IEmailService>();
        _mockLogger = new Mock<ILogger>();
        
        _service = new OrderService(
            _mockRepo.Object,
            _mockEmail.Object,
            _mockLogger.Object);
    }

    [Fact]
    public async Task CreateOrder_WithValidItems_SavesAndNotifies()
    {
        // Arrange
        var order = new Order
        {
            CustomerId = 1,
            Items = new List<OrderItem>
            {
                new() { ProductId = 1, Quantity = 2, Price = 10 }
            }
        };
        
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
            .ReturnsAsync(order);
        
        // Act
        var result = await _service.CreateOrderAsync(order);
        
        // Assert
        result.ShouldNotBeNull();
        result.Total.ShouldBe(20);
        
        _mockRepo.Verify(r => r.SaveAsync(It.Is<Order>(o => 
            o.Total == 20)), Times.Once);
        
        _mockEmail.Verify(e => e.SendOrderConfirmationAsync(order), Times.Once);
        
        _mockLogger.Verify(l => l.LogInformation(It.IsAny<string>()), Times.AtLeastOnce);
    }

    [Fact]
    public async Task CreateOrder_WithRepositoryFailure_LogsError()
    {
        // Arrange
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>()))
            .ThrowsAsync(new DatabaseException("Connection failed"));
        
        // Act & Assert
        await Should.ThrowAsync<DatabaseException>(async () =>
        {
            await _service.CreateOrderAsync(new Order());
        });
        
        _mockLogger.Verify(l => l.LogError(It.IsAny<Exception>(), It.IsAny<string>()));
    }
}
```

### Pattern 2: Repository Pattern

```csharp
public class OrderRepositoryTests
{
    // Test the actual repository with InMemory database
    [Fact]
    public async Task GetById_ExistingOrder_ReturnsOrder()
    {
        // Arrange
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        
        using var context = new AppDbContext(options);
        context.Orders.Add(new Order { Id = 1, CustomerId = 42 });
        await context.SaveChangesAsync();
        
        var repository = new OrderRepository(context);
        
        // Act
        var order = await repository.GetByIdAsync(1);
        
        // Assert
        order.ShouldNotBeNull();
        order.CustomerId.ShouldBe(42);
    }
}
```

---

## Best Practices

### ✅ DO's

```csharp
// 1. Mock interfaces, not concrete classes
var mock = new Mock<IOrderRepository>(); // ✅
var mock = new Mock<OrderRepository>();   // ❌ Prefer interfaces

// 2. Use meaningful variable names
var orderRepositoryMock = new Mock<IOrderRepository>(); // ✅
var mock = new Mock<IOrderRepository>();             // ❌ Too vague

// 3. Verify only what matters
mock.Verify(r => r.Save(It.IsAny<Order>()), Times.Once); // ✅
// Don't verify every single method call

// 4. Setup only what's needed
// Don't setup every method if not used in test

// 5. Use appropriate verification
mock.Verify(r => r.GetById(1), Times.Once); // Exact match
mock.Verify(r => r.GetById(It.IsAny<int>())); // Any value
mock.Verify(r => r.GetById(It.IsInRange(1, 100))); // Range

// 6. Clean up with VerifyNoOtherCalls when needed
mock.VerifyNoOtherCalls(); // Ensures no unexpected calls
```

### ❌ DON'Ts

```csharp
// 1. Don't mock what you don't own
var mock = new Mock<HttpClient>(); // ❌ Don't mock framework classes
// Use HttpClientFactory with test server instead

// 2. Don't verify every call
mock.Verify(r => r.Method1()); // ❌ Over-specification
mock.Verify(r => r.Method2());
mock.Verify(r => r.Method3());
// Just verify the important interactions

// 3. Don't setup return values that aren't used
mock.Setup(r => r.GetById(1)).Returns(order); // ❌ if not called

// 4. Don't use mocks for value objects
var mock = new Mock<OrderDto>(); // ❌ OrderDto is just data

// 5. Don't forget to verify async methods properly
mock.Verify(r => r.SaveAsync(It.IsAny<Order>())); // ✅
mock.Verify(r => r.Save(It.IsAny<Order>())); // ❌ Wrong method
```

---

## Interview Questions

**Q: What is mocking and why is it used?**> Mocking creates test doubles that simulate real dependencies. It's used to isolate the unit under test, making tests faster, more reliable, and independent of external systems like databases or APIs.

**Q: What's the difference between a mock and a stub?**> A stub provides canned answers to calls ("return 42 when asked"). A mock verifies behavior ("verify this method was called with these arguments"). Stubs help with state verification, mocks help with interaction verification.

**Q: When should you not use mocks?**> Don't use mocks for: (1) Value objects/DTOs, (2) Simple utilities with no dependencies, (3) When testing interactions with external systems (use integration tests), (4) When the mock becomes more complex than the real implementation.

**Q: What's wrong with mocking concrete classes?**> Mocking concrete classes can lead to fragile tests because internal implementation details may change. Mocking interfaces is preferred because interfaces define the contract that should be stable.

**Q: How do you verify that a method was never called in Moq?**> Use `mock.Verify(m => m.Method(), Times.Never)` or `mock.VerifyNoOtherCalls()` after verifying the expected calls to ensure no unexpected interactions occurred.

**Q: What's the difference between MockBehavior.Strict and MockBehavior.Loose?**> Strict requires all methods to be explicitly setup and throws if an unexpected call is made. Loose (default) returns default values for unconfigured methods. Strict helps catch unexpected calls, Loose is more flexible.
