# Configuration Sources — All Ways

## Overview

ASP.NET Core provides a flexible configuration system that reads from multiple sources, merging them into a single `IConfiguration` interface. Understanding the precedence order is critical for debugging configuration issues and building resilient applications.

## Configuration Provider Precedence (Lowest → Highest)

ASP.NET Core merges configuration from multiple sources. **Later sources override earlier ones** for the same key. This is the default order when using `WebApplication.CreateBuilder()`:

```
1. appsettings.json                              // base defaults
2. appsettings.{Environment}.json               // env-specific overrides (e.g. Development, Production)
3. User Secrets                                   // Development only, per-developer local overrides
4. Environment Variables                         // system/container overrides
5. Command-line Arguments                        // highest priority, runtime overrides
```

### Understanding the Precedence

Think of this like layers:
- Layer 1 (appsettings.json): The foundation — default values everyone gets
- Layer 2 (appsettings.Production.json): Environment-specific tweaks
- Layer 3 (User Secrets): Developer personal overrides
- Layer 4 (Environment Variables): System-level configuration (Docker, Azure)
- Layer 5 (Command-line): Emergency overrides for operators

> **Key Rule**: A value set via an **environment variable** always wins over `appsettings.json`. A **command-line argument** wins over everything. This is intentional for container/cloud deployments.

### Visual Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Configuration Pipeline                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   appsettings.json                                          │
│        │                                                     │
│        │ (override)                                         │
│        v                                                     │
│   appsettings.{Environment}.json                            │
│        │                                                     │
│        │ (override)                                         │
│        v                                                     │
│   User Secrets (Development only)                           │
│        │                                                     │
│        │ (override)                                         │
│        v                                                     │
│   Environment Variables                                     │
│        │                                                     │
│        │ (override)                                         │
│        v                                                     │
│   Command-line Arguments                                    │
│        │                                                     │
│        │ (final)                                            │
│        v                                                     │
│   IConfiguration (Your App)                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Precedence Matrix

| Source | Priority | Env | Use Case |
|--------|----------|-----|----------|
| appsettings.json | Lowest (1) | All | Baseline defaults, non-sensitive settings |
| appsettings.Development.json | 2 | Development | Local dev overrides (verbose logging, local DB) |
| User Secrets | 3 | Development | Developer-specific secrets, never in source control |
| Environment Variables | 4 | All | Docker, CI/CD, Azure App Service settings |
| Command-line Arguments | Highest (5) | All | One-off overrides, testing, scripted deployments |

## Demo: Reading the Same Key from All Sources

### appsettings.json
```json
{ "AppName": "From appsettings.json" }
```

### appsettings.Development.json
```json
{ "AppName": "From appsettings.Development.json" }
```

### User Secret (set via CLI)
```
dotnet user-secrets set "AppName" "From User Secrets"
```

### Program.cs — In-memory config + printing resolved value

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add in-memory config (lower than env vars but useful for testing)
builder.Configuration.AddInMemoryCollection(new Dictionary<string, string>
{
    ["AppName"] = "From InMemory"  // will be overridden by higher-priority sources
});

var app = builder.Build();

app.MapGet("/config-demo", (IConfiguration config) =>
{
    var value = config["AppName"];
    return Results.Ok(new {
        ResolvedValue = value,
        Environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
    });
});
app.Run();
```

### Expected Output per Scenario

| Scenario | Resolved Value |
|----------|---------------|
| No env var, no CLI arg, Environment=Development | "From User Secrets" |
| No User Secret, Environment=Development | "From appsettings.Development.json" |
| ENV AppName=FromEnvVar set in terminal | "FromEnvVar" |
| dotnet run --AppName="FromCLI" | "FromCLI" |
| Environment=Production, no overrides | "From appsettings.json" |

## Adding Custom Configuration Sources

### JSON File Configuration

```csharp
builder.Configuration.AddJsonFile("custom.json", optional: true, reloadOnChange: true);
```

### XML Configuration

```csharp
builder.Configuration.AddXmlFile("config.xml", optional: true);
```

### INI File Configuration

```csharp
builder.Configuration.AddIniFile("config.ini", optional: true);
```

### Environment Variables from Custom Prefix

```csharp
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");
```

### Command-line Args with Custom Prefix

```csharp
builder.Configuration.AddCommandLineArgs(args, prefix: "custom");
```

## Environment Variable Nested Key Separator

When setting nested config keys via environment variables, use `__` (double underscore) as the separator on Linux, or `:` on Windows:

```bash
# Linux/Docker — maps to EmailSettings:SmtpHost
export EmailSettings__SmtpHost=smtp.sendgrid.com

# Windows
set EmailSettings:SmtpHost=smtp.sendgrid.com
```

### Common Mapping Examples

| appsettings.json key | Environment Variable |
|---------------------|---------------------|
| ConnectionStrings.Default | `ConnectionStrings__Default` |
| Logging.LogLevel.Default | `Logging__LogLevel__Default` |
| EmailSettings.Smtp.Host | `EmailSettings__Smtp__Host` |

## Configuration in Different Environments

### Development Environment
- Verbose logging enabled
- User Secrets available
- Local database connection
- Browser link errors shown
- Detailed exception pages

### Production Environment
- Minimal logging
- No User Secrets (not loaded)
- Production database
- Generic error pages
- Strict security headers

### Switching Environments

```bash
# Via environment variable (recommended for production)
export ASPNETCORE_ENVIRONMENT=Production

# Via command-line argument
dotnet run --environment Production

# Via launch profile (VS)
```

## Best Practices

1. **Never store secrets in appsettings.json** — use User Secrets or Key Vault
2. **Use environment-specific files** — appsettings.Production.json for prod defaults
3. **Validate configuration at startup** — use ValidateOnStart() with options pattern
4. **Document required configuration** — provide a template or checklist
5. **Use Azure Key Vault in production** — centralize secret management

## References

- [Microsoft Docs — Configuration in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/)
- [Use multiple environments in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/environments)