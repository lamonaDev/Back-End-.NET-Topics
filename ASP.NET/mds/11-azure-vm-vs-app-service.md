# Azure VMs vs Azure App Service

## Overview

Choosing between Azure Virtual Machines (IaaS) and Azure App Service (PaaS) is a fundamental infrastructure decision that affects your application's scalability, maintenance burden, and operational costs. Understanding the trade-offs is essential for making the right choice for your project.

## Understanding the Models

### Infrastructure as a Service (IaaS)

```
IaaS (Virtual Machines):
┌─────────────────────────────────────────────────────────────────┐
│                        Your Responsibility                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Your Application                                           │ │
│  │ Your Runtime (.NET, Node, Python)                          │ │
│  │ Your Configuration (appsettings.json)                     │ │
│  │ Your Data (Databases, Files)                              │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Operating System (Windows Server / Linux)                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Virtual Machine (Compute, Storage, Networking)            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
│                     Microsoft Azure Data Center                     │
└─────────────────────────────────────────────────────────────────┘

You manage: OS updates, security patches, runtime, networking, scaling
Microsoft manages: Physical hardware, data center, network infrastructure
```

### Platform as a Service (PaaS)

```
PaaS (App Service):
┌─────────────────────────────────────────────────────────────────┐
│                        Your Responsibility                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Your Application                                           │ │
│  │ Your Configuration (appsettings.json)                       │ │
│  │ Your Data (Databases, Files)                               │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
│                        Microsoft Responsibility                     │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ App Service Plan (Auto-scaling, Load balancing)             │ │
│  │ Runtime (.NET 8, Node 20, Python 3.12)                      │ │
│  │ Operating System (Patched automatically)                    │ │
│  │ Virtual Machine (Compute, Storage, Networking)            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                     │
│                     Microsoft Azure Data Center                     │
└─────────────────────────────────────────────────────────────────┘

You manage: Your application code and data
Microsoft manages: Everything else — OS, runtime, scaling, patching
```

## Detailed Comparison

