# Events, Delegates, and Observer Pattern in C#

## Table of Contents
1. [Delegates Fundamentals](#delegates-fundamentals)
2. [Events in C#](#events-in-c)
3. [Anonymous Methods and Lambdas](#anonymous-methods-and-lambdas)
4. [Observer Pattern Implementation](#observer-pattern-implementation)
5. [Publish/Subscribe Model](#publishsubscribe-model)

---

## Delegates Fundamentals

### What is a Delegate?

A delegate is a **type-safe function pointer** — it holds a reference to a method and can call that method indirectly.

### Memory Representation

```
┌─────────────────────────────────────────────────────────────────┐
│                    DELEGATE MEMORY LAYOUT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HEAP:                                                          │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Delegate Object                                        │    │
│   │  ├─ _target: 0x7FFE1000  (object reference or null)      │    │
│   │  ├─ _methodPtr: 0x7FFE5000 (method address)            │    │
│   │  └─ _invocationList: null (for multicast)              │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   When invoked:                                                  │
│   1. Load _target (instance or null for static)                 │
│   2. Jump to _methodPtr                                          │
│   3. Execute method                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Delegate Example

```csharp
// 1. Declare the delegate type
public delegate void NotificationHandler(string message);

// 2. Methods that match the signature
public class EmailService
{
    public void SendEmail(string message)
    {
        Console.WriteLine($"📧 Email: {message}");
    }
}

public class SmsService
{
    public void SendSms(string message)
    {
        Console.WriteLine($"📱 SMS: {message}");
    }
}

// 3. Usage
public class Program
{
    public static void Main()
    {
        // Create delegate instance pointing to a method
        NotificationHandler notifier = new EmailService().SendEmail;
        
        // Invoke the delegate
        notifier("Hello!");  // Output: 📧 Email: Hello!
        
        // Multicast - chain multiple methods
        notifier += new SmsService().SendSms;
        notifier("Important!");  // Calls BOTH methods!
    }
}
```

### Built-in Generic Delegates

C# provides generic delegates so you don't need to declare your own:

```csharp
// Action<T> - returns void, takes 0-16 parameters
Action<string> log = Console.WriteLine;
log("Hello");

Action<int, int> printSum = (a, b) => Console.WriteLine(a + b);
printSum(3, 5);  // Output: 8

// Func<T, TResult> - returns TResult, takes 0-16 parameters
Func<int, int, int> add = (a, b) => a + b;
int result = add(3, 5);  // result = 8

// Predicate<T> - returns bool, takes 1 parameter
Predicate<int> isEven = n => n % 2 == 0;
bool result = isEven(4);  // true
```

---

## Events in C#

### What is an Event?

An event is a **specialized delegate** that provides encapsulation:
- Only the declaring class can invoke (publish) the event
- Other classes can only subscribe/unsubscribe (+= and -=)

### Memory Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT VS DELEGATE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Delegate (public):                                             │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  public NotificationHandler OnNotify;                     │    │
│   │                                                         │    │
│   │  ❌ Anyone can invoke: obj.OnNotify("x")                │    │
│   │  ❌ Anyone can overwrite: obj.OnNotify = otherMethod    │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│   Event (controlled):                                            │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  public event NotificationHandler OnNotify;             │    │
│   │                                                         │    │
│   │  ✓ Others can only: obj.OnNotify += handler             │    │
│   │  ✓ Only declaring class can: OnNotify?.Invoke("x")     │    │
│   │  ❌ Cannot overwrite with =                            │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Complete Event Example

```csharp
// EventArgs - data passed with event
public class NotificationEventArgs : EventArgs
{
    public string Title { get; set; }
    public string Message { get; set; }
    public DateTime Timestamp { get; set; }
    
    public NotificationEventArgs(string title, string message)
    {
        Title = title;
        Message = message;
        Timestamp = DateTime.Now;
    }
}

// Publisher - declares and raises events
public class NotificationPublisher
{
    // Event declaration using EventHandler<T>
    public event EventHandler<NotificationEventArgs> OnNotificationSent;
    
    // Alternative: custom delegate
    // public event NotificationHandler OnNotificationSent;
    
    public void SendNotification(string title, string message)
    {
        // ... send logic ...
        
        // Raise event (null-conditional invocation)
        OnNotificationSent?.Invoke(this, new NotificationEventArgs(title, message));
    }
}

// Subscriber - listens to events
public class StudentSubscriber
{
    public string Name { get; set; }
    
    public StudentSubscriber(string name)
    {
        Name = name;
    }
    
    // Event handler method
    public void OnNotificationReceived(object sender, NotificationEventArgs e)
    {
        Console.WriteLine($"[{Name}] Received: {e.Title} - {e.Message} at {e.Timestamp:HH:mm:ss}");
    }
}

// Usage
public class Program
{
    public static void Main()
    {
        var publisher = new NotificationPublisher();
        var student1 = new StudentSubscriber("Ahmed");
        var student2 = new StudentSubscriber("Sara");
        
        // Subscribe (+=)
        publisher.OnNotificationSent += student1.OnNotificationReceived;
        publisher.OnNotificationSent += student2.OnNotificationReceived;
        
        // Publish event
        publisher.SendNotification("New Grade", "Your exam results are available");
        
        // Output:
        // [Ahmed] Received: New Grade - Your exam results are available at 14:30:15
        // [Sara] Received: New Grade - Your exam results are available at 14:30:15
        
        // Unsubscribe (-=)
        publisher.OnNotificationSent -= student1.OnNotificationReceived;
    }
}
```

---

## Anonymous Methods and Lambdas

### Evolution of Event Handlers

```csharp
// 1. Named method (C# 1.0)
button.Click += Button_Click;
void Button_Click(object sender, EventArgs e) { }

// 2. Anonymous method (C# 2.0)
button.Click += delegate(object sender, EventArgs e) 
{ 
    Console.WriteLine("Clicked!"); 
};

// 3. Lambda expression (C# 3.0+)
button.Click += (sender, e) => Console.WriteLine("Clicked!");

// With multiple statements
button.Click += (sender, e) => 
{
    Console.WriteLine("Clicked!");
    SaveData();
    RefreshUI();
};
```

### Closure and Captured Variables

```csharp
public class CounterExample
{
    public void DemonstrateClosure()
    {
        int count = 0;  // Local variable
        
        // Lambda captures 'count' by reference
        Action increment = () => 
        {
            count++;
            Console.WriteLine($"Count: {count}");
        };
        
        increment();  // Count: 1
        increment();  // Count: 2
        increment();  // Count: 3
        
        // 'count' is captured and persists!
    }
}

// Memory visualization:
// The compiler generates a class to hold captured variables
public class <>c__DisplayClass
{
    public int count;
    public void <DemonstrateClosure>b__0()
    {
        count++;
        Console.WriteLine($"Count: {count}");
    }
}
```

---

## Observer Pattern Implementation

### Pattern Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVER PATTERN                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌─────────────────┐                         │
│                      │    ISubject     │                         │
│                      │  (Observable)   │                         │
│                      │  ─────────────  │                         │
│                      │  + Attach()      │                         │
│                      │  + Detach()      │                         │
│                      │  + Notify()      │                         │
│                      └────────┬────────┘                         │
│                               │                                  │
│                    ┌──────────┴──────────┐                       │
│                    │                       │                       │
│           ┌────────▼────────┐    ┌────────▼────────┐              │
│           │  ConcreteSubject│    │    IObserver    │              │
│           │  ───────────────│    │  ───────────────│              │
│           │  - observers[]   │◄───│  + Update()     │              │
│           │  - state         │    │                 │              │
│           │  + GetState()    │    └────────┬────────┘              │
│           │  + SetState()    │             │                      │
│           └───────────────────┘    ┌──────────┴──────────┐             │
│                                  │                    │             │
│                         ┌────────▼────────┐   ┌────────▼────────┐    │
│                         │ConcreteObserverA│   │ConcreteObserverB│    │
│                         │────────────────│   │────────────────│    │
│                         │  + Update()     │   │  + Update()     │    │
│                         │  - subject      │   │  - subject      │    │
│                         └─────────────────┘   └─────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Complete Implementation

```csharp
// === INTERFACES ===

// The Observer interface
public interface IObserver
{
    void Update(string notificationType, string message);
}

// The Subject interface
public interface ISubject
{
    void Attach(IObserver observer);
    void Detach(IObserver observer);
    void Notify(string notificationType, string message);
}

// === CONCRETE IMPLEMENTATIONS ===

// Concrete Subject - maintains state and notifies observers
public class CourseChannel : ISubject
{
    private readonly List<IObserver> _observers = new();
    private readonly string _courseName;
    
    public CourseChannel(string courseName)
    {
        _courseName = courseName;
    }
    
    // Subscribe an observer
    public void Attach(IObserver observer)
    {
        Console.WriteLine($"📥 {((Student)observer).Name} subscribed to {_courseName}");
        _observers.Add(observer);
    }
    
    // Unsubscribe an observer
    public void Detach(IObserver observer)
    {
        Console.WriteLine($"📤 {((Student)observer).Name} unsubscribed from {_courseName}");
        _observers.Remove(observer);
    }
    
    // Notify all observers
    public void Notify(string notificationType, string message)
    {
        Console.WriteLine($"\n🔔 Broadcasting to {_observers.Count} students in {_courseName}:");
        foreach (var observer in _observers.ToList())  // ToList for safe iteration
        {
            observer.Update(notificationType, $"[{_courseName}] {message}");
        }
    }
    
    // Business methods that trigger notifications
    public void PostAnnouncement(string announcement)
    {
        Notify("Announcement", announcement);
    }
    
    public void PostGrade(string studentName, string grade)
    {
        Notify("Grade", $"{studentName}'s grade: {grade}");
    }
    
    public void PostAssignment(string assignmentName, DateTime dueDate)
    {
        Notify("Assignment", $"New assignment '{assignmentName}' due {dueDate:MMM dd}");
    }
}

// Concrete Observer - receives notifications
public class Student : IObserver
{
    public string Name { get; }
    private readonly List<string> _notifications = new();
    
    public Student(string name)
    {
        Name = name;
    }
    
    // Called when Subject notifies
    public void Update(string notificationType, string message)
    {
        var notification = $"[{notificationType}] {message}";
        _notifications.Add(notification);
        Console.WriteLine($"  📨 {Name} received: {notification}");
    }
    
    public void ShowNotificationHistory()
    {
        Console.WriteLine($"\n📋 {Name}'s notification history:");
        foreach (var n in _notifications)
        {
            Console.WriteLine($"   • {n}");
        }
    }
}

// === USAGE ===
public class Program
{
    public static void Main()
    {
        // Create the subject (observable)
        var csharpCourse = new CourseChannel("C# Advanced");
        
        // Create observers
        var ahmed = new Student("Ahmed");
        var sara = new Student("Sara");
        var omar = new Student("Omar");
        
        // Subscribe students
        csharpCourse.Attach(ahmed);
        csharpCourse.Attach(sara);
        csharpCourse.Attach(omar);
        
        // Post content - all subscribers get notified
        csharpCourse.PostAnnouncement("Class will start at 2 PM tomorrow");
        csharpCourse.PostAssignment("Generic Repository", DateTime.Now.AddDays(7));
        
        // Sara unsubscribes
        csharpCourse.Detach(sara);
        
        // Only Ahmed and Omar receive this
        csharpCourse.PostGrade("Ahmed", "A+");
        
        // Check individual notification histories
        ahmed.ShowNotificationHistory();
        sara.ShowNotificationHistory();
    }
}

// Output:
// 📥 Ahmed subscribed to C# Advanced
// 📥 Sara subscribed to C# Advanced
// 📥 Omar subscribed to C# Advanced
// 
// 🔔 Broadcasting to 3 students in C# Advanced:
//   📨 Ahmed received: [Announcement] [C# Advanced] Class will start at 2 PM tomorrow
//   📨 Sara received: [Announcement] [C# Advanced] Class will start at 2 PM tomorrow
//   📨 Omar received: [Announcement] [C# Advanced] Class will start at 2 PM tomorrow
//
// 🔔 Broadcasting to 3 students in C# Advanced:
//   📨 Ahmed received: [Assignment] [C# Advanced] New assignment 'Generic Repository' due Apr 12
//   ...
// 📤 Sara unsubscribed from C# Advanced
//
// 🔔 Broadcasting to 2 students in C# Advanced:
//   📨 Ahmed received: [Grade] [C# Advanced] Ahmed's grade: A+
//   📨 Omar received: [Grade] [C# Advanced] Ahmed's grade: A+
```

---

## Publish/Subscribe Model

### Differences from Observer Pattern

| Observer Pattern | Publish/Subscribe |
|------------------|-------------------|
| Subject knows observers directly | Publisher doesn't know subscribers |
| Synchronous notification | Often asynchronous |
| Direct method calls | Usually via event bus/mediator |
| Tightly coupled | Loosely coupled |

### Simple Pub/Sub Implementation

```csharp
// Message/Event that gets published
public class MessageEvent
{
    public string Topic { get; set; }
    public string Payload { get; set; }
    public DateTime Timestamp { get; set; }
}

// Event Bus (mediator between publishers and subscribers)
public class EventBus
{
    private readonly Dictionary<string, List<Action<MessageEvent>>> _subscriptions = new();
    
    // Subscribe to a topic
    public void Subscribe(string topic, Action<MessageEvent> handler)
    {
        if (!_subscriptions.ContainsKey(topic))
            _subscriptions[topic] = new List<Action<MessageEvent>>();
        
        _subscriptions[topic].Add(handler);
    }
    
    // Unsubscribe from a topic
    public void Unsubscribe(string topic, Action<MessageEvent> handler)
    {
        if (_subscriptions.ContainsKey(topic))
            _subscriptions[topic].Remove(handler);
    }
    
    // Publish to a topic
    public void Publish(string topic, string payload)
    {
        if (!_subscriptions.ContainsKey(topic))
        {
            Console.WriteLine($"⚠️ No subscribers for topic: {topic}");
            return;
        }
        
        var message = new MessageEvent
        {
            Topic = topic,
            Payload = payload,
            Timestamp = DateTime.Now
        };
        
        foreach (var handler in _subscriptions[topic].ToList())
        {
            handler.Invoke(message);  // Could be async in real implementation
        }
    }
}

// Usage
public class Program
{
    public static void Main()
    {
        var eventBus = new EventBus();
        
        // Subscriber 1: listens to "orders" topic
        eventBus.Subscribe("orders", msg => 
            Console.WriteLine($"🛒 Order Service: {msg.Payload}"));
        
        // Subscriber 2: listens to "orders" topic
        eventBus.Subscribe("orders", msg => 
            Console.WriteLine($"📧 Email Service: Order notification sent"));
        
        // Subscriber 3: listens to "inventory" topic
        eventBus.Subscribe("inventory", msg => 
            Console.WriteLine($"📦 Inventory Service: {msg.Payload}"));
        
        // Publisher sends messages
        Console.WriteLine("Publishing to 'orders':");
        eventBus.Publish("orders", "New order #12345 received");
        
        Console.WriteLine("\nPublishing to 'inventory':");
        eventBus.Publish("inventory", "Stock updated for SKU-ABC");
        
        Console.WriteLine("\nPublishing to 'payments':");
        eventBus.Publish("payments", "Payment processed");  // No subscribers!
    }
}

// Output:
// Publishing to 'orders':
// 🛒 Order Service: New order #12345 received
// 📧 Email Service: Order notification sent
//
// Publishing to 'inventory':
// 📦 Inventory Service: Stock updated for SKU-ABC
//
// Publishing to 'payments':
// ⚠️ No subscribers for topic: payments
```

---

## Event-Driven Architecture

### Key Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌─────────────┐    Event    ┌─────────────┐    Event    ┌────────┐│
│   │  Producer   │ ───────────>│  Event Bus  │ ───────────>│Consumer││
│   │  (Service A)│             │  (Broker)   │             │(Svc B) ││
│   └─────────────┘             └─────────────┘             └────────┘│
│          │                           │                           │   │
│          │                           │                           │   │
│   ┌─────────────┐             ┌─────────────┐             ┌────────┐│
│   │  Producer   │ ───────────>│  Event Bus  │ ───────────>│Consumer││
│   │  (Service C)│             │             │             │(Svc D) ││
│   └─────────────┘             └─────────────┘             └────────┘│
│                                                                      │
│   Benefits:                                                          │
│   • Loose coupling - services don't know about each other           │
│   • Scalability - add consumers without changing producers          │
│   • Resilience - events can be queued and replayed                  │
│   • Real-time - immediate notification of changes                   │
│                                                                      │
│   Common Implementations:                                            │
│   • RabbitMQ, Kafka, Azure Service Bus, AWS SNS/SQS                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q: What's the difference between a delegate and an event?**
> An event is a delegate with restricted access - only the declaring class can invoke it (publish), while others can only subscribe/unsubscribe. This prevents external code from bypassing business logic.

**Q: What is the Observer pattern?**
> A behavioral pattern where an object (Subject) maintains a list of dependents (Observers) and notifies them automatically of any state changes. Enables loose coupling between components.

**Q: When should you use EventHandler<T> vs custom delegates?**
> Use EventHandler<T> for standard events following .NET conventions. Use custom delegates when you need different parameter signatures or specific return types.

**Q: How does the `?.` operator work with event invocation?**
> `OnEvent?.Invoke()` is null-conditional - it only invokes if the event has subscribers (is not null). Without it, you'd get NullReferenceException on events with no subscribers.

**Q: What's a memory leak risk with events?**
> If an observer subscribes to an event but never unsubscribes, the subject keeps a reference to the observer, preventing garbage collection. Always unsubscribe (-=) or use weak event patterns.
