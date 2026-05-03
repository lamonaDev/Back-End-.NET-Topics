# User Secrets — What and Why

## What Are User Secrets?

User Secrets is a development-only mechanism that stores sensitive configuration (API keys, connection strings, passwords) **outside the project directory**, so they are never accidentally committed to source control. The secret store lives in your OS user profile folder, not inside the repo.

### Storage Locations

| Platform | Location |
|----------|----------|
| Windows | `%APPDATA%\Microsoft\UserSecrets\<guid>\secrets.json` |
| Linux / macOS | `~/.microsoft/usersecrets/<guid>/secrets.json` |

Each project gets a unique ID (GUID) generated when you run `dotnet user-secrets init`, ensuring secrets are isolated per project.

## Why They Are Needed

Storing secrets in `appsettings.json` is a catastrophic mistake: the file is tracked by git, pushed to GitHub, and exposed to anyone with repo access (teammates, CI bots, or public viewers). This is responsible for thousands of real credential leaks every year — GitHub's secret scanning catches millions of leaked tokens annually.

### Real-World Consequences

- **Credential Theft**: Exposed API keys can be used by attackers to access your services
- **Data Breaches**: Database credentials can lead to complete database compromise
- **Financial Impact**: AWS/Azure credentials can result in unexpected bills
- **Reputation Damage**: Customer data leaks can destroy trust
- **Legal Consequences**: GDPR, HIPAA violations can result in massive fines

> **Never Do This**: `"ConnectionStrings": { "Default": "Server=prod;User=sa;Password=MyS3cret!" }`
> This appears in git history even after deletion. Use `git filter-repo` to purge it.

### The Git History Problem

Even if you delete a secret from `appsettings.json`, it remains in git history forever:

```
# This secret is STILL in git history
git log --all -p | grep "Password"
```

Tools like GitGuardian, TruffleHog, and GitHub's secret scanning continuously scan public repos for leaked credentials.

## Setup and Usage

### Terminal — initialize secrets for a project

```bash
# Run inside your project folder
dotnet user-secrets init

# Set a secret
dotnet user-secrets set "ConnectionStrings:Default" "Server=.;Database=MyDb;Trusted_Connection=True"
dotnet user-secrets set "Jwt:Secret" "super-secret-key-here"

# List all secrets
dotnet user-secrets list

# Remove a secret
dotnet user-secrets remove "Jwt:Secret"
```

### Program.cs — User Secrets are auto-loaded in Development by default

```csharp
// WebApplication.CreateBuilder() automatically adds User Secrets
// when ASPNETCORE_ENVIRONMENT == "Development"
var builder = WebApplication.CreateBuilder(args);
// No extra code needed — secrets.json is merged into IConfiguration
```

### Manual Configuration (if needed)

```csharp
// Only needed if using WebHost or older template
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

## How User Secrets Work

```
+------------------+     +-------------------+     +------------------+
|   secrets.json   |     |   Configuration   |     |  IConfiguration  |
|  (User Profile)  |---->|     Provider      |---->|    (Merged)      |
+------------------+     +-------------------+     +------------------+
                                                       |
                                                       v
                                                +------------------+
                                                | Your Code        |
                                                | DI Container     |
                                                +------------------+
```

## User Secrets vs Environment Variables

| Aspect | User Secrets | Environment Variables |
|--------|--------------|----------------------|
| Scope | Local developer machine only | Any environment |
| Environment | Development only | Dev / Staging / Production |
| Storage | OS user profile folder | OS process environment / shell |
| Visibility | Only current OS user | All processes in the session |
| Production use | Not suitable | Suitable (with caution) |
| Azure App Service | Not applicable | Application Settings UI |
| Version Control | Not tracked | Not tracked |
| Per-developer override | Yes | Yes |

### When to Use Each

**User Secrets**:
- Local development only
- Never need to share with team (each dev has their own values)
- Testing with real credentials locally

**Environment Variables**:
- Any environment (dev, staging, production)
- Container deployments (Docker, Kubernetes)
- CI/CD pipelines
- Azure App Service Application Settings

## Secret Management Flow: Local → CI/CD → Production

```
Developer Machine  ──→  secrets.json (User Secrets)        // never in git
      │
      │              dotnet user-secrets set "Key" "Value"
      │
      v
GitHub Actions / Azure DevOps  ──→  Pipeline Secret Variables  // masked in logs
      │
      │              GitHub Secrets / Azure Pipeline Variables
      │
      v
Azure App Service  ──→  App Settings / Key Vault Reference  // encrypted at rest
      │
      │              ASPNETCORE_ENVIRONMENT=Production
      │
      v
Production Runtime  ──→  IConfiguration (env vars override appsettings)
```

## Azure Key Vault for Production

For production, use Azure Key Vault instead of Application Settings for sensitive data:

```
Azure Portal → Key Vault → Secrets → Generate/Import
Azure Portal → App Service → Configuration → Key Vault Reference
```

Benefits:
- Secrets are encrypted at rest
- Access controlled via Azure AD
- Audit logs for secret access
- No secret rotation pain (update in one place)

## Do / Don't Checklist

- **Do**: Run `dotnet user-secrets init` before adding any sensitive config locally
- **Do**: Add `secrets.json` path pattern to `.gitignore` as a safety net
- **Do**: Use Azure Key Vault or AWS Secrets Manager in staging/production
- **Do**: Rotate secrets regularly; treat leaked secrets as permanently compromised
- **Do**: Use .gitignore to exclude configuration files that might contain secrets
- **Don't**: Store passwords, API keys, or connection strings in `appsettings.json`
- **Don't**: Commit `.env` files to source control
- **Don't**: Use User Secrets in Docker containers or CI pipelines — use pipeline secrets instead
- **Don't**: Log configuration values that may contain sensitive data
- **Don't**: Share secrets via chat, email, or documentation

## References

- [Microsoft Docs — Safe Storage of App Secrets in Development](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets)
- [Azure Key Vault Overview](https://learn.microsoft.com/en-us/azure/key-vault/general/overview)
- [GitGuardian — State of Secrets Sprawl Report](https://gitguardian.com/state-of-secrets-sprawl)