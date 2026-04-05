# Inheritance in C#

## What is Inheritance?

Inheritance is an **IS-A relationship** where a class (derived/child) acquires the properties and behaviors of another class (base/parent). It promotes code reuse and establishes a natural hierarchy between classes.

> **In simple terms:** Inheritance means "a specialized version of something that already exists."

---

## The Real-World Analogy

Think of **animal classification**:

- A `Dog` **IS-A** `Mammal`
- A `Mammal` **IS-A** `Animal`
- An `Animal` **IS-A** `LivingThing`

The `Dog` inherits characteristics from all its ancestors:
- From `Mammal`: warm-blooded, has fur, produces milk
- From `Animal`: breathes, moves, eats
- From `LivingThing`: grows, reproduces, responds to stimuli

**The dog adds its own specific traits** (barking, wagging tail) on top of inherited characteristics.

---

## When to Use Inheritance

### ✅ GOOD Use Cases

1. **True IS-A Relationship**
   ```csharp
   public class SavingsAccount : BankAccount { }
   // SavingsAccount IS-A BankAccount ✓
   ```

2. **Framework Extension Points**
   ```csharp
   public class MyController : Controller { }
   // Extending ASP.NET Core framework ✓
   ```

3. **Base Entity Pattern**
   ```csharp
   public class Order : BaseEntity { }
   // Shares common audit columns ✓
   ```

### ❌ BAD Use Cases (Code Smells)

1. **Code Reuse Only**
   ```csharp
   public class Stack : ArrayList { }
   // Stack IS-NOT-A ArrayList! Only wanted the methods ✗
   ```

2. **Deep Hierarchies (>2 levels)**
   ```csharp
   class Entity -> AuditableEntity -> SoftDeletableEntity -> TenantEntity -> Order
   // 5 levels! Changes at top break everything below ✗
   ```

---

## Basic Inheritance Syntax

```csharp
// Base class
public class Animal
{
    // Protected: accessible in derived classes
    protected string Name { get; set; }
    
    // Constructor
    public Animal(string name)
    {
        Name = name;
    }

    // Virtual: CAN be overridden
    public virtual void MakeSound()
    {
        Console.WriteLine($"{Name} makes a generic sound");
    }

    // Non-virtual: CANNOT be overridden
    public void Eat()
    {
        Console.WriteLine($"{Name} is eating");
    }
}

// Derived class
public class Dog : Animal  // Dog IS-A Animal
{
    // Additional property specific to Dog
    public string Breed { get; set; }

    // Constructor chaining to base
    public Dog(string name, string breed) : base(name)
    {
        Breed = breed;
    }

    // Override: Replace base implementation
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} barks: Woof! Woof!");
    }

    // New method specific to Dog
    public void Fetch()
    {
        Console.WriteLine($"{Name} fetches the ball");
    }
}

// Another derived class
public class Cat : Animal
{
    public int Whiskers { get; set; }

    public Cat(string name, int whiskers) : base(name)
    {
        Whiskers = whiskers;
    }

    public override void MakeSound()
    {
        Console.WriteLine($"{Name} meows: Meow!");
    }
}

// Usage
var dog = new Dog("Rex", "German Shepherd");
dog.MakeSound();  // Rex barks: Woof! Woof!
dog.Eat();       // Rex is eating (inherited)
dog.Fetch();     // Rex fetches the ball (specific)

var cat = new Cat("Whiskers", 24);
cat.MakeSound();  // Whiskers meows: Meow!
cat.Eat();        // Whiskers is eating (inherited)
```

---

## Memory Allocation for Inheritance

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INHERITANCE MEMORY LAYOUT                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  METHOD TABLE (Shared by all instances of same type):                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Animal Method Table                                         │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │ Slot 0: MakeSound() → Animal.MakeSound               │    │    │
│  │  │ Slot 1: Eat() → Animal.Eat                           │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                    │                                │
│                                    │ Override replaces slot       │
│                                    ▼                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Dog Method Table                                           │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │ Slot 0: MakeSound() → Dog.MakeSound  ◄── OVERRIDDEN  │    │    │
│  │  │ Slot 1: Eat() → Animal.Eat (inherited)               │    │    │
│  │  │ Slot 2: Fetch() → Dog.Fetch                          │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  HEAP OBJECT (Dog instance @ 0x7FFE2000):                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  [Object Header]          ← 8 bytes                          │    │
│  │  [Method Table Pointer] ───────┐ → Points to Dog vtable    │    │
│  │  [Animal base section]         │                             │    │
│  │    - Name: "Rex"              │                             │    │
│  │  [Dog specific section]        │                             │    │
│  │    - Breed: "German Shepherd" │                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Key: Object stores base class fields first (Animal), then          │
│  derived class fields (Dog). Single contiguous memory block.         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Types of Inheritance in C#

### 1. Single Inheritance

```csharp
class A { }
class B : A { }  // B inherits from A
// class C : A, B { }  // ❌ NOT ALLOWED in C#
```

### 2. Multilevel Inheritance

```csharp
class LivingThing { }
class Animal : LivingThing { }
class Mammal : Animal { }
class Dog : Mammal { }

// Dog has: LivingThing + Animal + Mammal + Dog features
```

### 3. Hierarchical Inheritance

```csharp
class Animal { }
class Dog : Animal { }
class Cat : Animal { }
class Bird : Animal { }
// Multiple classes inherit from same base
```

---

## Inheritance Keywords Deep Dive

### `virtual` and `override`

