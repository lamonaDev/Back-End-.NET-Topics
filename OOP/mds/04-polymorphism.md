# Polymorphism in C#

## What is Polymorphism?

Polymorphism means **"many forms"** — the ability of an object to take on many forms. In OOP, it allows methods to behave differently based on the object that invokes them, enabling a single interface to represent different underlying forms (data types).

> **In simple terms:** Polymorphism means "the same call can do different things depending on the actual object."

---

## The Real-World Analogy

Think of the **verb "open"**:

- Open a **door** → turn handle, push/pull
- Open a **file** → click icon, read from disk
- Open a **bank account** → fill forms, verify identity
- Open a **conversation** → say "hello", ask questions

Same word, different implementations. The caller just says "open" — the specific action depends on what is being opened.

---

## Types of Polymorphism in C#

| Type | Also Called | Binding Time | Mechanism |
|------|-------------|--------------|-----------|
| **Static** | Compile-time, Early | Compile | Method Overloading, Operator Overloading |
| **Dynamic** | Runtime, Late | Runtime | Method Overriding, Virtual Methods |

---

## Static Polymorphism (Compile-Time)

Resolved at compile time based on method signature.

### Method Overloading

```csharp
public class Calculator
{
    // Same method name, different parameters
    public int Add(int a, int b) => a + b;
    
    public int Add(int a, int b, int c) => a + b + c;
    
    public double Add(double a, double b) => a + b;
    
    public string Add(string a, string b) => a + b;
}

// Usage - compiler picks correct version at compile time
var calc = new Calculator();
calc.Add(1, 2);        // Calls int version
calc.Add(1, 2, 3);     // Calls 3-param version
calc.Add(1.5, 2.5);    // Calls double version
calc.Add("Hello", " "); // Calls string version
```

### Memory Perspective (Static Polymorphism)

```
COMPILE TIME:
┌──────────────────────────────────────────────────────────────┐
│  Source: calc.Add(1, 2)                                        │
│  Compiler sees: two int parameters                              │
│  Result: Emit call to Calculator.Add(int, int)                │
│  Binding: FIXED at compile time                                 │
└──────────────────────────────────────────────────────────────┘

RUNTIME:
┌──────────────────────────────────────────────────────────────┐
│  No decision needed - address already known                     │
│  Direct jump to Calculator.Add(int, int)                      │
│  Fast execution - no lookup overhead                            │
└──────────────────────────────────────────────────────────────┘
```

---

## Dynamic Polymorphism (Runtime)

Resolved at runtime based on the actual object's type.

### Method Overriding

```csharp
public class Animal
{
    // Virtual: CAN be overridden
    public virtual void MakeSound()
    {
        Console.WriteLine("Generic animal sound");
    }
    
    // Virtual property
    public virtual string Species => "Unknown";
}

public class Dog : Animal
{
    // Override: Replace base implementation
    public override void MakeSound()
    {
        Console.WriteLine("Woof! Woof!");
    }
    
    public override string Species => "Canis lupus familiaris";
}

public class Cat : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Meow!");
    }
    
    public override string Species => "Felis catus";
}

public class Parrot : Animal
{
    // No override - uses base implementation
}

// The Power of Polymorphism
public void AnimalConcert(List<Animal> animals)
{
    foreach (var animal in animals)
    {
        // Same call - different behavior!
        animal.MakeSound();
        // Dog → "Woof! Woof!"
        // Cat → "Meow!"
        // Parrot → "Generic animal sound" (base)
    }
}

// Usage
var animals = new List<Animal>
{
    new Dog(),
    new Cat(),
    new Parrot()
};

AnimalConcert(animals);  // Each animal makes its own sound!
```

---

