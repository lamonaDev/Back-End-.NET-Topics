# C# Language Fundamentals

## Table of Contents
1. [var vs dynamic vs object](#var-vs-dynamic-vs-object)
2. [const vs readonly](#const-vs-readonly)
3. [nameof Operator](#nameof-operator)
4. [string[] args in Main](#string-args-in-main)

---

## var vs dynamic vs object

### Overview
Three ways to declare variables with different type safety guarantees and compile-time/runtime behavior.

### var - Implicit Typing

**When to Use:**
- Type is obvious from right side
- Anonymous types
- LINQ queries

**Code Example:**
```csharp
// var is resolved at compile time
var message = "Hello";      // Compiler infers: string
var number = 42;            // Compiler infers: int
var today = DateTime.Now;   // Compiler infers: DateTime

// var with complex types
var users = new List<User>();
var query = from u in users
              where u.Age > 18
              select new { u.Name, u.Email };  // Anonymous type

// Error: must initialize
var x;                      // ❌ Error: Implicitly-typed variables must be initialized

// Error: cannot change type
var count = 10;
count = "ten";              // ❌ Error: Cannot convert string to int
```

**Memory Visualization:**
```
Compile-time type resolution:
┌─────────────────────────────────────────────┐
│ Source: var message = "Hello";                │
├─────────────────────────────────────────────┤
│ Compiler sees:                              │
│   - Right side: string literal              │
│   - Infers type: string                      │
│   - IL code: string message = "Hello";      │
└─────────────────────────────────────────────┘

Stack at runtime:
┌──────────────┐
│ message      │ ──→ "Hello" (in heap)
│ (string type)│
└──────────────┘
```

### dynamic - Runtime Binding

**When to Use:**
- COM interop
- Dynamic languages (IronPython)
- Reflection scenarios
- When compile-time type is unknown

**Code Example:**
```csharp
// dynamic resolved at runtime
dynamic value = 10;
Console.WriteLine(value.ToString());  // Works

value = "Hello";
Console.WriteLine(value.ToUpper());   // Works - resolved at runtime

value = new DateTime(2024, 1, 1);
Console.WriteLine(value.Year);        // Works

// Runtime error (not compile-time)
dynamic x = 5;
Console.WriteLine(x.NonExistentMethod());  // ❌ RuntimeBinderException

// COM interop example
dynamic excel = Activator.CreateInstance(
    Type.GetTypeFromProgID("Excel.Application"));
excel.Visible = true;
excel.Workbooks.Add();

// JSON deserialization (Newtonsoft.Json)
dynamic json = JsonConvert.DeserializeObject(@"{ ""name"": ""John"", ""age"": 30 }");
Console.WriteLine(json.name);  // "John"
```

**Memory Visualization:**
```
Runtime resolution:
┌─────────────────────────────────────────────┐
│ Source: dynamic value = 10;                  │
│ value = "Hello";                             │
├─────────────────────────────────────────────┤
│ Compile-time:                                 │
│   - Stored as object with runtime binder    │
│   - No IntelliSense                          │
│                                               │
│ Runtime:                                      │
│   - Value stored as boxed object             │
│   - DLR (Dynamic Language Runtime) resolves  │
│   - Late binding: method lookup at call time  │
└─────────────────────────────────────────────┘

Heap (boxed):
┌────────────────────┐
│ RuntimeCache       │ ──→ Type: int, then string
│ Method dispatch    │
└────────────────────┘
```

### object - Base Type

**When to Use:**
- When any type is acceptable
- Boxing value types
- Generic constraints
- Inheritance hierarchies

**Code Example:**
```csharp
// object is compile-time type with boxing
object anything = 42;       // Boxed int
object text = "Hello";      // Reference type (no boxing)
object date = DateTime.Now; // Boxed DateTime

// Unboxing required
int num = (int)anything;    // Unboxing
string str = (string)text;  // Cast

// Useful for heterogeneous collections
object[] mixed = new object[] { 1, "two", 3.0, DateTime.Now };
foreach (object item in mixed)
{
    Console.WriteLine(item.GetType().Name);
}
// Output: Int32, String, Double, DateTime
```

**Memory Visualization:**
```
Boxing behavior:
┌─────────────────────────────────────────┐
│ int value = 42; (value type on stack)  │
├─────────────────────────────────────────┤
│ object boxed = value;                   │
│   ↓                                     │
│ Heap allocation:                        │
│ ┌──────────────────────┐                │
│ │ Object header        │                │
│ │ Method table ptr     │ ──→ System.Int32
│ │ Value: 42            │                │
│ └──────────────────────┘                │
│   ↑                                     │
│ Reference on stack points to heap       │
└─────────────────────────────────────────┘
```

### Comparison Table

| Feature | var | dynamic | object |
|---------|-----|---------|--------|
| **Type check** | Compile-time | Runtime | Compile-time |
| **IntelliSense** | Yes | No | Limited |
| **Performance** | Fast | Slower (DLR) | Boxing cost |
| **Errors** | Compile-time | Runtime | Compile-time |
| **Use case** | Convenience | Interop | Generics |

**Real-World Example:**
```csharp
// API response handling
public IActionResult GetData(string returnType)
{
    var data = _service.GetData();  // var: List<User>
    
    if (returnType == "xml")
    {
        dynamic xml = ConvertToXml(data);  // dynamic for late binding
        return Content(xml.ToString());
    }
    
    // Cache in object form
    object cached = _cache.Get("users");  // object: could be anything
    if (cached is List<User> users)
        return Ok(users);
    
    return Ok(data);
}
```

---

## const vs readonly

### Overview
Two ways to define constants, with different evaluation timing and usage scenarios.

### const - Compile-Time Constant

**When to Use:**
- Values known at compile time
- Never changing (mathematical constants)
- Embedded into calling code

**Code Example:**
```csharp
public class MathConstants
{
    public const double Pi = 3.14159265359;
    public const int MaxRetries = 3;
    public const string ApiVersion = "v1";
    
    // Error: must be primitive or string
    // public const DateTime StartDate = DateTime.Now;  // ❌ Not constant
    
    // Error: cannot use new (except null for reference types)
    // public const List<int> Numbers = new List<int>();  // ❌
}

// Usage - value embedded at compile time
public double CalculateCircleArea(double radius)
{
    return MathConstants.Pi * radius * radius;
    // IL: ldc.r8 3.14159265359 (literal embedded)
}
```

**Memory Visualization:**
```
Compile-time embedding:

Source code:
┌─────────────────────────────┐
│ const int MaxRetries = 3;   │
│                             │
│ void Run() {                │
│     for (int i = 0; i <     │
│         MaxRetries; i++)     │
│ }                            │
└─────────────────────────────┘
              ↓
Compiler:
┌─────────────────────────────┐
│ IL Code in Run():           │
│   ldc.i4.3  ← literal 3     │
│   ...                       │
│ No reference to MaxRetries  │
│ Constant is "baked in"     │
└─────────────────────────────┘
```

### readonly - Runtime Constant

**When to Use:**
- Values computed at runtime
- Instance-level constants
- Complex initialization

**Code Example:**
```csharp
public class Configuration
{
    // Static readonly - computed once
    public static readonly DateTime StartupTime = DateTime.Now;
    public static readonly string MachineName = Environment.MachineName;
    
    // Instance readonly - set in constructor
    public readonly string Id;
    public readonly int MaxConnections;
    
    public Configuration(string id, int maxConnections)
    {
        Id = id;                        // Can assign in constructor
        MaxConnections = maxConnections;
    }
    
    // Error: cannot assign outside constructor
    public void Update() {
        // MaxConnections = 100;  // ❌ Error
    }
}

// Usage
var config = new Configuration("prod-1", 100);
Console.WriteLine(Configuration.StartupTime);
```

**Memory Visualization:**
```
Runtime initialization:

Static readonly:
┌─────────────────────────────────────────────┐
│ public static readonly DateTime StartupTime│
│   = DateTime.Now;                           │
├─────────────────────────────────────────────┤
│ Static constructor / type initializer:        │
│   Before first use: execute initialization  │
│   Store value in static field              │
│                                             │
│ Memory layout:                              │
│ ┌─────────────────┐                         │
│ │ StartupTime     │ ──→ DateTime struct    │
│ │ (static field)  │                         │
│ └─────────────────┘                         │
└─────────────────────────────────────────────┘

Instance readonly:
┌─────────────────────────────────────────────┐
│ Heap object:                                │
│ ┌────────────────────────┐                  │
│ │ Object header          │                  │
│ │ Method table           │                  │
│ │ Id (readonly string)   │ ──→ "prod-1"     │
│ │ MaxConnections (int)   │     100          │
│ └────────────────────────┘                  │
└─────────────────────────────────────────────┘
```

### Comparison Table

| Feature | const | readonly |
|---------|-------|----------|
| **Evaluation** | Compile-time | Runtime |
| **Type restrictions** | Primitives, string, null | Any type |
| **Static/Instance** | Implicitly static | Both |
| **Assignment** | Declaration only | Constructor too |
| **Versioning** | Changes require recompile | Binary compatible |
| **Memory** | Embedded in IL | Stored in field |

**Real-World Example:**
```csharp
public class ApiClient
{
    // const: never changes, compiled into callers
    public const string DefaultApiVersion = "v2";
    public const int DefaultTimeoutMs = 30000;
    
    // readonly: set per instance, runtime values
    public readonly string BaseUrl;
    public readonly string ApiKey;
    public readonly HttpClient HttpClient;
    
    public ApiClient(string baseUrl, string apiKey)
    {
        BaseUrl = baseUrl ?? throw new ArgumentNullException(nameof(baseUrl));
        ApiKey = apiKey;
        HttpClient = new HttpClient { Timeout = TimeSpan.FromMilliseconds(DefaultTimeoutMs) };
    }
}

// Versioning consideration:
// If const DefaultTimeoutMs changes, ALL assemblies using it must recompile
// If it were readonly, only the library needs updating
```

---

## nameof Operator

### Overview
Returns the string name of a variable, type, or member, resolved at compile time.

### Why Use nameof?
- Refactoring safe (renames propagate)
- Compile-time checking
- No runtime overhead

**Code Example:**
```csharp
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
}

public void RegisterUser(string name, int age)
{
    // Without nameof - fragile
    if (string.IsNullOrWhiteSpace(name))
        throw new ArgumentException("Value cannot be empty", "name");  // Magic string!
    
    // With nameof - refactoring safe
    if (age < 0)
        throw new ArgumentOutOfRangeException(nameof(age), "Age must be non-negative");
    
    // Works with properties
    var propertyName = nameof(User.Name);  // "Name"
    
    // Works with methods
    var methodName = nameof(RegisterUser);  // "RegisterUser"
    
    // Works with types
    var typeName = nameof(User);  // "User"
}

// Property change notification (INotifyPropertyChanged)
public class ViewModel : INotifyPropertyChanged
{
    private string _userName;
    
    public string UserName
    {
        get => _userName;
        set
        {
            if (_userName != value)
            {
                _userName = value;
                OnPropertyChanged(nameof(UserName));  // "UserName"
            }
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected void OnPropertyChanged(string propertyName) => 
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
}
```

**Memory Visualization:**
```
Compile-time resolution:

Source:
┌─────────────────────────────────────────────┐
│ nameof(user.Name)                            │
├─────────────────────────────────────────────┤
│ Compiler:                                   │
│   - Find symbol "user.Name"                 │
│   - Extract simple name: "Name"             │
│   - Emit IL: ldstr "Name"                   │
│                                             │
│ Result: Same as "Name" literal               │
│   - Zero runtime cost                        │
│   - Refactoring updates automatically        │
└─────────────────────────────────────────────┘
```

**Real-World Example:**
```csharp
// Logging with context
public class Logger<T>
{
    private readonly string _category = typeof(T).Name;
    
    public void LogDebug(Expression<Func<object>> expression)
    {
        var memberName = ((MemberExpression)expression.Body).Member.Name;
        var value = expression.Compile()();
        Console.WriteLine($"[{_category}] {memberName} = {value}");
    }
}

// Usage
var logger = new Logger<OrderService>();
logger.LogDebug(() => order.Total);  // Logs: "Total = 150.00"

// Argument validation helper
public static class ThrowHelper
{
    public static void IfNull(object value, string paramName)
    {
        if (value == null)
            throw new ArgumentNullException(paramName);
    }
}

// Usage - nameof ensures parameter name stays correct
void Process(User user)
{
    ThrowHelper.IfNull(user, nameof(user));
    // ...
}
```

---

## string[] args in Main

### Overview
Command-line arguments passed to console applications.

### How It Works
```csharp
// Entry point signature
class Program
{
    static void Main(string[] args)
    {
        // args contains all command-line arguments
        // args[0] = first argument (not program name)
    }
}

// Modern C# top-level statements (C# 9+)
// args is implicitly available
Console.WriteLine($"Arguments count: {args.Length}");
```

**Code Example:**
```csharp
class Program
{
    static void Main(string[] args)
    {
        // No arguments
        if (args.Length == 0)
        {
            Console.WriteLine("Usage: MyApp --name=John --age=25");
            return;
        }
        
        // Iterate arguments
        foreach (var arg in args)
        {
            Console.WriteLine($"Argument: {arg}");
        }
        
        // Parse named arguments
        var config = ParseArguments(args);
        Console.WriteLine($"Name: {config.Name}, Age: {config.Age}");
    }
    
    static (string Name, int Age) ParseArguments(string[] args)
    {
        string name = null;
        int age = 0;
        
        foreach (var arg in args)
        {
            if (arg.StartsWith("--name="))
                name = arg.Substring(7);
            else if (arg.StartsWith("--age="))
                int.TryParse(arg.Substring(6), out age);
        }
        
        return (name, age);
    }
}

// Run: dotnet run -- --name=Belal --age=25
// Output:
// Argument: --name=Belal
// Argument: --age=25
// Name: Belal, Age: 25
```

**Memory Visualization:**
```
Command line: dotnet run -- --name=Belal --env=prod

┌─────────────────────────────────────────────────┐
│ args array (Heap):                              │
│ ┌─────────────────┐                             │
│ │ Length: 2       │                             │
│ ├─────────────────┤                             │
│ │ [0] ──→ "--name=Belal"                       │
│ │ [1] ──→ "--env=prod"                         │
│ └─────────────────┘                             │
│       ↑                                         │
│ args reference on stack points to array          │
└─────────────────────────────────────────────────┘
```

**Real-World Example:**
```csharp
// CLI tool with arguments
public class Program
{
    static async Task Main(string[] args)
    {
        var rootCommand = new RootCommand("File processor");
        
        var inputOption = new Option<string>(
            "--input",
            "Input file path") { IsRequired = true };
        
        var outputOption = new Option<string>(
            "--output",
            "Output file path");
        
        var verboseOption = new Option<bool>(
            "--verbose",
            "Enable verbose output");
        
        rootCommand.AddOption(inputOption);
        rootCommand.AddOption(outputOption);
        rootCommand.AddOption(verboseOption);
        
        rootCommand.SetHandler(async (string input, string output, bool verbose) =>
        {
            await ProcessFile(input, output, verbose);
        }, inputOption, outputOption, verboseOption);
        
        await rootCommand.InvokeAsync(args);
    }
}

// Usage: myapp --input=data.csv --output=result.csv --verbose
```

---

*Source: C# language specification, .NET documentation, and modern C# programming practices.*
