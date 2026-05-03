# Azure Configurations + Environment Variables

## Overview

Proper configuration management is critical for deploying ASP.NET Core applications to Azure. Understanding how Azure App Service handles configuration, how to secure sensitive data, and how to leverage Azure's features for configuration management will help you build secure and maintainable applications.

## How Azure App Service Configuration Works

Azure App Service exposes two types of configuration in the **Configuration** blade of the Azure Portal:

- **Application Settings** -> become environment variables inside the app process. They *override* any matching keys in `appsettings.json`.
- **Connection Strings** -> also become environment variables, prefixed by type (e.g., `SQLAZURECONNSTR_DefaultConnection`).

### Configuration Flow Visualized

```
┌─────────────────────────────────────────────────────────────────────┐
│              Configuration Override Flow in Azure                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. appsettings.json                                                 │
│        │                                                             │
│        │ (bundled in deployment package)                             │
│        ▼                                                             │
│  2. appsettings.Production.json                                     │
│        │                                                             │
│        │ (if ASPNETCORE_ENVIRONMENT=Production)                    │
│        ▼                                                             │
│  3. Azure App Service — Application Settings                        │
│        │                                                             │
│        │ (Portal / ARM / CLI / GitHub Actions)                       │
│        │ Override values like:                                       │
│        │ - Logging:LogLevel:Default = "Warning"                     │
│        │ - FeatureFlags:NewUI = "true"                               │
│        ▼                                                             │
│  4. Azure App Service — Connection Strings                          │
│        │                                                             │
│        │ (SQLAZURECONNSTR_DefaultConnection)                         │
│        │ Maps to ConnectionStrings:DefaultConnection in config     │
│        ▼                                                             │
│  5. Effective Runtime IConfiguration                                │
│        │                                                             │
│        │ Your app reads via DI:                                       │
│        │ configuration["ConnectionStrings:DefaultConnection"]       │
│        │ or GetConnectionString("DefaultConnection")                │
│        ▼                                                             │
│  6. Your Application Code                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Application Settings

Application Settings in Azure become environment variables that your application can read just like any other configuration source.

### Setting Application Settings

**Azure Portal:**
```
Configuration → Application Settings → New application setting
Name: EmailSettings__SmtpHost
Value: smtp.sendgrid.com
```

**Azure CLI:**
```bash
az webapp config appsettings set \
  --resource-group MyRG \
  --name my-api \
  --settings EmailSettings__SmtpHost=smtp.sendgrid.com \
             EmailSettings__SmtpPort=587 \
             FeatureFlags__NewUI=true
```

**ARM Template:**
```json
{
  "resources": [{
    "type": "Microsoft.Web/sites/config",
    "name": "appsettings",
    "properties": {
      "EmailSettings__SmtpHost": "smtp.sendgrid.com",
      "FeatureFlags__NewUI": "true"
    }
  }]
}
```

## Configuration Override Flow

```
1. appsettings.json                              // bundled in deployment package
   ↓ overridden by
2. appsettings.Production.json                  // if ASPNETCORE_ENVIRONMENT=Production
   ↓ overridden by
3. Azure App Service — Application Settings     // Portal / ARM / CLI / GitHub Actions
   ↓ overridden by
4. Azure App Service — Connection Strings       // prefixed env vars, highest for DB strings
   ↓ becomes
5. Effective Runtime IConfiguration             // what your app reads via DI
```

## Key Naming in Azure App Settings (Nested Keys)

For nested configuration (e.g., `EmailSettings:SmtpHost`), Azure App Service uses **colons** in the Name field on the portal, or double underscores in CLI:

```bash
# Azure CLI — set nested app setting
az webapp config appsettings set \
  --resource-group MyRG \
  --name my-api \
  --settings EmailSettings__SmtpHost=smtp.sendgrid.com \
             EmailSettings__SmtpPort=587 \
             Jwt__Secret=@secure-value-from-keyvault
