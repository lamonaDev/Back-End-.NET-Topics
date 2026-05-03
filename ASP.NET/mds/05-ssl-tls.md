# SSL / TLS — How It Works

## Overview

**TLS (Transport Layer Security)** is the cryptographic protocol that secures HTTP communications. "SSL" is the older name — technically we use TLS 1.2 or TLS 1.3 today. When you see `https://`, TLS is in play.

TLS provides three guarantees:
- **Confidentiality**: Data is encrypted in transit — no eavesdropping
- **Integrity**: Data cannot be tampered with undetected
- **Authentication**: You're talking to the real server, not an impersonator

## TLS Versions

| Version | Status | Notes |
|---------|--------|-------|
| SSL 2.0 | Deprecated | Contains severe vulnerabilities |
| SSL 3.0 | Deprecated | POODLE attack vulnerability |
| TLS 1.0 | Deprecated | Vulnerable to BEAST attack |
| TLS 1.1 | Deprecated | Multiple security issues |
| TLS 1.2 | Supported | Current minimum standard |
| TLS 1.3 | Current | Faster, more secure |

Always use TLS 1.2 or higher in production.

## TLS Handshake — Step by Step

The TLS handshake is the process where the client and server establish a secure connection:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     TLS Handshake Process                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Client ──────────── ClientHello ───────────→ Server               │
│     (TLS version, cipher suites, random nonce)                        │
│                                                                         │
│  2. Client ←────────── ServerHello ──────────── Server               │
│     (Selected cipher, server nonce)                                   │
│                                                                         │
│  3. Client ←────────── Certificate ──────────── Server               │
│     (X.509 certificate chain)                                         │
│                                                                         │
│  4. Client ←────────── ServerKeyExchange ────── Server               │
│     (Optional: parameters for key exchange)                           │
│                                                                         │
│  5. Client ←────────── CertificateRequest ───── Server              │
│     (Optional: server requests client cert)                          │
│                                                                         │
│  6. Client ←────────── ServerHelloDone ───────── Server               │
│                                                                         │
│  7. Client ─────────── ClientKeyExchange ──────→ Server              │
│     (Pre-master secret, encrypted with server's public key)          │
│                                                                         │
│  8. Client ─────────── Certificate ─────────────→ Server             │
│     (Optional: client certificate)                                    │
│                                                                         │
│  9. Client ─────────── CertificateVerify ──────→ Server             │
│     (Optional: prove client owns private key)                         │
│                                                                         │
│ 10. Client ─────────── ChangeCipherSpec ────────→ Server             │
│     (Both sides now use new cipher)                                   │
│                                                                         │
│ 11. Client ═══════════ Encrypted Data ════════════ Server             │
│     (Secure communication begins)                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Handshake Time Cost

- **TLS 1.2**: Requires 2-3 round trips to complete
- **TLS 1.3**: Reduces to 1 round trip (0-RTT resumption possible)

This is why HTTP/3 with TLS 1.3 is significantly faster for new connections.

## Certificate Trust Chain

Your browser trusts a certificate only if it chains up to a **root CA** it already trusts (stored in the OS trust store). The chain is:

```
Root CA (Trusted by OS)
    │
    └── Intermediate CA 1
            │
            └── Intermediate CA 2
                    │
                    └── Server Certificate (your domain)
```

### Certificate Types

| Type | What It Validates | Use Case |
|------|-------------------|----------|
| Domain Validation (DV) | You control the domain | Personal sites, blogs |
| Organization Validation (OV) | Domain + organization identity | Business websites |
| Extended Validation (EV) | Rigorous identity verification | E-commerce, banking |
| Self-signed | No CA validation | Development only |

**Let's Encrypt** is a free, automated, trusted CA that provides DV certificates, used by millions of sites.

## Without TLS — Attack Scenarios

| Attack | What Happens | Impact |
|--------|--------------|--------|
| Eavesdropping | Attacker on same Wi-Fi reads HTTP packets | Passwords, tokens, credit cards stolen |
| Man-in-the-Middle | Attacker intercepts and modifies responses | Injects malicious scripts, phishing |
| Session Hijacking | Steals session cookie from plain HTTP | Account takeover without login |
| Credential Replay | Captures login POST, replays it | Unauthorized access |
| Data Modification | Attacker changes data in transit | Corrupted data, false transactions |

### Real Incident Example

In 2010, the **Firesheep** browser extension demonstrated live session hijacking on Facebook and Twitter over public Wi-Fi. Any coffee shop patron could one-click hijack another user's account because both sites served their post-login pages over plain HTTP. This forced both companies to adopt HTTPS everywhere within months. The attack required zero technical skill — just a Firefox extension and an open network.

## Impact on Login / Session / Token Security

- **JWT tokens** sent over HTTP can be stolen and replayed indefinitely
- **Session cookies** — always set `Secure` flag; browser won't send over HTTP
- **Refresh tokens** are long-lived — one theft = persistent account access
- **CORS** and **CSRF** protections are weakened without HTTPS

### Cookie Security Headers

```csharp
// Set secure cookies
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
options.Cookie.SameSite = SameSiteMode.Strict;
options.Cookie.HttpOnly = true;
options.Cookie.Name = "YourApp.Session";
```

## Operational Concerns

| Concern | Detail | Action |
|---------|--------|--------|
| Certificate Expiry | Certs expire (90 days for Let's Encrypt) | Automate renewal with Certbot or Azure App Service managed certs |
| HSTS | HTTP Strict Transport Security header forces HTTPS | Add `app.UseHsts()` in production |
| Mixed Content | HTTPS page loading HTTP resources | Ensure all assets served over HTTPS |
| TLS Version | TLS 1.0/1.1 deprecated — insecure | Enforce TLS 1.2+ minimum |
| Dev Certificate | ASP.NET Core dev cert is self-signed | `dotnet dev-certs https --trust` |
| Certificate Pinning | Binding certificate to prevent MITM | For mobile apps and high-security apps |

## HSTS (HTTP Strict Transport Security)

HSTS tells browsers to only connect via HTTPS for a specified duration:

```
Server sends: Strict-Transport-Security: max-age=31536000; includeSubDomains
Browser remembers: For one year, only connect via HTTPS to this domain
```

### Program.cs — HTTPS redirect + HSTS

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts(); // Sends Strict-Transport-Security header
}
app.UseHttpsRedirection(); // Redirect HTTP → HTTPS (301)
```

### Configure HSTS in appsettings.json

```json
{
  "Hsts": {
    "Enabled": true,
    "MaxAge": 31536000,
    "IncludeSubDomains": true,
    "Preload": true
  }
}
```

## ASP.NET Core Development Certificate

### Trust the Dev Certificate

```bash
# Windows
dotnet dev-certs https --trust

# macOS
dotnet dev-certs https --trust

# Linux (browser-specific)
dotnet dev-certs https -p/your/key/path
```

### Clean Up and Regenerate

```bash
# Clean existing certs
dotnet dev-certs https --clean

# Generate new cert
dotnet dev-certs https
```

## Best Practices Checklist

- Always use TLS 1.2 or higher in production
- Enable HSTS with a long max-age
- Set cookie `Secure` flag always
- Redirect HTTP to HTTPS in production
- Use Let's Encrypt or Azure managed certificates
- Automate certificate renewal
- Check for mixed content warnings
- Implement certificate transparency logging

## References

- [Microsoft Docs — Enforce HTTPS in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/enforcing-ssl)
- [The Illustrated TLS 1.3 Connection](https://tls.ulfheim.net/)
- [Let's Encrypt — How It Works](https://letsencrypt.org/how-it-works/)