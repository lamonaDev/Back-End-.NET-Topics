# Class Relationships in C#

## Overview

In Object-Oriented Programming, classes interact with each other through defined relationships. Understanding these relationships is crucial for designing maintainable, loosely-coupled systems.

There are **four main types of class relationships** (ordered from weakest to strongest):

1. **Dependency** (uses temporarily)
2. **Association** (has a reference)
3. **Aggregation** (has a collection, weak ownership)
4. **Composition** (has a collection, strong ownership)

---

## Comparison Table

| Relationship | UML Symbol | Lifetime | Creation | Strength |
|-------------|-----------|----------|----------|----------|
| **Dependency** | `---▷` (dashed) | Method scope | Passed as param | Weakest |
| **Association** | `───▷` (solid) | Independent | Passed in | Weak |
| **Aggregation** | `◇───` (white diamond) | Child outlives parent | Passed in | Medium |
| **Composition** | `◆───` (black diamond) | Child dies with parent | Created inside | Strong |

---

## 1. Dependency (Weakest)

### Definition
**Class A depends on Class B if changes to B might affect A.** This is the most temporary relationship — B is used only within a method.

### Real-World Analogy
**Going to the doctor:**
- You depend on the doctor during the visit
- You don't keep the doctor after the visit
- The doctor exists independently

### Code Example

```csharp
public class EmailService
{
    // No field for SmtpClient - temporary usage
    public void SendEmail(string to, string body, SmtpClient smtpClient)
    {
        // Dependency: SmtpClient passed as parameter
        // Exists only during this method call
        var message = new MailMessage("from@example.com", to, "Subject", body);
        smtpClient.Send(message);
    }  // After this, EmailService has no reference to SmtpClient
}

// Usage
var emailService = new EmailService();
var smtp = new SmtpClient("smtp.gmail.com", 587);
emailService.SendEmail("user@example.com", "Hello", smtp);
// smtp can be reused elsewhere, passed to other methods
```

### Memory Allocation

```
STACK during SendEmail call:
┌─────────────────────────────────────────┐
│  Method Frame                            │
│  ├─ to: "user@example.com"             │
│  ├─ body: "Hello"                       │
│  ├─ smtpClient: 0x7FFE1000  ◄── Reference│
│  └─ message: 0x7FFE2000                  │
└─────────────────────────────────────────┘

HEAP:
┌─────────────────────────────────────────┐
│  SmtpClient @ 0x7FFE1000                │
│  ├─ [fields]                            │
│  └─ ...                                 │
│                                         │
│  EmailService has NO field for SmtpClient│
│  Relationship lasts only during method   │
└─────────────────────────────────────────┘
```

---

## 2. Association (Permanent Reference)

### Definition
**Class A is associated with Class B if A holds a permanent reference to B as a field or property.**

### Real-World Analogy
**Employee and ID Card:**
- An employee always HAS an ID card
- The ID card exists independently (was created by HR)
- If the employee is fired, the ID card still exists (can be reassigned)

### Code Example

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
}

public class Order
{
    public int OrderId { get; set; }
    
    // Association: Order always has a Customer
    // Customer was created OUTSIDE and passed in
    public Customer Customer { get; set; }
    
    public void DisplayInfo()
    {
        Console.WriteLine($"Order {OrderId} by {Customer.Name}");
    }
}

// Usage
var customer = new Customer { Id = 1, Name = "Alice" };  // Created independently
var order = new Order { OrderId = 100, Customer = customer };  // Just a reference

// Delete order → customer still exists
order = null;
Console.WriteLine(customer.Name);  // "Alice" - still alive!
```

### Memory Allocation

```
HEAP:
┌─────────────────────────────────────────┐
│  Order @ 0x7FFE1000                     │
│  ├─ OrderId: 100                        │
│  ├─ Customer: 0x7FFE2000  ◄── REFERENCE │
│  └─ ...                                 │
│                                         │
│  Customer @ 0x7FFE2000                  │
│  ├─ Id: 1                               │
│  ├─ Name: "Alice"                       │
│  └─ ...                                 │
│                                         │
│  Both objects exist independently        │
│  Order just "knows about" Customer       │
└─────────────────────────────────────────┘
```

---

## 3. Aggregation (Weak Ownership, Collection)

### Definition
**Class A aggregates Class B if A groups multiple B instances that were created OUTSIDE and can exist WITHOUT A.**

### Real-World Analogy
**University and Students:**
- A university HAS many students
- Students exist before joining the university
- If the university closes, students continue to exist (go to other schools)
- Students can belong to multiple organizations (clubs, teams)

### Code Example

```csharp
public class Player
{
    public string Name { get; set; }
    public int SkillLevel { get; set; }
}

