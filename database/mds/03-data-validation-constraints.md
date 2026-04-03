# Data Validation & Constraints

## Table of Contents
1. [NO ACTION vs RESTRICT](#no-action-vs-restrict-in-postgresql)
2. [Name Validation with SQL](#name-validation)
3. [Password Validation](#password-validation)
4. [Email Validation](#email-validation)
5. [LIKE Operator Limitations](#like-operator-limitations)

---

## NO ACTION vs RESTRICT in PostgreSQL

### Overview
Both NO ACTION and RESTRICT are referential actions for foreign key constraints that prevent deleting or updating a parent row when child rows exist. They behave identically in PostgreSQL.

### NO ACTION
- **Default behavior** for foreign key constraints
- Checks constraint at end of transaction
- Allows deferring the constraint check
- Can be deferred until transaction commit

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id)
        ON DELETE NO ACTION  -- Default
);
```

### RESTRICT
- Prevents deletion immediately
- Cannot be deferred
- Error raised as soon as DELETE is attempted

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id)
        ON DELETE RESTRICT
);
```

### Key Differences

| Aspect | NO ACTION | RESTRICT |
|--------|-----------|----------|
| **Timing** | End of transaction | Immediate |
| **Deferrable** | Yes | No |
| **Behavior** | Same final result | Same final result |

### In PostgreSQL
PostgreSQL treats NO ACTION and RESTRICT identically for both ON DELETE and ON UPDATE. Both raise an error if referencing rows exist. The difference exists for SQL standard compatibility.

### Other Referential Actions
| Action | Behavior |
|--------|----------|
| **CASCADE** | Delete/update child rows automatically |
| **SET NULL** | Set foreign key to NULL |
| **SET DEFAULT** | Set to default value |

---

## Name Validation

### Requirement
Ensure that a name field does NOT contain any numeric digits (0-9).

### Using LIKE (Pattern Matching)
```sql
-- Check that name contains NO digits
WHERE Name NOT LIKE '%[0-9]%'

-- PostgreSQL version
WHERE Name !~ '[0-9]'
```

### Complete Validation Example
```sql
-- SQL Server
CREATE TABLE Users (
    UserID INT PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL
        CONSTRAINT CHK_Name_NoDigits 
        CHECK (Name NOT LIKE '%[0-9]%')
);

-- Insert validation
INSERT INTO Users (UserID, Name)
SELECT 1, 'John Doe'  -- Valid
WHERE 'John Doe' NOT LIKE '%[0-9]%';

INSERT INTO Users (UserID, Name)
SELECT 2, 'John123'   -- Invalid - blocked
WHERE 'John123' NOT LIKE '%[0-9]%';
```

### Using PATINDEX (SQL Server)
```sql
-- Returns 0 if no digits found, position if found
WHERE PATINDEX('%[0-9]%', Name) = 0
```

### Using Regular Expressions (PostgreSQL)
```sql
-- POSIX regular expression
WHERE Name !~ '[0-9]'

-- Or using SIMILAR TO
WHERE Name NOT SIMILAR TO '%[0-9]%'
```

---

## Password Validation

### Requirements
- At least one uppercase letter (A-Z)
- At least one lowercase letter (a-z)
- Contains digits (0-9)
- Contains at least one special character (!@#$%^&* etc.)
- Minimum length of 8 characters

### SQL Server Implementation
```sql
CREATE TABLE Users (
    UserID INT PRIMARY KEY,
    Password NVARCHAR(100) NOT NULL,
    
    CONSTRAINT CHK_Password_Complexity CHECK (
        LEN(Password) >= 8                              -- Minimum length
        AND Password LIKE '%[A-Z]%'                     -- Uppercase
        AND Password LIKE '%[a-z]%'                     -- Lowercase
        AND Password LIKE '%[0-9]%'                     -- Digit
        AND Password LIKE '%[!@#$%^&*()_+-=[]{}|;:,./<>?]%'  -- Special char
    )
);
```

### Breakdown of Each Condition

| Condition | Pattern | Purpose |
|-----------|---------|---------|
| Length | `LEN(Password) >= 8` | Minimum 8 characters |
| Uppercase | `LIKE '%[A-Z]%'` | At least one A-Z |
| Lowercase | `LIKE '%[a-z]%'` | At least one a-z |
| Digit | `LIKE '%[0-9]%'` | At least one 0-9 |
| Special | `LIKE '%[!@#$%...]%'` | At least one special |

### PostgreSQL Implementation
```sql
CREATE TABLE Users (
    UserID INT PRIMARY KEY,
    Password VARCHAR(100) NOT NULL,
    
    CONSTRAINT CHK_Password_Complexity CHECK (
        LENGTH(Password) >= 8
        AND Password ~ '[A-Z]'
        AND Password ~ '[a-z]'
        AND Password ~ '[0-9]'
        AND Password ~ '[!@#$%^&*()_+\-=\[\]{}|;:,./<>?]'
    )
);
```

### Important Security Note
**NEVER store plain-text passwords.** Always hash passwords using bcrypt, Argon2, or PBKDF2 before storage. The validation above should occur before hashing.

---

## Email Validation

### Requirements
- Does NOT start with a digit
- Contains @ symbol
- Contains a dot (.) after @
- Ends with valid domain extension (.com, .net, etc.)

### SQL Server Implementation
```sql
CREATE TABLE Users (
    UserID INT PRIMARY KEY,
    Email NVARCHAR(255) NOT NULL,
    
    CONSTRAINT CHK_Email_Format CHECK (
        -- Does not start with digit
        Email NOT LIKE '[0-9]%'
        -- Contains @
        AND Email LIKE '%@%'
        -- Contains . after @
        AND Email LIKE '%@%.%'
        -- Ends with valid domain
        AND Email LIKE '%.com'
            OR Email LIKE '%.net'
            OR Email LIKE '%.org'
            OR Email LIKE '%.edu'
            OR Email LIKE '%.io'
            OR Email LIKE '%.co%'
    )
);
```

### Enhanced Version with PATINDEX
```sql
-- More robust validation
WHERE PATINDEX('[0-9]%', Email) = 0           -- No leading digit
  AND PATINDEX('%@%.%', Email) > 0              -- Has @ and dot after
  AND Email NOT LIKE '%@%@%'                   -- Only one @
  AND RIGHT(Email, 4) LIKE '.___'             -- Ends with .xxx
```

### PostgreSQL with Regex
```sql
-- RFC 5322 simplified pattern
WHERE Email ~ '^[^0-9][a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
```

### Common Email Pattern
```sql
-- SQL Server: More complete validation
CREATE FUNCTION dbo.ValidateEmail(@email NVARCHAR(255))
RETURNS BIT
AS
BEGIN
    DECLARE @IsValid BIT = 0;
    
    IF @email IS NULL RETURN 0;
    
    -- Check: no leading digit, contains @, has domain, valid chars
    IF @email NOT LIKE '[0-9]%'
       AND @email LIKE '%@%.%'
       AND @email NOT LIKE '%@%@%'
       AND @email NOT LIKE '%..%'
       AND @email NOT LIKE '%@%[^a-zA-Z0-9.-]%'
       AND RIGHT(@email, 4) LIKE '.[a-zA-Z][a-zA-Z][a-zA-Z]'
    BEGIN
        SET @IsValid = 1;
    END
    
    RETURN @IsValid;
END;
```

---

## LIKE Operator Limitations

### Limitations of LIKE

#### 1. Limited Pattern Matching
- No support for alternation (OR patterns)
- Limited repetition quantifiers
- No capture groups or backreferences
- Character classes are basic ([a-z], [0-9])

#### 2. Performance Issues
- Cannot use indexes effectively with leading wildcards (`%text`)
- Full table scans required for pattern searches
- No optimization for complex patterns

#### 3. Validation Weaknesses
| Limitation | Example | Problem |
|------------|---------|---------|
| Cannot validate structure | `___-__-____` (SSN) | Complex patterns difficult |
| No quantifiers | Exactly 10 digits | Requires multiple LIKE checks |
| No anchors | Start/end validation | Hard to enforce position |
| Character exclusions | Only letters | `[^0-9]` not universally supported |

#### 4. Database-Specific Syntax
- Different escape characters across databases
- Pattern syntax varies (MySQL vs SQL Server vs PostgreSQL)
- POSIX vs T-SQL regex differences

### When to Use a Better Approach

#### Use Regular Expressions When:
- Complex validation rules needed
- Position-specific matching required
- Alternation (OR) patterns needed
- Precise quantification (`{3,5}`)
- Real-world email/phone validation

#### SQL Server: CLR or LIKE
```sql
-- For complex patterns, consider CLR integration
-- or use application-layer validation
```

#### PostgreSQL: Native Regex
```sql
-- PostgreSQL has superior regex support
WHERE column ~ '^[A-Za-z]{2,50}$'
WHERE column ~* '^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$'
```

#### Recommended Approach
| Validation Type | Recommended Method |
|-----------------|-------------------|
| Simple patterns | LIKE with PATINDEX |
| Complex validation | Application layer |
| Email/URL/Phone | Regex (PostgreSQL) or Application |
| Security-critical | Application layer + Database check |

### Best Practice: Application + Database
```csharp
// Application layer (C# example)
public bool IsValidEmail(string email)
{
    var regex = new Regex(@"^[^0-9][a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$");
    return regex.IsMatch(email);
}
```

```sql
-- Database layer (defense in depth)
CONSTRAINT CHK_Email_Basic CHECK (Email LIKE '%@%.%')
```

---

## Alternative: CHECK Constraints with Functions

### Creating a Validation Function
```sql
-- SQL Server: Scalar function for validation
CREATE FUNCTION dbo.IsValidPassword(@password NVARCHAR(100))
RETURNS BIT
AS
BEGIN
    IF LEN(@password) < 8 RETURN 0;
    IF @password NOT LIKE '%[A-Z]%' RETURN 0;
    IF @password NOT LIKE '%[a-z]%' RETURN 0;
    IF @password NOT LIKE '%[0-9]%' RETURN 0;
    IF @password NOT LIKE '%[!@#$%^&*()]%' RETURN 0;
    RETURN 1;
END;

-- Use in constraint
ALTER TABLE Users
ADD CONSTRAINT CHK_Password_Valid 
CHECK (dbo.IsValidPassword(Password) = 1);
```

---

*Source: SQL Server and PostgreSQL documentation, data validation best practices.*
