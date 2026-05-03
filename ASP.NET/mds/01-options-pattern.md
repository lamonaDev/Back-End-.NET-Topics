# Options Pattern in ASP.NET Core

## Concept Overview

The **Options Pattern** is ASP.NET Core's recommended way to bind strongly-typed configuration objects to sections of `appsettings.json` (or any configuration source). Instead of reading raw strings from `IConfiguration`, you define a plain C# class and let the framework bind and inject it.

This pattern provides several key advantages over traditional configuration access:

- **Strongly-typed access**: No magic strings or casting errors
- **IntelliSense support**: IDE autocomplete for configuration properties
- **Validation**: Built-in support for DataAnnotations and custom validation
- **DI integration**: Configuration objects are injected like any other service
- **Post-configuration**: Ability to modify values after binding

Three main interfaces exist: `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`. Choosing the wrong one can cause stale configuration data in long-running services or unnecessary re-reads.

## The Three Interfaces Compared

| Interface | DI Lifetime | Reads Config | Re-reads on Change | Best For |
|-----------|-------------|--------------|---------------------|----------|
| `IOptions<T>` | Singleton | Once at startup | Never | Immutable app-wide settings (DB timeouts, API base URLs) |
| `IOptionsSnapshot<T>` | Scoped | Once per request | Per request cycle | Web API controllers, per-request feature flags |
| `IOptionsMonitor<T>` | Singleton | On demand + on change | Real-time via callback | Background services, rate limiters, long-running jobs |

### When to Use Each Interface

**IOptions<T> (Singleton)**
- Use for configuration that never changes during application lifetime
- Ideal for database connection timeouts, API base URLs, caching durations
- Loaded once at application startup
- Most memory-efficient option

**IOptionsSnapshot<T> (Scoped)**
- Use for configuration that may change between requests
- Ideal for feature flags, tenant-specific settings, request-specific configuration
- Creates a snapshot at the beginning of each HTTP request
- Perfect for web APIs where each request might need fresh configuration

**IOptionsMonitor<T> (Singleton with callbacks)**
- Use for configuration that changes at runtime and you need to react
- Ideal for background workers, rate limiters, configuration reload scenarios
- Supports `OnChange` callbacks for reacting to configuration updates
- Best of both worlds: singleton lifetime with change notifications

> **Critical Pitfall**: Never inject `IOptionsSnapshot<T>` into a **Singleton** service — this causes a "captive dependency" and will throw an `InvalidOperationException` at runtime. Use `IOptionsMonitor<T>` for singletons.

### Captive Dependency Explanation

```
// WRONG - This will throw InvalidOperationException at runtime
public class MySingletonService
{
    public MySingletonService(IOptionsSnapshot<Settings> options)  // WRONG!
    {
        // Singleton service holds onto Scoped dependency
    }
}

// CORRECT - Use IOptionsMonitor for singleton services
public class MySingletonService
{
    public MySingletonService(IOptionsMonitor<Settings> options)  // CORRECT!
    {
        // Monitor can be injected into singletons
    }
}
```

## Configuration Binding Flow

```
+------------------+     +-------------------+     +------------------+
|   appsettings    |     |   Configuration  |     |   POCO Class     |
|       .json      |---->|    Provider      |---->|   (TOptions)    |
+------------------+     +-------------------+     +------------------+
                               |                            |
                               v                            v
                        +-------------------+     +------------------+
                        |  IConfiguration   |     |  IOptions<T>      |
                        |   (Dictionary)   |     |  (DI Container)  |
                        +-------------------+     +------------------+
```

## Realistic Configuration Class + Binding

### appsettings.json

```json
{
  "EmailSettings": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "From": "no-reply@myapp.com",
    "EnableSsl": true,
    "MaxRetries": 3
  }
}
```

### EmailSettings.cs — POCO with DataAnnotations validation

```csharp
using System.ComponentModel.DataAnnotations;

public class EmailSettings
{
    [Required]
    public string SmtpHost { get; set; } = "";

    [Range(1, 65535)]
    public int SmtpPort { get; set; }

    [Required, EmailAddress]
    public string From { get; set; } = "";

    public bool EnableSsl { get; set; }

    [Range(0, 10)]
    public int MaxRetries { get; set; } = 3;
}
```

### Program.cs — Registration with validation

```csharp
builder.Services
    .AddOptions<EmailSettings>()
    .BindConfiguration("EmailSettings")
    .ValidateDataAnnotations()
    .ValidateOnStart(); // Fail fast — crash at startup, not at runtime

// Alternative: custom validation
builder.Services
    .AddOptions<EmailSettings>()
    .BindConfiguration("EmailSettings")
    .Validate(s => s.SmtpPort != 25, "Port 25 is blocked — use 587 or 465")
    .ValidateOnStart();
```

## Usage in a Web API Controller

```csharp
public class NotificationController : ControllerBase
{
    private readonly EmailSettings _email;

    // IOptionsSnapshot re-reads config each request — good for feature flags
    public NotificationController(IOptionsSnapshot<EmailSettings> options)
        => _email = options.Value;

    [HttpPost("send")]
    public IActionResult Send()
    {
        // Use _email.SmtpHost, _email.SmtpPort …
        return Ok($"Sending via {_email.SmtpHost}:{_email.SmtpPort}");
    }
}
```

## Usage in a Background Service (IOptionsMonitor)

```csharp
public class EmailQueueWorker : BackgroundService
{
    private readonly IOptionsMonitor<EmailSettings> _monitor;

    public EmailQueueWorker(IOptionsMonitor<EmailSettings> monitor)
    {
        _monitor = monitor;
        // React to config file changes at runtime (no restart needed)
        _monitor.OnChange(settings =>
            Console.WriteLine($"Config changed: new host = {settings.SmtpHost}"));
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var current = _monitor.CurrentValue; // always up-to-date
            // process queue using current settings…
            await Task.Delay(5000, ct);
        }
    }
}
```

## Advanced: Post-Configuration Action

Sometimes you need to modify configuration values after binding:

```csharp
builder.Services.Configure<EmailSettings>(options =>
{
    // Post-configuration: modify values after binding
    options.SmtpHost = options.SmtpHost?.ToLower();
});
```

## Validation Failure Behavior

- Without `ValidateOnStart()`: validation runs on *first access* of `.Value` — the app starts fine but crashes at runtime when the option is first used.
- With `ValidateOnStart()`: an `OptionsValidationException` is thrown during application startup — app fails fast with a clear error message. This is **strongly preferred** in production.
- The exception message lists all failing validation rules, not just the first one.

### Example Validation Exception

```
OptionsValidationException: Multiple validation failures:
1. The SmtpPort field must be between 1 and 65535.
2. The From field is required.
3. The MaxRetries field must be between 0 and 10.
```

## Best Practices Checklist

- Always use `ValidateOnStart()` in production environments
- Choose the correct interface based on your service lifetime
- Use DataAnnotations for simple validation rules
- Use custom validation for complex business rules
- Never store sensitive data in appsettings.json — use User Secrets or Key Vault
- Consider using `IOptionsMonitor` for services that need to react to config changes

## References

- [Microsoft Docs — Options Pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options)
- [Andrew Lock — Options Validation Deep Dive](https://andrewlock.net/adding-validation-to-strongly-typed-configuration-objects-in-dotnet-6/)
- [IOptions, IOptionsSnapshot, IOptionsMonitor — .NET Docs](https://learn.microsoft.com/en-us/dotnet/core/extensions/options)