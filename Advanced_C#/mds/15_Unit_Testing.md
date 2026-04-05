# Unit Testing with xUnit and Shouldly

## Table of Contents
1. [Understanding Unit Testing](#understanding-unit-testing)
2. [xUnit Fundamentals](#xunit-fundamentals)
3. [Test Structure and Organization](#test-structure-and-organization)
4. [Assertions with Shouldly](#assertions-with-shouldly)
5. [Test Data and Theories](#test-data-and-theories)
6. [Real-World Patterns](#real-world-patterns)
7. [Best Practices](#best-practices)

---

## Understanding Unit Testing

### What is Unit Testing?

Unit testing is the practice of testing **individual units of code** (methods, classes) in isolation to verify they behave as expected. Tests are automated, repeatable, and run quickly.

### The Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────┐
│                    TESTING PYRAMID                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                          /\                                      │
│                         /  \                                     │
│                        /E2E \        Few tests                  │
│                       / Tests \       Slow, expensive           │
│                      /__________\                               │
│                     /            \                             │
│                    /  Integration  \    Medium tests            │
│                   /     Tests       \    Test interactions       │
│                  /___________________\                           │
│                 /                      \                         │
│                /      Unit Tests        \   Many tests            │
│               /      (Focus here!)     \   Fast, isolated        │
│              /__________________________\                        │
│                                                                  │
│   Unit Test Characteristics:                                     │
│   ├─ Fast (< 10ms each)                                         │
│   ├─ Isolated (no dependencies)                                  │
│   ├─ Repeatable (same result every time)                        │
│   ├─ Self-verifying (pass/fail clearly)                         │
│   └─ Timely (write with production code)                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The AAA Pattern

```csharp
[Fact]
public void CalculateTotal_WithItems_ReturnsSum()
{
    // Arrange - Set up the test
    var cart = new ShoppingCart();
    cart.AddItem(new Item { Price = 10 });
    cart.AddItem(new Item { Price = 20 });
    
    // Act - Execute the code being tested
    var total = cart.CalculateTotal();
    
    // Assert - Verify the result
    Assert.Equal(30, total);
}
```

---

## xUnit Fundamentals

### Basic Test Attributes

```csharp
// Install packages:
// dotnet add package xunit
// dotnet add package xunit.runner.visualstudio
// dotnet add package Microsoft.NET.Test.Sdk

public class CalculatorTests
{
    // [Fact] - A simple test method
    [Fact]
    public void Add_TwoNumbers_ReturnsSum()
    {
        var calculator = new Calculator();
        
        var result = calculator.Add(2, 3);
        
        Assert.Equal(5, result);
    }
    
    // [Theory] - Parameterized test
    [Theory]
    [InlineData(1, 1, 2)]
    [InlineData(2, 3, 5)]
    [InlineData(-1, -1, -2)]
    [InlineData(0, 0, 0)]
    public void Add_VariousInputs_ReturnsExpectedSum(int a, int b, int expected)
    {
        var calculator = new Calculator();
        
        var result = calculator.Add(a, b);
        
        Assert.Equal(expected, result);
    }
    
    // Skip a test temporarily
    [Fact(Skip = "Not implemented yet")]
    public void Multiply_NotImplemented() { }
}
```

### Test Lifecycle

```csharp
public class DatabaseTests : IDisposable
{
    private readonly DatabaseConnection _connection;
    
    // Constructor runs before EACH test
    public DatabaseTests()
    {
        _connection = new DatabaseConnection("test_db");
        _connection.Open();
    }
    
    // IDisposable.Dispose runs after EACH test
    public void Dispose()
    {
        _connection.Close();
    }
    
    [Fact]
    public void Query_ReturnsResults()
    {
        var results = _connection.Query("SELECT * FROM Users");
        Assert.NotEmpty(results);
    }
}

// Alternative: IAsyncLifetime for async setup
public class AsyncDatabaseTests : IAsyncLifetime
{
    private DatabaseConnection _connection = null!;
    
    public async Task InitializeAsync()
    {
        _connection = new DatabaseConnection("test_db");
        await _connection.OpenAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _connection.CloseAsync();
    }
    
    [Fact]
    public async Task QueryAsync_ReturnsResults()
    {
        var results = await _connection.QueryAsync("SELECT * FROM Users");
        Assert.NotEmpty(results);
    }
}
```

### Collection Fixtures (Shared Context)

```csharp
// Shared context across multiple test classes
public class DatabaseFixture : IAsyncLifetime
{
    public DatabaseConnection Connection { get; private set; } = null!;
    
    public async Task InitializeAsync()
    {
        Connection = new DatabaseConnection("shared_test_db");
        await Connection.OpenAsync();
    }
    
    public async Task DisposeAsync()
    {
        await Connection.DisposeAsync();
    }
}

[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

[Collection("Database collection")]
public class UserRepositoryTests
{
    private readonly DatabaseConnection _connection;
    
    public UserRepositoryTests(DatabaseFixture fixture)
    {
        _connection = fixture.Connection;
    }
    
    [Fact]
    public void GetUser_ReturnsUser() { /* use _connection */ }
}

[Collection("Database collection")]
public class OrderRepositoryTests
{
    private readonly DatabaseConnection _connection;
    
    public OrderRepositoryTests(DatabaseFixture fixture)
    {
        _connection = fixture.Connection;
    }
    
    [Fact]
    public void GetOrder_ReturnsOrder() { /* use _connection */ }
}
```

---

## Test Structure and Organization

### Naming Conventions

```csharp
// Pattern: MethodName_StateUnderTest_ExpectedBehavior

public class CalculatorTests
{
    // ✅ Good names
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum() { }
    
    [Fact]
    public void Add_MaxIntValue_ThrowsOverflowException() { }
    
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException() { }
    
    [Fact]
    public void CalculateTotal_WithEmptyCart_ReturnsZero() { }
    
    // ❌ Bad names
    [Fact]
    public void Test1() { }
    
    [Fact]
    public void AddWorks() { }
}
```

### Project Organization

```
MySolution/
├── src/
│   └── MyApp/
│       ├── Services/
│       │   └── OrderService.cs
│       └── Models/
│           └── Order.cs
└── tests/
    └── MyApp.Tests/
        ├── Services/
        │   └── OrderServiceTests.cs    ← Mirrors source structure
        ├── Models/
        │   └── OrderTests.cs
        └── TestHelpers/
            └── TestDataBuilder.cs
```

---

## Assertions with Shouldly

### Why Shouldly?

```csharp
// ❌ Standard xUnit assertions - cryptic failures
Assert.Equal(expected, actual);
// Expected: 5
// Actual:   6

// ✅ Shouldly - readable failure messages
actual.ShouldBe(expected);
// actual should be 5 but was 6
```

### Common Shouldly Assertions

```csharp
// Install: dotnet add package Shouldly

public class ShouldlyExamples
{
    [Fact]
    public void BasicAssertions()
    {
        // Equality
        var result = 42;
        result.ShouldBe(42);
        result.ShouldNotBe(0);
        
        // Null checks
        string? name = "John";
        name.ShouldNotBeNull();
        name.ShouldBe("John");
        
        // Booleans
        var isActive = true;
        isActive.ShouldBeTrue();
        
        // Collections
        var numbers = new[] { 1, 2, 3 };
        numbers.ShouldContain(2);
        numbers.ShouldNotContain(4);
        numbers.Count().ShouldBe(3);
        
        // Strings
        var text = "Hello World";
        text.ShouldContain("World");
        text.ShouldStartWith("Hello");
        text.ShouldEndWith("World");
        
        // Types
        object value = "test";
        value.ShouldBeOfType<string>();
        
        // Exceptions
        Should.Throw<ArgumentException>(() =>
        {
            // Code that throws
            throw new ArgumentException();
        });
    }
    
    [Fact]
    public void CollectionAssertions()
    {
        var users = new[] { "Alice", "Bob", "Charlie" };
        
        users.ShouldContain("Alice");
        users.ShouldContain(u => u.StartsWith("A")); // Predicate
        users.ShouldNotBeEmpty();
        users.ShouldBeInOrder(); // Alphabetical
        
        // Equivalent to expected
        users.ShouldBe(new[] { "Alice", "Bob", "Charlie" });
        
        // Subset/superset
        users.ShouldBeSubsetOf(new[] { "Alice", "Bob", "Charlie", "David" });
    }
    
    [Fact]
    public void ObjectAssertions()
    {
        var user = new User { Name = "John", Age = 30 };
        var expected = new User { Name = "John", Age = 30 };
        
        // Deep comparison (by default)
        user.ShouldBeEquivalentTo(expected);
        
        // Ignoring fields
        user.ShouldBeEquivalentTo(expected, options =>
            options.Excluding(u => u.CreatedAt));
    }
}
```

---

## Test Data and Theories

### InlineData

```csharp
[Theory]
[InlineData(1, 1, 2)]
[InlineData(2, 3, 5)]
[InlineData(-1, -1, -2)]
public void Add_TwoNumbers_ReturnsExpectedSum(int a, int b, int expected)
{
    var calculator = new Calculator();
    
    var result = calculator.Add(a, b);
    
    result.ShouldBe(expected);
}
```

### MemberData

```csharp
public class CalculatorTests
{
    public static TheoryData<int, int, int> AdditionData = new()
    {
        { 1, 1, 2 },
        { 2, 3, 5 },
        { int.MaxValue, 0, int.MaxValue }
    };
    
    [Theory]
    [MemberData(nameof(AdditionData))]
    public void Add_WithMemberData_ReturnsExpectedSum(int a, int b, int expected)
    {
        var calculator = new Calculator();
        
        var result = calculator.Add(a, b);
        
        result.ShouldBe(expected);
    }
    
    // Dynamic data from method
    public static IEnumerable<object[]> GetComplexData()
    {
        yield return new object[] { new Order { Total = 100 }, 110 }; // 10% tax
        yield return new object[] { new Order { Total = 200 }, 220 };
    }
    
    [Theory]
    [MemberData(nameof(GetComplexData))]
    public void CalculateTotal_WithTax_ReturnsExpectedTotal(Order order, decimal expected)
    {
        var service = new OrderService();
        
        var result = service.CalculateTotal(order);
        
        result.ShouldBe(expected);
    }
}
```

### ClassData

```csharp
public class DivisionTestData : TheoryData<int, int, int>
{
    public DivisionTestData()
    {
        Add(10, 2, 5);
        Add(20, 4, 5);
        Add(100, 10, 10);
    }
}

public class CalculatorTests
{
    [Theory]
    [ClassData(typeof(DivisionTestData))]
    public void Divide_TwoNumbers_ReturnsExpectedQuotient(int dividend, int divisor, int expected)
    {
        var calculator = new Calculator();
        
        var result = calculator.Divide(dividend, divisor);
        
        result.ShouldBe(expected);
    }
}
```

---

## Real-World Patterns

### Testing Services

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateOrder_WithValidItems_CreatesOrder()
    {
        // Arrange
        var repository = new Mock<IOrderRepository>();
        var emailService = new Mock<IEmailService>();
        var service = new OrderService(repository.Object, emailService.Object);
        
        var items = new List<OrderItem>
        {
            new() { ProductId = 1, Quantity = 2, Price = 10 }
        };
        
        // Act
        var order = await service.CreateOrderAsync(1, items);
        
        // Assert
        order.ShouldNotBeNull();
        order.Total.ShouldBe(20);
        repository.Verify(r => r.SaveAsync(order), Times.Once);
        emailService.Verify(e => e.SendOrderConfirmationAsync(order), Times.Once);
    }
    
    [Fact]
    public async Task CreateOrder_WithEmptyItems_ThrowsValidationException()
    {
        var service = new OrderService(Mock.Of<IOrderRepository>(), Mock.Of<IEmailService>());
        
        await Should.ThrowAsync<ValidationException>(async () =>
        {
            await service.CreateOrderAsync(1, new List<OrderItem>());
        });
    }
}
```

### Testing Exceptions

```csharp
public class ExceptionTests
{
    [Fact]
    public void Constructor_NullName_ThrowsArgumentNullException()
    {
        Should.Throw<ArgumentNullException>(() =>
        {
            _ = new User(null!, "email@test.com");
        }).ParamName.ShouldBe("name");
    }
    
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        var calculator = new Calculator();
        
        var ex = Should.Throw<DivideByZeroException>(() =>
        {
            calculator.Divide(10, 0);
        });
        
        ex.Message.ShouldContain("divide");
    }
    
    [Fact]
    public async Task AsyncOperation_Timeout_ThrowsTimeoutException()
    {
        await Should.ThrowAsync<TimeoutException>(async () =>
        {
            await SlowOperationAsync().WaitAsync(TimeSpan.FromMilliseconds(10));
        });
    }
}
```

---

## Best Practices

### ✅ DO's

```csharp
// 1. One concept per test
[Fact]
public void CalculateTotal_WithSingleItem_ReturnsItemPrice() { }

[Fact]
public void CalculateTotal_WithMultipleItems_ReturnsSum() { }

[Fact]
public void CalculateTotal_WithDiscount_AppliesDiscount() { }

// 2. Use descriptive names
[Fact]
public void ProcessPayment_WithInsufficientFunds_ReturnsDeclined() { }

// 3. Test edge cases
[Theory]
[InlineData(0)]      // Zero
[InlineData(1)]      // One
[InlineData(-1)]     // Negative
[InlineData(int.MaxValue)] // Max value
public void IsPositive_VariousInputs_ReturnsExpectedResult(int value) { }

// 4. Keep tests fast
[Fact]
public void QuickOperation_CompletesQuickly()
{
    // Avoid Thread.Sleep, real I/O, etc.
}

// 5. Use builders for complex objects
[Fact]
public void Order_WithDiscount_CalculatesCorrectly()
{
    var order = new OrderBuilder()
        .WithItem("Product A", 100)
        .WithItem("Product B", 50)
        .WithDiscount(10)
        .Build();
    
    // Assert
}
```

### ❌ DON'Ts

```csharp
// 1. Multiple asserts testing different concepts
[Fact]
public void User_TestsEverything() // ❌ Too broad
{
    var user = new User("John", "john@test.com");
    
    Assert.Equal("John", user.Name);
    Assert.Equal("john@test.com", user.Email);
    Assert.True(user.IsActive);
    Assert.NotNull(user.CreatedAt);
}

// 2. Logic in tests
[Fact]
public void CalculateTotal_DynamicTest() // ❌ Hard to understand
{
    for (int i = 0; i < 10; i++)
    {
        // Test logic...
    }
}

// 3. Dependencies between tests
[Fact]
public void Test1() { _sharedValue = 5; }

[Fact]
public void Test2() { Assert.Equal(5, _sharedValue); } // ❌ Depends on Test1

// 4. External dependencies
[Fact]
public void Test_WithRealDatabase() // ❌ Slow, unreliable
{
    var db = new RealDatabaseConnection(); // Don't do this
}

// 5. Ignoring tests without explanation
[Fact(Skip = "")] // ❌ Why is this skipped?
public void ImportantTest() { }
```

---

## Interview Questions

**Q: What is the AAA pattern in unit testing?**
> AAA stands for Arrange, Act, Assert. Arrange sets up the test data and dependencies. Act executes the code being tested. Assert verifies the results meet expectations.

**Q: What's the difference between [Fact] and [Theory] in xUnit?**> [Fact] is a simple test with no parameters. [Theory] is a parameterized test that runs multiple times with different data, specified via [InlineData], [MemberData], or [ClassData].

**Q: Why use Shouldly over standard assertions?**> Shouldly provides more readable failure messages and fluent syntax. Instead of "Expected: 5, Actual: 6", you get "result should be 5 but was 6", making failures easier to understand.

**Q: What's a test fixture?**> A test fixture is the context in which tests run. In xUnit, the test class itself acts as a fixture. Shared fixtures can be created using ICollectionFixture for expensive setup (like database connections) shared across multiple test classes.

**Q: What makes a good unit test?**> Good unit tests are: Fast (<10ms), Isolated (no dependencies), Repeatable (same result every time), Self-verifying (clear pass/fail), and Timely (written with production code).

**Q: How do you test async methods in xUnit?**> Mark the test method as `async Task` and use `await` in the test body. Use `Should.ThrowAsync<T>()` for exception testing, and xUnit will properly handle the async operation.