```csharp
public class Shape
{
    // Virtual: CAN be overridden by derived classes
    public virtual double GetArea() => 0;
    
    // Virtual with default implementation
    public virtual string Description => $"Shape with area {GetArea():F2}";
}

public class Circle : Shape
{
    public double Radius { get; set; }
    
    // Override: Replaces base implementation
    public override double GetArea() => Math.PI * Radius * Radius;
    
    // Can call base implementation
    public override string Description
    {
        get
        {
            var baseDesc = base.Description; // Call base getter
            return $"Circle: {baseDesc}";
        }
    }
}
```

### `new` (Method Hiding)

```csharp
public class Animal
{
    public void MakeSound() => Console.WriteLine("Generic sound");
}

public class Dog : Animal
{
    // new HIDES base method, doesn't override
    public new void MakeSound() => Console.WriteLine("Woof!");
}

// Critical difference:
Animal animal = new Dog();
animal.MakeSound();  // "Generic sound" (base version!)
// Dog.MakeSound() is HIDDEN when accessed through Animal reference

Dog dog = new Dog();
dog.MakeSound();     // "Woof!" (Dog's version)
```

### `sealed`

```csharp
// Prevent further inheritance
public sealed class FinalClass : SomeBase { }
// class Derived : FinalClass { }  // ❌ COMPILE ERROR

// Prevent method overriding
public class Base
{
    public virtual void Method() { }
}

public class Derived : Base
{
    public sealed override void Method() { }
    // No further override possible
}
```

### `abstract`

```csharp
public abstract class PaymentProcessor
{
    // Abstract: MUST be implemented by derived classes
    public abstract string ProcessPayment(decimal amount);
    
    // Virtual: CAN be overridden
    public virtual void Log(string message) { }
    
    // Concrete: Shared implementation
    public void ValidateAmount(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException();
    }
}

public class StripeProcessor : PaymentProcessor
{
    // Required implementation
    public override string ProcessPayment(decimal amount)
    {
        // Stripe-specific logic
        return "stripe_123";
    }
}
```

---

## The Liskov Substitution Principle (LSP)

> **"Objects of a superclass shall be replaceable with objects of its subclasses without breaking the application."**

### ✅ LSP-Compliant Example

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public virtual int Area => Width * Height;
}

public class Square : Rectangle
{
    // Square IS-A Rectangle mathematically
    // But this VIOLATES LSP!
    public override int Width
    {
        set { base.Width = value; base.Height = value; }
    }
    public override int Height
    {
        set { base.Width = value; base.Height = value; }
    }
}

// This breaks:
void TestRectangle(Rectangle rect)
{
    rect.Width = 5;
    rect.Height = 10;
    Debug.Assert(rect.Area == 50);  // FAILS for Square!
}
```

### ✅ Correct Design

```csharp
// Use composition instead of inheritance
public interface IShape
{
    int Area { get; }
}

public class Rectangle : IShape
{
    public int Width { get; set; }
    public int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : IShape
{
    public int Side { get; set; }
    public int Area => Side * Side;
}
```

---

## Constructor Chaining

```csharp
public class Employee
{
    public string Name { get; }
    public int Id { get; }
    
    public Employee(string name, int id)
    {
        Name = name;
        Id = id;
    }
}

public class Manager : Employee
{
    public string Department { get; }
    public List<Employee> Reports { get; }
    
    // Chaining to base constructor
    public Manager(string name, int id, string department) 
        : base(name, id)  // Must call base first
    {
        Department = department;
        Reports = new List<Employee>();
    }
    
    // Chaining to another constructor in same class
    public Manager(string name, int id) 
        : this(name, id, "General")  // Calls constructor above
    {
    }
}
```

---

## Common Pitfalls

### ❌ Fragile Base Class Problem

```csharp
public class BaseClass
{
    public void DoWork()
    {
        Step1();
        Step2();  // Added in new version
    }
    
    protected virtual void Step1() { }
    protected virtual void Step2() { }  // New method!
}

public class DerivedClass : BaseClass
{
    protected override void Step1()
    {
        base.Step1();
        // Assumed Step1 was called, then DoWork finished
        // But now Step2 is called after - breaking change!
    }
}
```

### ❌ Deep Hierarchies

```csharp
// DON'T DO THIS
class Entity -> AuditableEntity -> SoftDeletableEntity -> TenantEntity -> Order

// TOO DEEP: Hard to understand, hard to change
// Prefer: Composition with interfaces
class Order : Entity, IAuditable, ISoftDeletable, ITenant
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INHERITANCE BEST PRACTICES                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✅ DO:                                                             │
│     • Keep hierarchies shallow (max 2 levels)                       │
│     • Use "IS-A" test: Does Child truly IS-A Parent?                │
│     • Favor composition over inheritance                            │
│     • Design base classes for inheritance (document, seal if not)   │
│     • Follow Liskov Substitution Principle                            │
│     • Use abstract classes for base framework code                  │
│                                                                      │
│  ❌ DON'T:                                                          │
│     • Use inheritance just for code reuse ("HAS-A" vs "IS-A")       │
│     • Create deep inheritance hierarchies                             │
│     • Break encapsulation in derived classes                        │
│     • Assume base class implementation details                      │
│     • Use 'new' keyword without understanding implications          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: Why does C# not support multiple inheritance?**
> To avoid the "diamond problem" - ambiguity when two base classes have the same method. Interfaces provide similar capability without this issue.

**Q: What's the difference between `virtual` and `abstract`?**
> `virtual` has a default implementation; `abstract` has no implementation and MUST be overridden.

**Q: What happens if you don't call `base()` in a derived constructor?**> The compiler automatically inserts a call to the parameterless base constructor. If none exists, compile error.

**Q: When should a class be `sealed`?**> When it's not designed for inheritance, or when security/reliability requires preventing derivation (e.g., `string` class).

**Q: What's method hiding with `new`?**> It creates a completely separate method that happens to share a name. No polymorphic dispatch - the parent's version is still called through base references.
