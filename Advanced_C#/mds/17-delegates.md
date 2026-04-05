# Delegates in C#

## 🧠 What Is a Delegate?

A **delegate** is a type that holds a reference to a method — similar to a function pointer in C/C++, but **type-safe** and **object-oriented**. Think of it as a variable that stores a *method* instead of data.

> **Simple mental model:** A delegate is a typed pointer to a method. When you declare `delegate int MathOp(int a, int b)`, you're saying: *"I want a variable that can hold any method taking two ints and returning an int."*

---

## 🌍 Real-World Analogy

Imagine a **TV remote**. The remote doesn't care what TV it's pointing at — it just knows it can press "volume up." A delegate is that remote: it defines what the *call looks like* without caring which *method* it calls. You can swap the TV (the actual method) any time.

---

## 🔑 Core Properties

| Property | Description |
|---|---|
| **Type-Safe** | The delegate signature must exactly match the method's return type and parameters |
| **First-Class** | Methods can be passed as arguments, returned from methods, or stored in variables |
| **Multicast-Ready** | One delegate can chain multiple methods and invoke them all at once |

---

## 💻 Declaring & Using a Delegate — 4 Steps

```csharp
// ── STEP 1: Declare the delegate type ──────────────────────────
public delegate int MathOperation(int a, int b);

// ── STEP 2: Create methods that match the signature ────────────
public static class Calculator
{
    public static int Add(int a, int b)      => a + b;
    public static int Subtract(int a, int b) => a - b;
    public static int Multiply(int a, int b) => a * b;
}

// ── STEP 3: Assign the method to a delegate variable ───────────
MathOperation operation = Calculator.Add;
// 📦 HEAP: new MathOperation { _target=null, _methodPtr=→Add }
// 📦 STACK: 'operation' holds reference to heap object

// ── STEP 4: Invoke ─────────────────────────────────────────────
int result = operation(10, 5);         // result = 15
result     = operation.Invoke(10, 5);  // same thing — explicit form

// ── Reassign to a different method ─────────────────────────────
operation = Calculator.Subtract;
result = operation(10, 5); // result = 5
```

---

## ⚙️ Memory Layout

```
Stack                    Heap
─────────────────        ─────────────────────────────────────────
operation (ref) ──────►  MathOperation object
                         ├── _target       = null (static method)
                         ├── _methodPtr    = → Calculator.Add
                         └── _invocationList = null (single method)

// After reassignment:
operation (ref) ──────►  NEW MathOperation object (old one → GC ♻️)
                         ├── _methodPtr = → Calculator.Subtract
```

---

## 🌍 Why Delegates? The Open/Closed Problem

```csharp
// ❌ Hard-coded — every new channel requires modifying this method
void SendAlert(string msg, string channel)
{
    if (channel == "email") EmailNotifier.Send(msg);
    else if (channel == "sms") SmsNotifier.Send(msg);
    // New channel? MODIFY THIS CODE — violates OCP
}

// ✅ With delegates — caller decides the behavior
void SendAlert(string msg, Action<string> notify)
{
    notify(msg);
    // New channel? Just write a new method and pass it in. Done.
}

// Usage:
SendAlert("Server down!", EmailNotifier.Send);
SendAlert("Server down!", SmsNotifier.Send);
// Combine both — multicast (covered below)
```

| Problem | Hard-Coded | With Delegates |
|---|---|---|
| New channel | Modify existing code | Add new method only |
| Runtime decisions | Not possible | Assign different delegate |
| Combine behaviors | More if/else | Multicast `+=` operator |
| Testability | Hard to mock | Pass test double |
| OCP compliance | Violates Open/Closed | Fully open for extension |

---

## 📦 Multicast Delegates

A single delegate variable can hold **multiple methods**. They are all invoked in the order they were added.

```csharp
public delegate void NotifyDelegate(string message);

NotifyDelegate notify = EmailNotifier.Send;
notify += SmsNotifier.Send;   // chain a second method
notify += PushNotifier.Send;  // chain a third

notify("Server is down!"); // calls all three in order

// Remove from chain
notify -= SmsNotifier.Send;
notify("All clear!"); // calls Email + Push only
```

### ⚠️ Return Values in Multicast Delegates

```csharp
// Only the LAST method's return value is captured
// Previous return values are silently discarded
public delegate int Calculate(int x);

Calculate calc = Double;
calc += Triple;

int result = calc(5);
// Double(5) = 10 ← discarded
// Triple(5) = 15 ← this is what result gets
Console.WriteLine(result); // 15
```

### ⚠️ Null Delegate Bug

```csharp
// ❌ Throws NullReferenceException if no subscriber
NotifyDelegate? notify = null;
notify("Hello!"); // CRASH

// ✅ Safe null check
notify?.Invoke("Hello!"); // does nothing if null
```

---

## 🏗️ Where to Define a Delegate