public class Team
{
    public string TeamName { get; set; }
    
    // Aggregation: Team has Players, but didn't create them
    // Players were created outside and passed in
    public List<Player> Players { get; set; }
    
    public Team(string teamName, List<Player> players)  // ◄── Passed from outside
    {
        TeamName = teamName;
        Players = players;
    }
    
    public void DisplayRoster()
    {
        Console.WriteLine($"{TeamName} Roster:");
        Players.ForEach(p => Console.WriteLine($"  - {p.Name}"));
    }
}

// Usage
var players = new List<Player>  // Players created independently
{
    new Player { Name = "Alice", SkillLevel = 90 },
    new Player { Name = "Bob", SkillLevel = 85 },
    new Player { Name = "Charlie", SkillLevel = 88 }
};

var team = new Team("Eagles", players);

// Team disbands → players still exist!
team = null;
players.ForEach(p => Console.WriteLine(p.Name));  // All still alive!

// Players can join another team
var newTeam = new Team("Hawks", players);  // Same players, different team
```

### Memory Allocation

```
HEAP:
┌─────────────────────────────────────────┐
│  Team @ 0x7FFE1000                      │
│  ├─ TeamName: "Eagles"                   │
│  ├─ Players: 0x7FFE3000  ◄── List ref   │
│  └─ ...                                 │
│                                         │
│  List<Player> @ 0x7FFE3000              │
│  ├─ [0]: 0x7FFE2000  ◄── Player ref    │
│  ├─ [1]: 0x7FFE2500                   │
│  └─ [2]: 0x7FFE2800                   │
│                                         │
│  Player @ 0x7FFE2000                    │
│  ├─ Name: "Alice"                       │
│  └─ SkillLevel: 90                      │
│                                         │
│  Player @ 0x7FFE2500                    │
│  ├─ Name: "Bob"                         │
│  └─ ...                                 │
│                                         │
│  ◄── Players exist INDEPENDENTLY       │
│  ◄── Can outlive Team, join other teams │
└─────────────────────────────────────────┘
```

---

## 4. Composition (Strong Ownership)

### Definition
**Class A is composed of Class B if A CREATES B internally and B CANNOT exist without A.**

### Real-World Analogy
**House and Rooms:**
- A house IS COMPOSED OF rooms
- Rooms are created WITH the house (built together)
- If the house is demolished, the rooms cease to exist
- A room has no meaning without its house ("Living Room of House #42")

### Code Example

```csharp
public class Room
{
    public string Name { get; }
    public double Area { get; }
    
    // Internal constructor - can only be created from House
    internal Room(string name, double area)
    {
        Name = name;
        Area = area;
    }
}

public class House
{
    public string Address { get; set; }
    
    // Private list - only House can create/modify rooms
    private readonly List<Room> _rooms = new();
    
    // Public read-only access
    public IReadOnlyCollection<Room> Rooms => _rooms;
    
    public House(string address)
    {
        Address = address;
        
        // Composition: Rooms created INSIDE House
        _rooms.Add(new Room("Living Room", 400));
        _rooms.Add(new Room("Kitchen", 200));
        _rooms.Add(new Room("Bedroom", 350));
    }
    
    // Only way to add rooms - controlled by House
    public void AddRoom(string name, double area)
    {
        _rooms.Add(new Room(name, area));
    }
    
    public double TotalArea => _rooms.Sum(r => r.Area);
}

// Usage
var house = new House("123 Main St");

// Can read rooms
foreach (var room in house.Rooms)
{
    Console.WriteLine($"{room.Name}: {room.Area} sqft");
}

// Cannot create room independently
// var room = new Room("Garage", 500);  // ❌ COMPILE ERROR - internal constructor