| Aspect | Azure Virtual Machine | Azure App Service |
|--------|----------------------|-------------------|
| **Abstraction Level** | IaaS — full OS access | PaaS — app-only deployment |
| **OS Management** | You patch and manage the OS | Microsoft manages OS and runtime |
| **ASP.NET Core Deploy** | Install .NET runtime, configure IIS/Nginx | Push code/container, done |
| **Scaling** | Manual or VM Scale Sets (slower) | Auto-scale rules, scale-out in seconds |
| **Startup Time** | Minutes (VM boot) | Seconds (app cold start) |
| **Custom Runtime** | Any — install whatever you need | Supported runtimes only (.NET, Node, Java, Python, Go) |
| **Networking** | Full VNET control, custom firewall | VNET Integration available, simpler |
| **Cost Model** | Pay for running VM 24/7 | Pay per App Service Plan (can host multiple apps) |
| **Free Tier** | (B1 is cheapest) | Free F1 tier available |
| **SSL/HTTPS** | Configure manually (Let's Encrypt) | Free managed certificate, one click |
| **Deployment** | RDP, SSH, custom scripts | GitHub Actions, Azure DevOps, zip deploy |
| **Observability** | Manual agent install | Application Insights built-in integration |
| **Maintenance burden** | High — your team owns everything | Low — Microsoft manages infrastructure |
| **High Availability** | Manual configuration | Built-in redundancy |
| **Disaster Recovery** | Manual setup | Backup/Restore features available |

## Decision Framework

### Choose Azure App Service When:

> You're building a standard web API or web app with ASP.NET Core. You want fast deployments from CI/CD. Your team doesn't have DevOps/sysadmin expertise. You want built-in scaling, SSL, and monitoring with minimal configuration.

**Ideal for:**
- Web applications and REST APIs
- Containerized microservices
- WebJobs for background processing
- Mobile app backends
- Quickly prototyping new applications

**Benefits:**
- Faster time to production
- Lower operational overhead
- Built-in auto-scaling
- Integrated monitoring with Application Insights
- Free SSL certificates
- Easy CI/CD integration

### Choose Azure VM When:

> You need a specific OS version or custom kernel module. You're running legacy software that requires specific system libraries. You need full network control (custom firewall rules, specific port bindings). You're running non-HTTP workloads (game servers, specialized compute).

**Ideal for:**
- Legacy applications requiring specific OS/versions
- Applications with non-standard runtime requirements
- Windows Server-specific features (COM+, MSMQ)
- Running desktop applications (like SQL Server Reporting Services)
- Complex networking configurations
- Fully custom infrastructure requirements

**Benefits:**
- Complete control over the environment
- Install anything you need
- Full Windows Server features
- Use existing licenses (BYOL)
- Complex third-party software

## Cost Comparison

```
Virtual Machine (D2s v3 - 2 vCPU, 8GB):
  - Pay hourly: ~$84/month (on-demand)
  - Reserved: ~$49/month (1-year)
  - Plus: Storage, networking, backups
  
App Service (P2v2 - 2 cores, 7GB):
  - Basic: ~$75/month (always on)
  - Standard: ~$150/month with auto-scale
  
Cost Analysis:
┌─────────────────────────────────────────────────────────────┐
│                    Monthly Cost Comparison                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  VM (on-demand):    ████████████████████████████ $84         │
│  VM (reserved):    ██████████████████ $49                  │
│  App Service (B):  █████████████████████████████ $75       │
│  App Service (S):  █████████████████████████████████ $150  │
│                                                             │
│  Note: App Service can host multiple apps per plan          │
│        VM hosts one app per VM                              │
└─────────────────────────────────────────────────────────────┘
```

## Typical Route Academy Project Scenario

Your ASP.NET Core Web API project -> **Azure App Service** is the correct choice:

1. **Deploy via GitHub Actions** — Simple YAML configuration
2. **Free SSL certificate** — One-click HTTPS
3. **Application Insights** — Built-in monitoring
4. **Auto-scale** — Handle traffic spikes automatically
5. **Lower maintenance** — Focus on code, not infrastructure

### Recommended App Service Configuration for .NET APIs

```
App Service Plan: P2v2 (Production)
  - 2 cores, 7GB RAM
  - Auto-scale: 2-10 instances
  - Always on: enabled

Configuration:
  - HTTPS only (redirect HTTP)
  - Application Insights enabled
  - ARR affinity: disabled (stateless API)
  - WebSockets: enabled (if needed)
```

## Migration Path

### From VM to App Service (Lift and Shift)

```
Phase 1: Assess
  - Inventory current VM configuration
  - Identify dependencies
  - Document environment variables

Phase 2: Prepare
  - Create App Service Plan
  - Configure Application Insights
  - Set up staging deployment slot

Phase 3: Migrate
  - Remove IIS dependencies
  - Update configuration for App Service
  - Deploy to staging slot
  - Test thoroughly

Phase 4: Go Live
  - Swap staging to production
  - Monitor application metrics
  - Rollback if needed (quick swap back)
```

## Decision Matrix

| Requirement | Recommended |
|-------------|-------------|
| .NET Web API | App Service |
| .NET Web App (MVC/Razor) | App Service |
| Containerized microservice | App Service (Containers) or AKS |
| Legacy .NET Framework app | VM (if requires IIS-specific features) |
| SQL Server on VM | VM (but consider Azure SQL) |
| Complex networking (VPN, firewall) | VM or App Service (VNET Integration) |
| Quick prototype | App Service (Free tier) |
| Development environment | App Service (Free tier) |
| Production enterprise app | App Service (Standard+) |
| Running SQL Server | VM (or Azure SQL DB) |

## References

- [Azure App Service Overview](https://learn.microsoft.com/en-us/azure/app-service/overview)
- [Azure Virtual Machines Overview](https://learn.microsoft.com/en-us/azure/virtual-machines/overview)
- [Reference Video — Azure App Service (YouTube)](https://www.youtube.com/watch?v=4BwyqmRTrx8)
- [Azure App Service Pricing](https://azure.microsoft.com/en-us/pricing/details/app-service/windows/)