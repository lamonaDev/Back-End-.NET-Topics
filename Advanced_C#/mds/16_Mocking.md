# Mocking in C# with Moq

## 🧠 What Is Mocking?

**Mocking** is the practice of replacing real dependencies (databases, APIs, email services, etc.) with **fake, controllable versions** during unit testing. A mock lets you test a class **in isolation** — without actually hitting a database or sending emails.

---

## 🌍 Real-World Analogy

A flight simulator is a mock. Pilots train by using a fake cockpit that *behaves like* a real plane without any of the actual risk. You can simulate engine failure, bad weather, landing — all in a safe, controlled environment.

Mocks are simulators for your dependencies. They *behave like* the real thing, but you control what they return and can verify how they were called.

---

## 🆚 Mock vs Stub vs Fake

| Type | What It Does | Example |
|---|---|---|
| **Stub** | Returns hardcoded values — no behavior verification | `GetUser()` always returns a fixed user object |
| **Mock** | Returns values AND **verifies interactions** (was it called? how many times?) | Verify `SendEmail()` was called exactly once |
| **Fake** | A working minimal implementation (e.g., in-memory DB) | `InMemoryRepository` instead of SQL |

> In practice, **Moq** creates mocks that can also act as stubs. The distinction is mostly conceptual.

---

## ⚙️ Architecture: Why Dependencies Break Unit Tests

```
Class Under Test: OrderService
      │
      ├── depends on: IEmailService       ← sends real emails in production
      ├── depends on: IOrderRepository    ← hits real database
      └── depends on: IPaymentGateway     ← charges real credit cards

Without mocking: unit test would hit DB, send emails, charge cards ← WRONG
With mocking:    all dependencies are replaced with controlled fakes ← CORRECT
```

---

## 💻 Setup

```bash
dotnet add package Moq
dotnet add package Shouldly
```

---

## 💻 Full Example: Mocking a Service Dependency

### Step 1: Define the interface and class under test

```csharp
// Interface (the dependency contract)
public interface IEmailService
{
    void SendWelcomeEmail(string toAddress, string userName);
    bool IsValidEmail(string email);
}

// Interface for data access
public interface IUserRepository
{
    User? GetById(int id);
    void Save(User user);
}

// The class we want to test
public class UserRegistrationService
{
    private readonly IEmailService _emailService;
    private readonly IUserRepository _userRepo;

    public UserRegistrationService(IEmailService emailService, IUserRepository userRepo)
    {
        _emailService = emailService;
        _userRepo = userRepo;
    }

    public bool RegisterUser(string email, string name)
    {
        if (!_emailService.IsValidEmail(email))
            return false;

        var user = new User { Email = email, Name = name };
        _userRepo.Save(user);
        _emailService.SendWelcomeEmail(email, name);
        return true;
    }
}

public class User
{
    public string Email { get; set; } = "";
    public string Name { get; set; } = "";
}
```

### Step 2: Write tests with Moq