// When house is destroyed, rooms are too
house = null;  // All 3 Room objects become eligible for GC
```

### Memory Allocation

```
HEAP:
┌─────────────────────────────────────────┐
│  House @ 0x7FFE1000                     │
│  ├─ Address: "123 Main St"             │
│  ├─ _rooms: 0x7FFE3000  ◄── List ref   │
│  └─ ...                                 │
│                                         │
│  List<Room> @ 0x7FFE3000               │
│  ├─ [0]: 0x7FFE4000  ◄── Room created   │
│  ├─ [1]: 0x7FFE4200      WITH House    │
│  └─ [2]: 0x7FFE4400                    │
│                                         │
│  Room @ 0x7FFE4000                      │
│  ├─ Name: "Living Room"                │
│  ├─ Area: 400                          │
│  └─ ...                                 │
│                                         │
│  Room @ 0x7FFE4200                      │
│  ├─ Name: "Kitchen"                    │
│  └─ ...                                 │
│                                         │
│  ◄── Rooms have NO independent life    │
│  ◄── Constructor prevents external     │
│      creation - enforced encapsulation   │
└─────────────────────────────────────────┘
```

---

## Relationship Decision Flowchart

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IDENTIFYING RELATIONSHIPS                          │
└─────────────────────────────────────────────────────────────────────┘

Is it stored as a field/property?
    │
    ├── NO → It's DEPENDENCY (temporary, method parameter)
    │
    └── YES → Continue...

Was it created with "new" INSIDE the class?
    │
    ├── YES → It's COMPOSITION (strong ownership)
    │
    └── NO → Continue...

Is it a collection?
    │
    ├── YES → It's AGGREGATION (weak ownership, shared)
    │
    └── NO → It's ASSOCIATION (single reference)
```

---

## Practical Examples in Entity Framework

```csharp
// COMPOSITION: Order and OrderItem
// OrderItem cannot exist without Order
// Cascade delete: delete Order → delete OrderItems
public class Order
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; } = new();  // Composition
}

// AGGREGATION: Course and Student
// Student exists independently, can be in multiple courses
public class Course
{
    public int Id { get; set; }
    public List<Student> Students { get; set; } = new();  // Aggregation
}

// ASSOCIATION: Employee and Department
// Employee has one Department, Department has many Employees
public class Employee
{
    public int Id { get; set; }
    public Department Department { get; set; }  // Association
}

// DEPENDENCY: Service using Repository within method
public class OrderService
{
    public void ProcessOrder(int orderId, IEmailService emailService)  // Dependency
    {
        // Use emailService temporarily
        emailService.SendConfirmation(orderId);
    }
}
```

---

## Common Mistakes

### ❌ Breaking Composition Rules

```csharp
// WRONG: Allowing external creation
public class Order
{
    public List<OrderItem> Items { get; set; }  // Public setter!
}

// External code can now break invariants:
order.Items = null;  // Breaks encapsulation!
order.Items.Add(new OrderItem());  // Bypasses Order's business logic
```

### ✅ Proper Composition

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items;
    
    public void AddItem(Product product, int quantity)
    {
        // Controlled creation within Order
        _items.Add(new OrderItem(product, quantity, this));
    }
}
```

---

## Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RELATIONSHIP STRENGTH                            │
│                    (Weakest → Strongest)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DEPENDENCY        ASSOCIATION        AGGREGATION     COMPOSITION    │
│  ──────────        ──────────        ──────────     ───────────    │
│                                                                      │
│  "I use you        "I know you"       "I group you"  "I own you"    │
│   briefly"                                                         │
│                                                                      │
│  • Method param    • Field ref        • Collection   • Collection    │
│  • Temporary       • Independent      • Passed in   • Created in   │
│  • No ownership    • No ownership     • Shared       • Exclusive     │
│                                                                      │
│  Lifetime:         Lifetime:          Lifetime:      Lifetime:      │
│  Method scope      Independent        Child >       Child =        │
│                                      Parent         Parent         │
│                                                                      │
│  Example:          Example:           Example:     Example:        │
│  Service using     Order has          Team has       House has       │
│  Repository        Customer           Players        Rooms           │
│  (temporarily)     (reference)        (weak)         (strong)        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: What's the difference between aggregation and composition?**
> Aggregation: Parts exist independently, can be shared. Composition: Parts created by parent, die with parent, exclusive ownership.

**Q: Can a class have both aggregation and composition relationships?**
> Yes. A University aggregates Students (composition with Departments), aggregates Courses (weak), and is associated with its President (single reference).

**Q: How do you enforce composition in C#?**
> Make child constructor internal or private, only create instances within parent, expose read-only collections, implement IDisposable if cleanup needed.

**Q: What's the UML notation for composition vs aggregation?**
> Composition: filled black diamond (◆). Aggregation: empty white diamond (◇). Both on the "whole" side of the relationship.

**Q: In EF Core, how does composition translate to database?**
> Composition often uses cascade delete (DeleteBehavior.Cascade). Aggregation uses simple foreign keys without cascade.