## Memory Allocation for Dynamic Polymorphism

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VIRTUAL METHOD TABLE (vtable)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Each class has its own method table containing addresses for       │
│  virtual methods. When a method is overridden, the vtable slot     │
│  is replaced with the derived class's implementation.             │
│                                                                      │
│  Animal vtable:                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Slot 0: MakeSound() → Animal.MakeSound                       │    │
│  │ Slot 1: get_Species() → Animal.get_Species                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              │ Override replaces slot              │
│                              ▼                                      │
│  Dog vtable:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Slot 0: MakeSound() → Dog.MakeSound  ◄── DIFFERENT ADDRESS   │    │
│  │ Slot 1: get_Species() → Dog.get_Species ◄── DIFFERENT        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              │ Override replaces slot              │
│                              ▼                                      │
│  Cat vtable:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Slot 0: MakeSound() → Cat.MakeSound                          │    │
│  │ Slot 1: get_Species() → Cat.get_Species                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  HEAP OBJECTS:                                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │ Dog @ 0x1000    │  │ Cat @ 0x2000    │  │ Parrot @ 0x3000 │     │
│  │ ├─ MT Ptr ──────┼──┼─────────────────┼──┼─→ Dog vtable     │     │
│  │ ├─ [Animal base]│  │ ├─ MT Ptr ──────┼──┼─→ Cat vtable     │     │
│  │ └─ Breed        │  │ │ ├─ [Animal base]│  │ ├─ MT Ptr ─────┼──┐  │
│  │                 │  │ │ └─ Whiskers     │  │ │ ├─ Animal base│  │  │
│  │                 │  │ │                 │  │ │ └─ (no extra) │  │  │
│  │                 │  │ │                 │  │ │               │  │  │
│  │                 │  │ │                 │  │ └───────────────┘  │  │
│  │                 │  │ │                 │  └────────────────────  │  │
│  └─────────────────┘  └─────────────────┘                         │  │
│                                                                    │  │
│  DISPATCH:                                                         │  │
│  animal.MakeSound() is compiled as callvirt                       │  │
│  At runtime:                                                        │  │
│  1. Load 'animal' reference (e.g., 0x1000)                          │  │
│  2. Follow MT Ptr to vtable (Dog vtable)                          │  │
│  3. Read Slot 0 (MakeSound) → Dog.MakeSound address               │  │
│  4. Jump to that address and execute                              │  │
│                                                                     │  │
│  Same IL code, different execution paths based on actual object!   │  │
│                                                                     │  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Dynamic Dispatch in Action

```csharp
public class PaymentProcessor
{
    public virtual string Process(decimal amount) => "Base processing";
}

public class StripeProcessor : PaymentProcessor
{
    public override string Process(decimal amount) => $"Stripe: Charged {amount:C}";
}

public class PayPalProcessor : PaymentProcessor
{
    public override string Process(decimal amount) => $"PayPal: Charged {amount:C}";
}

public class PaymentService
{
    // This method knows NOTHING about Stripe or PayPal!
    // It just knows it has a PaymentProcessor
    public string ChargeCustomer(PaymentProcessor processor, decimal amount)
    {
        return processor.Process(amount);
    }
}

// Usage
var service = new PaymentService();

// Same method, different processors
Console.WriteLine(service.ChargeCustomer(new StripeProcessor(), 100m));
// Output: Stripe: Charged $100.00

Console.WriteLine(service.ChargeCustomer(new PayPalProcessor(), 100m));
// Output: PayPal: Charged $100.00

// Can even add new processors WITHOUT changing PaymentService!
public class CryptoProcessor : PaymentProcessor
{
    public override string Process(decimal amount) => $"Crypto: Sent {amount} BTC";
}

Console.WriteLine(service.ChargeCustomer(new CryptoProcessor(), 100m));
// Output: Crypto: Sent 100 BTC
// PaymentService didn't change at all!
```

---

## Method Overriding Rules

### The `virtual` / `override` / `new` Hierarchy

```csharp
public class BaseClass
{
    public virtual void VirtualMethod() => Console.WriteLine("Base virtual");
    public void NonVirtualMethod() => Console.WriteLine("Base non-virtual");
}

public class DerivedClass : BaseClass
{
    // CORRECT: Override for true polymorphism
    public override void VirtualMethod() => Console.WriteLine("Derived override");
    
    // WRONG: Using new (hiding)
    public new void NonVirtualMethod() => Console.WriteLine("Derived new");
}

// Demonstrating the difference
void TestPolymorphism()
{
    BaseClass obj = new DerivedClass();
    
    obj.VirtualMethod();      // "Derived override" - runtime dispatch
    obj.NonVirtualMethod();   // "Base non-virtual" - compile-time binding!
    
    // The 'new' method is only accessible through DerivedClass reference
    var derived = new DerivedClass();
    derived.NonVirtualMethod(); // "Derived new"
}
```

---

## Abstract Classes and Polymorphism