```csharp
// ════════════════════════════════════════════════
// OPTION 1 — Namespace level (most common for shared delegates)
namespace MyApp.Notifications
{
    public delegate void NotifyDelegate(string message);   // public = any project
    internal delegate bool ValidationDelegate(string input); // internal = this assembly only
}

// ════════════════════════════════════════════════
// OPTION 2 — Nested inside a class (tightly coupled to one class)
public class OrderService
{
    public delegate void OrderHandler(string orderId);   // callers say: OrderService.OrderHandler
    private delegate bool InternalCheck(string orderId); // only used inside OrderService
}

// ════════════════════════════════════════════════
// OPTION 3 — Skip declaration entirely (use Action/Func)
void Process(Action<string> notify) { }       // no custom delegate needed
void Filter(Func<int, bool> predicate) { }   // use built-in
```

| Scenario | Location | Modifier |
|---|---|---|
| Used by multiple classes | Namespace level | `public` |
| Assembly-internal only | Namespace level | `internal` |
| Belongs to one class's API | Nested in class | `public` |
| Only used inside one class | Nested in class | `private` |
| Generic callback (no domain meaning) | Skip — use `Action`/`Func` | N/A |

---

## 🚀 The Road to Lambda: Anonymous Methods & Lambdas

Delegates evolved — you no longer need to declare a named method every time.

```csharp
public delegate int MathOp(int a, int b);

// ── Old way: named method ──────────────────────────────
public static int Add(int a, int b) => a + b;
MathOp op = Add;

// ── Anonymous method (C# 2.0) ─────────────────────────
MathOp op = delegate(int a, int b) { return a + b; };

// ── Lambda expression (C# 3.0+) ───────────────────────
MathOp op = (a, b) => a + b;   // most concise — preferred today
```

---

## 🔒 Closures & Variable Capture

Lambdas can capture variables from the surrounding scope. This is powerful but has memory implications.

```csharp
int multiplier = 3;

Func<int, int> triple = x => x * multiplier; // 'multiplier' is CAPTURED

Console.WriteLine(triple(5)); // 15

// ⚠️ The captured variable is SHARED — mutation affects the lambda
multiplier = 10;
Console.WriteLine(triple(5)); // 50 — not 15!
```

### Memory: How Closures Work

```
// The compiler generates a hidden class to hold captured variables:

// [Compiler-generated]
private sealed class <>c__DisplayClass
{
    public int multiplier;           // captured variable lives HERE on the heap
    public int <Main>b__0(int x) => x * multiplier;
}

// 'triple' delegate points to an instance of this class
// The instance lives on the heap and keeps 'multiplier' alive (no GC)
```

---

## 📦 Built-in Delegates: Action, Func, Predicate

You rarely need to declare custom delegates. .NET provides three generic ones for 99% of cases:

```csharp
// Action<T> — returns void, takes 0-16 params
Action<string>         print    = msg => Console.WriteLine(msg);
Action<int, int>       addPrint = (a, b) => Console.WriteLine(a + b);
Action                 greet    = () => Console.WriteLine("Hello!");

// Func<T, TResult> — returns a value, takes 0-16 params (last type = return type)
Func<int, int, int>    add      = (a, b) => a + b;
Func<string, int>      length   = s => s.Length;
Func<bool>             coinFlip = () => Random.Shared.Next(2) == 0;

// Predicate<T> — returns bool, takes one param (sugar for Func<T, bool>)
Predicate<int>         isEven   = n => n % 2 == 0;
Predicate<string>      notEmpty = s => !string.IsNullOrEmpty(s);
```

### Comparison Table

| Delegate | Return Type | Parameters | Example Use |
|---|---|---|---|
| `Action<T>` | `void` | 0–16 | Event handlers, side effects |
| `Func<T, TResult>` | Any type | 0–16 | Transformations, calculations |
| `Predicate<T>` | `bool` | Exactly 1 | Filters, conditions |
| Custom delegate | Any | Any | Domain-specific, named intent |

---

## 🆚 Delegates vs Interfaces

| Aspect | Delegate | Interface |
|---|---|---|
| What it represents | A single method | A contract (multiple methods) |
| Multicast | ✅ Chain multiple | ❌ |
| Overhead | Lower (one method call) | Slightly higher (vtable) |
| When to use | Single behavior callback | Full contract with multiple members |

---

## 🏷️ Naming Conventions

```csharp
// ✅ Suffix "Delegate" — generic callback concept
public delegate void NotifyDelegate(string message);
public delegate bool FilterDelegate(int number);

// ✅ Suffix "Handler" — event-style callbacks
public delegate void OrderPlacedHandler(string orderId);
public delegate void ErrorHandler(Exception ex);

// ✅ Suffix "Callback" — async or completion patterns
public delegate void CompletionCallback(bool success, string? error);
```

---

## 📌 Summary

> A **delegate** is a type-safe method reference that enables passing behavior as data. Use `Action`/`Func`/`Predicate` for generic callbacks — they cover 99% of use cases. Create custom delegates only when the name itself communicates domain intent. Delegates are the foundation of **LINQ**, **events**, **callbacks**, and **lambda expressions** in C#.