```csharp
using Moq;
using Shouldly;
using Xunit;

public class UserRegistrationServiceTests
{
    private readonly Mock<IEmailService> _mockEmailService;
    private readonly Mock<IUserRepository> _mockUserRepo;
    private readonly UserRegistrationService _sut;

    public UserRegistrationServiceTests()
    {
        // Arrange (shared setup): create mocks
        _mockEmailService = new Mock<IEmailService>();
        _mockUserRepo = new Mock<IUserRepository>();

        // Inject mocks into the class under test
        _sut = new UserRegistrationService(
            _mockEmailService.Object,
            _mockUserRepo.Object
        );
    }

    [Fact]
    public void RegisterUser_ValidEmail_ReturnsTrueAndSavesUser()
    {
        // Arrange: configure mock behavior
        _mockEmailService
            .Setup(s => s.IsValidEmail("alice@example.com"))
            .Returns(true); // ← mock will return true for this call

        // Act
        bool result = _sut.RegisterUser("alice@example.com", "Alice");

        // Assert: check return value
        result.ShouldBeTrue();

        // Assert: verify Save was called exactly once with a User that has correct email
        _mockUserRepo.Verify(
            r => r.Save(It.Is<User>(u => u.Email == "alice@example.com")),
            Times.Once
        );

        // Assert: verify welcome email was sent
        _mockEmailService.Verify(
            s => s.SendWelcomeEmail("alice@example.com", "Alice"),
            Times.Once
        );
    }

    [Fact]
    public void RegisterUser_InvalidEmail_ReturnsFalse_NoSave()
    {
        // Arrange: mock returns false for invalid email
        _mockEmailService
            .Setup(s => s.IsValidEmail("not-an-email"))
            .Returns(false);

        // Act
        bool result = _sut.RegisterUser("not-an-email", "Bob");

        // Assert
        result.ShouldBeFalse();

        // Verify Save was NEVER called (registration aborted)
        _mockUserRepo.Verify(r => r.Save(It.IsAny<User>()), Times.Never);

        // Verify welcome email was NEVER sent
        _mockEmailService.Verify(
            s => s.SendWelcomeEmail(It.IsAny<string>(), It.IsAny<string>()),
            Times.Never
        );
    }
}
```

---

## 🔧 Essential Moq Patterns

### Setting up return values

```csharp
// Return a value
mock.Setup(s => s.GetUser(42)).Returns(new User { Name = "Alice" });

// Return null
mock.Setup(s => s.GetUser(999)).Returns((User?)null);

// Throw an exception
mock.Setup(s => s.GetUser(-1)).Throws(new ArgumentException("Invalid ID"));

// Return different values on successive calls
mock.SetupSequence(s => s.GetNextId())
    .Returns(1)
    .Returns(2)
    .Returns(3);
```

### Argument matching with `It`

```csharp
// Match any argument of the given type
mock.Setup(s => s.IsValidEmail(It.IsAny<string>())).Returns(true);

// Match with a condition
mock.Setup(s => s.GetUser(It.Is<int>(id => id > 0))).Returns(someUser);
```

### Verifying interactions

```csharp
// Was the method called at all?
mock.Verify(s => s.SendEmail(It.IsAny<string>()), Times.AtLeastOnce);

// Was it called exactly N times?
mock.Verify(s => s.LogEvent("Login"), Times.Exactly(3));

// Was it NEVER called?
mock.Verify(s => s.DeleteUser(It.IsAny<int>()), Times.Never);
```

---

## 💻 Mocking Async Methods

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task SaveAsync(Order order);
}

// Mock async setup:
_mockRepo
    .Setup(r => r.GetByIdAsync(1))
    .ReturnsAsync(new Order { Id = 1, Amount = 99.99m });

_mockRepo
    .Setup(r => r.SaveAsync(It.IsAny<Order>()))
    .Returns(Task.CompletedTask);
```

---

## 🧠 Memory Model: Where Mocks Live

```
Test Process
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Mock<IEmailService>              Mock<IUserRepository>│
│  ┌────────────────────┐           ┌──────────────────┐ │
│  │ .Object (proxy)    │           │ .Object (proxy)  │ │
│  │ Setup rules        │           │ Setup rules      │ │
│  │ Call recorder      │           │ Call recorder    │ │
│  └────────────────────┘           └──────────────────┘ │
│           │                               │             │
│           └───────────────────────────────┘             │
│                         │                               │
│              UserRegistrationService (SUT)              │
│              (receives mock objects via constructor)    │
└────────────────────────────────────────────────────────┘
```

> Moq creates a **runtime proxy class** that implements the interface. Every call goes through the proxy, which checks your setup rules and records the call for later verification.

---

## 📌 Summary

> **Mocking** lets you test classes in complete isolation by replacing real dependencies with controlled fakes. Use **Moq** to set up return values, throw exceptions, and verify that specific methods were called the correct number of times. Always inject dependencies via interfaces — this is what makes mocking possible.