```csharp
public abstract class Shape
{
    // Abstract: MUST be implemented
    public abstract double CalculateArea();
    public abstract double CalculatePerimeter();
    
    // Virtual: CAN be overridden
    public virtual string Description => $"Area: {CalculateArea():F2}";
}

public class Circle : Shape
{
    public double Radius { get; set; }
    
    public override double CalculateArea() => Math.PI * Radius * Radius;
    public override double CalculatePerimeter() => 2 * Math.PI * Radius;
    public override string Description => $"Circle (r={Radius}): {base.Description}";
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    
    public override double CalculateArea() => Width * Height;
    public override double CalculatePerimeter() => 2 * (Width + Height);
}

public class Triangle : Shape
{
    public double A { get; set; }
    public double B { get; set; }
    public double C { get; set; }
    
    public override double CalculateArea()
    {
        // Heron's formula
        var s = (A + B + C) / 2;
        return Math.Sqrt(s * (s - A) * (s - B) * (s - C));
    }
    
    public override double CalculatePerimeter() => A + B + C;
}

// Polymorphic method
public void PrintShapeInfo(List<Shape> shapes)
{
    foreach (var shape in shapes)
    {
        Console.WriteLine(shape.Description);
        Console.WriteLine($"  Perimeter: {shape.CalculatePerimeter():F2}");
        Console.WriteLine();
    }
}

// Usage
var shapes = new List<Shape>
{
    new Circle { Radius = 5 },
    new Rectangle { Width = 4, Height = 6 },
    new Triangle { A = 3, B = 4, C = 5 }
};

PrintShapeInfo(shapes);
// Each shape calculates its own area and perimeter!
```

---

## Polymorphism with Interfaces

```csharp
public interface ILogger
{
    void Log(string message);
    void LogError(string message);
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => 
        Console.WriteLine($"[INFO] {DateTime.Now}: {message}");
    
    public void LogError(string message) => 
        Console.WriteLine($"[ERROR] {DateTime.Now}: {message}");
}

public class FileLogger : ILogger
{
    private readonly string _filePath;
    
    public FileLogger(string filePath) => _filePath = filePath;
    
    public void Log(string message) => File.AppendAllText(_filePath, $"INFO: {message}\n");
    
    public void LogError(string message) => File.AppendAllText(_filePath, $"ERROR: {message}\n");
}

public class DatabaseLogger : ILogger
{
    private readonly DbContext _context;
    
    public DatabaseLogger(DbContext context) => _context = context;
    
    public void Log(string message) => _context.Logs.Add(new LogEntry { Level = "INFO", Message = message });
    
    public void LogError(string message) => _context.Logs.Add(new LogEntry { Level = "ERROR", Message = message });
}

public class ApplicationService
{
    private readonly ILogger _logger;
    
    // Depends on abstraction, not concrete implementation
    public ApplicationService(ILogger logger)
    {
        _logger = logger;
    }
    
    public void DoWork()
    {
        _logger.Log("Starting work");
        // ... do work ...
        _logger.Log("Work completed");
    }
}

// Different logging strategies, same code!
var service1 = new ApplicationService(new ConsoleLogger());
var service2 = new ApplicationService(new FileLogger("app.log"));
var service3 = new ApplicationService(new DatabaseLogger(dbContext));
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    POLYMORPHISM AT A GLANCE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   STATIC (Compile-Time)                    DYNAMIC (Runtime)         │
│   ─────────────────────                  ─────────────────         │
│   Same name, different signatures          Same call, different      │
│                                            implementations           │
│                                                                      │
│   ┌─────────────────┐                     ┌─────────────────┐         │
│   │ Method          │                     │ Method          │         │
│   │ Overloading     │                     │ Overriding      │         │
│   └────────┬────────┘                     └────────┬────────┘         │
│            │                                       │                  │
│            ▼                                       ▼                  │
│   Resolved at COMPILE                        Resolved at RUNTIME       │
│   time by compiler                           by CLR via vtable         │
│                                                                      │
│   Example:                                 Example:                    │
│   Add(int, int)                            animal.MakeSound()        │
│   Add(double, double)                      ├─ Dog → "Woof!"          │
│   Add(string, string)                      ├─ Cat → "Meow!"          │
│                                            └─ Bird → "Tweet!"        │
│                                                                      │
│   IL: call                                 IL: callvirt              │
│   (direct address)                         (vtable lookup)           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: What's the difference between overriding and overloading?**
> Overloading: Same name, different parameters, compile-time resolution. Overriding: Same signature, replaces base implementation, runtime resolution via vtable.

**Q: Can you override a non-virtual method?**
> No. You can only hide it with `new`, but that breaks polymorphism. Only virtual/abstract methods support true overriding.

**Q: Why is virtual dispatch slower than direct calls?**
> Virtual dispatch requires: 1) dereferencing object to get vtable pointer, 2) indexing into vtable, 3) jumping to address. Direct calls jump to a fixed address.

**Q: What is the "diamond problem" and how does C# avoid it?**
> When a class inherits from two classes that share a common base. C# doesn't allow multiple inheritance; interfaces provide similar functionality without this issue.

**Q: How do you prevent a method from being overridden further?**
> Use the `sealed` keyword on the override: `public sealed override void Method()`.

**Q: Can properties be polymorphic?**
> Yes, properties are compiled to getter/setter methods which can be virtual/override just like regular methods.
