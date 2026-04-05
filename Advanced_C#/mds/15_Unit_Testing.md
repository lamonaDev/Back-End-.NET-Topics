# Unit Testing in C# with xUnit and Shouldly

## 🧠 What Is Unit Testing?

**Unit testing** is the practice of writing automated code that **verifies individual units of your application** work correctly in isolation. A "unit" is typically a single method or class.

Instead of manually running your app to test a feature, you write a test once and run it in milliseconds — any time your code changes.

---

## 🌍 Real-World Analogy

Think of a car assembly line. Before the car is assembled, each **individual part** (engine, brakes, steering) is tested independently. If the brake test fails, you know exactly which component broke — not "the car doesn't work."

Unit tests are the **quality checks for individual parts** of your software.

---

## 🧪 Types of Testing

| Type | What It Tests | Speed | Isolation |
|---|---|---|---|
| **Unit** | A single method/class in isolation | ⚡ Milliseconds | ✅ Full (no DB, no I/O) |
| **Integration** | Multiple components working together (e.g., service + DB) | 🐢 Seconds | ⚠️ Partial |
| **End-to-End (E2E)** | The entire application flow from user perspective | 🐌 Minutes | ❌ None — real system |

### When to use each:
- **Unit tests**: Validate business logic, edge cases, calculations.
- **Integration tests**: Validate database queries, API calls, messaging.
- **E2E tests**: Validate complete user scenarios (login, checkout, etc.).

---

## ⚙️ The AAA Pattern — Arrange, Act, Assert

Every good unit test follows this structure:

```
┌─────────────────────────────────────────┐
│  ARRANGE  │  Set up the test data and   │
│           │  objects needed             │
├─────────────────────────────────────────┤
│  ACT      │  Call the method under test │
├─────────────────────────────────────────┤
│  ASSERT   │  Verify the result is what  │
│           │  you expected               │
└─────────────────────────────────────────┘
```

---

## 💻 Setup: xUnit + Shouldly

```bash
# Create test project
dotnet new xunit -n MyApp.Tests

# Add Shouldly (fluent assertions)
dotnet add package Shouldly
```

---

## 💻 Task 1: Testing an `Add` Method

```csharp
// Production code
public class Calculator
{
    public int Add(int a, int b) => a + b;
}
```

```csharp
// Test file: CalculatorTests.cs
using Shouldly;
using Xunit;

public class CalculatorTests
{
    private readonly Calculator _sut; // SUT = System Under Test

    public CalculatorTests()
    {
        _sut = new Calculator(); // Arrange (shared setup)
    }

    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsCorrectSum()
    {
        // Arrange
        int a = 3, b = 5;

        // Act
        int result = _sut.Add(a, b);

        // Assert
        result.ShouldBe(8);
    }

    [Fact]
    public void Add_NegativeNumbers_ReturnsCorrectSum()
    {
        // Arrange & Act
        int result = _sut.Add(-3, -7);

        // Assert
        result.ShouldBe(-10);
    }

    [Fact]
    public void Add_ZeroAndNumber_ReturnsNumber()
    {
        _sut.Add(0, 42).ShouldBe(42);
    }

    // [Theory] + [InlineData] = data-driven tests
    [Theory]
    [InlineData(1, 1, 2)]
    [InlineData(5, 5, 10)]
    [InlineData(-1, 1, 0)]
    [InlineData(100, -50, 50)]
    public void Add_MultipleInputs_ReturnsCorrectSum(int a, int b, int expected)
    {
        _sut.Add(a, b).ShouldBe(expected);
    }
}
```

---

## 💻 Task 2: Testing a Multiply Method

```csharp
// Production code
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public int Multiply(int a, int b) => a * b;
}
```

```csharp
// Tests
[Fact]
public void Multiply_TwoPositiveNumbers_ReturnsProduct()
{
    // Arrange
    int a = 4, b = 3;

    // Act
    int result = _sut.Multiply(a, b);

    // Assert
    result.ShouldBe(12);
}

[Fact]
public void Multiply_ByZero_ReturnsZero()
{
    _sut.Multiply(999, 0).ShouldBe(0);
}

[Theory]
[InlineData(2, 3, 6)]
[InlineData(-2, 3, -6)]
[InlineData(-2, -3, 6)]
public void Multiply_Theory(int a, int b, int expected)
{
    _sut.Multiply(a, b).ShouldBe(expected);
}
```

---

## 💻 Task 3: Testing a Method That Returns a List

```csharp
// Production code
public class StudentService
{
    public List<string> GetStudents()
    {
        return new List<string> { "Alice", "Bob", "Carol" };
    }
}
```

```csharp
// Tests
public class StudentServiceTests
{
    private readonly StudentService _sut = new();

    [Fact]
    public void GetStudents_ReturnsNonEmptyList()
    {
        var result = _sut.GetStudents();
        result.ShouldNotBeEmpty();
    }

    [Fact]
    public void GetStudents_ReturnsThreeStudents()
    {
        var result = _sut.GetStudents();
        result.Count.ShouldBe(3);
    }

    [Fact]
    public void GetStudents_ContainsAlice()
    {
        var result = _sut.GetStudents();
        result.ShouldContain("Alice");
    }

    [Fact]
    public void GetStudents_ReturnsCorrectOrder()
    {
        var result = _sut.GetStudents();
        result[0].ShouldBe("Alice");
        result[1].ShouldBe("Bob");
        result[2].ShouldBe("Carol");
    }
}
```

---

## 💻 Task 4: Testing Whether a Student Exists

```csharp
// Production code
public class StudentService
{
    private readonly List<string> _students = new() { "Alice", "Bob", "Carol" };

    public bool StudentExists(string name) 
        => _students.Contains(name, StringComparer.OrdinalIgnoreCase);
}
```

```csharp
// Tests
[Fact]
public void StudentExists_KnownStudent_ReturnsTrue()
{
    var result = _sut.StudentExists("Alice");
    result.ShouldBeTrue();
}

[Fact]
public void StudentExists_UnknownStudent_ReturnsFalse()
{
    var result = _sut.StudentExists("Zara");
    result.ShouldBeFalse();
}

[Fact]
public void StudentExists_CaseInsensitive_ReturnsTrue()
{
    var result = _sut.StudentExists("alice"); // lowercase
    result.ShouldBeTrue();
}

[Fact]
public void StudentExists_EmptyString_ReturnsFalse()
{
    var result = _sut.StudentExists("");
    result.ShouldBeFalse();
}
```

---

## 🆚 [Fact] vs [Theory]

| Attribute | Use When | Example |
|---|---|---|
| `[Fact]` | One specific scenario | "Adding 2+2 returns 4" |
| `[Theory]` + `[InlineData]` | Same logic, multiple inputs | "Adding X+Y always returns X+Y" |

---

## 🆚 xUnit vs NUnit

| Feature | xUnit | NUnit |
|---|---|---|
| Test attribute | `[Fact]` | `[Test]` |
| Parameterized | `[Theory]` + `[InlineData]` | `[TestCase]` |
| Setup | Constructor | `[SetUp]` |
| Teardown | `IDisposable` | `[TearDown]` |
| Modern design | ✅ (designed for .NET Core) | Older, still maintained |

---

## 📌 Summary

> Unit testing with **xUnit + Shouldly** lets you verify individual methods work correctly in milliseconds. Follow the **AAA pattern** (Arrange, Act, Assert), use `[Fact]` for single scenarios and `[Theory]` for data-driven tests. Tests serve as living documentation and catch regressions before they reach production.