```

### Nested Key Mapping

| appsettings.json | Azure App Setting Name | Environment Variable |
|-----------------|----------------------|---------------------|
| EmailSettings:SmtpHost | EmailSettings:SmtpHost | EmailSettings__SmtpHost |
| Logging:LogLevel:Default | Logging:LogLevel:Default | Logging__LogLevel__Default |
| Jwt:Secret | Jwt:Secret | Jwt__Secret |

## Connection Strings in Azure

When you add a Connection String in Azure Portal under Connection Strings (not Application Settings), it is injected as environment variables with a type prefix:

### Connection String Prefixes

| Azure Type | Environment Variable Prefix | Maps To |
|------------|----------------------------|---------|
| SQLAzure | `SQLAZURECONNSTR_` | ConnectionStrings:DefaultConnection |
| SQLServer | `SQLCONNSTR_` | ConnectionStrings:DefaultConnection |
| MySQL | `MYSQLCONNSTR_` | ConnectionStrings:DefaultConnection |
| PostgreSQL | `POSTGRESQLCONNSTR_` | ConnectionStrings:DefaultConnection |
| Custom | `CUSTOMCONNSTR_` | ConnectionStrings:CustomConnection |

### How ASP.NET Core Reads Connection Strings

```csharp
// This automatically looks for:
// - SQLAZURECONNSTR_DefaultConnection (Azure SQL)
// - SQLCONNSTR_DefaultConnection (SQL Server)
// - DefaultConnection in appsettings.json
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
```

## Secure Secret Handling with Azure Key Vault

For production secrets, never paste them directly in Application Settings. Use **Key Vault References**:

### Key Vault Reference Syntax

```
# In Azure App Service Application Settings — KEY VAULT REFERENCE syntax
Jwt__Secret = @Microsoft.KeyVault(SecretUri=https://my-vault.vault.azure.net/secrets/JwtSecret/)
```

### Azure Portal Configuration

```
Configuration → Application Settings → New setting
Name: Jwt__Secret
Value: @Microsoft.KeyVault(SecretUri=https://my-vault.vault.azure.net/secrets/JwtSecret/)
```

### Program.cs — add Azure Key Vault as config provider

```csharp
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

if (builder.Environment.IsProduction())
{
    var keyVaultUri = new Uri($"https://{builder.Configuration["KeyVaultName"]}.vault.azure.net/");
    builder.Configuration.AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential());
    // DefaultAzureCredential uses Managed Identity in Azure — no credentials in code!
}
```

### Using Managed Identity

```csharp
// Enable Managed Identity in Azure Portal:
// Azure Portal → Identity → System-assigned → On

// Program.cs - no secrets needed!
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());
```

## Environment Variables in ASP.NET Core

### Setting ASPNETCORE_ENVIRONMENT

```
Configuration → Application Settings → New setting
Name: ASPNETCORE_ENVIRONMENT
Value: Production
```

### Environment-Specific Behavior

| Environment | Behavior |
|-------------|-----------|
| Development | User Secrets loaded, detailed errors, verbose logging |
| Staging | Production-like, detailed errors for testing |
| Production | No User Secrets, generic errors, minimal logging |

### Detecting Environment in Code

```csharp
// Check current environment
if (app.Environment.IsDevelopment())
{
    // Development-only code
}

if (app.Environment.IsProduction())
{
    // Production-only code
}

// Or use environment-specific configurations
builder.Configuration.AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json");
```

## Deployment Slots

Azure App Service supports **deployment slots** for testing changes before production:

```
Production Slot (myapp.azurewebsites.net)
    │
    │ [Swap]
    │
Staging Slot (myapp-staging.azurewebsites.net)
    │
    │ (Test here first)
    │
GitHub Actions Deployment
```

### Benefits of Deployment Slots

- **Zero-downtime deployments** — Swap production with staging
- **Test in production-like environment** — Same infrastructure
- **Warm-up before production** — Pre-load cache, JIT compile
- **Rollback** — Swap back if issues occur

## Azure App Service vs Kubernetes Configuration

If using Azure Kubernetes Service (AKS):

| Aspect | App Service | AKS |
|--------|-------------|-----|
| Config Management | App Settings + Key Vault | ConfigMaps + Secrets |
| Environment Variables | Automatic from App Settings | Manual, need Ingress or env vars |
| Secrets | Key Vault Reference | Kubernetes Secrets + Key Vault CSI |
| Configuration Reload | Restart required | Can reload without restart |

## Production-Safe Configuration Checklist

- **Do**: Set `ASPNETCORE_ENVIRONMENT=Production` in Azure App Service settings
- **Do**: Use Azure App Service Application Settings for non-secret config overrides
- **Do**: Use Key Vault references or direct Key Vault provider for all secrets
- **Do**: Enable Managed Identity on your App Service — avoids credential management entirely
- **Do**: Review Application Settings in the portal are encrypted at rest and masked in portal UI
- **Do**: Use deployment slots for staging before production
- **Do**: Configure backup for critical data
- **Don't**: Hard-code production connection strings or secrets in appsettings.json
- **Don't**: Put secrets directly in Application Settings — use Key Vault references instead
- **Don't**: Forget to set `ASPNETCORE_ENVIRONMENT` — default is `Production` on Azure but be explicit
- **Don't**: Use the same Key Vault for dev and production (separate vaults per environment)
- **Note**: Use Deployment Slots (staging/production) to test config before swapping to production

## Best Practices Summary

### Development
- Use User Secrets locally
- Use `appsettings.Development.json` for dev-specific config
- Test with `ASPNETCORE_ENVIRONMENT=Development`

### Staging
- Mirror production settings
- Use Key Vault with staging secrets
- Test deployment process

### Production
- Use Key Vault for all secrets
- Set explicit `ASPNETCORE_ENVIRONMENT=Production`
- Use deployment slots
- Enable Managed Identity
- Enable auto-scaling

## References

- [Azure Docs — Configure App Settings](https://learn.microsoft.com/en-us/azure/app-service/configure-common)
- [Azure Docs — Use Key Vault References in App Service](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references)
- [Azure Docs — Managed Identity for App Service](https://learn.microsoft.com/en-us/azure/app-service/overview-managed-identity)
- [Reference Video — Azure Configurations (YouTube)](https://www.youtube.com/watch?v=F0I6HgmvYFk)