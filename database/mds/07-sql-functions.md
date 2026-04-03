# SQL Server Functions Reference

## Table of Contents
1. [DATEFROMPARTS](#datefromparts)
2. [EOMONTH](#eomonth)
3. [LEN vs DATALENGTH](#len-vs-datalength)
4. [LEFT Function](#left-function)
5. [CONCAT_WS](#concat_ws)
6. [DATEDIFF vs DATEADD](#datediff-vs-dateadd)

---

## DATEFROMPARTS

### Purpose
Constructs a DATE value from integer year, month, and day components.

### Syntax
```sql
DATEFROMPARTS(year, month, day)
```

### Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| year | INT | Year value (1-9999) |
| month | INT | Month value (1-12) |
| day | INT | Day value (1-31) |

### Return Type
**DATE** - Returns NULL if any parameter is invalid.

### Examples
```sql
-- Create specific date
SELECT DATEFROMPARTS(2024, 6, 15) AS MyDate;
-- Result: 2024-06-15

-- Beginning of month
SELECT DATEFROMPARTS(2024, 6, 1) AS MonthStart;
-- Result: 2024-06-01

-- Dynamic current year and month
SELECT DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1) AS FirstOfMonth;
```

### Use Cases

#### Get First Day of Month
```sql
-- First day of current month
SELECT DATEFROMPARTS(YEAR(GETDATE()), MONTH(GETDATE()), 1);

-- First day of any month variable
DECLARE @Year INT = 2024;
DECLARE @Month INT = 6;
SELECT DATEFROMPARTS(@Year, @Month, 1);
```

#### Date Range Queries
```sql
-- Filter by specific month
DECLARE @Year INT = 2024;
DECLARE @Month INT = 6;

SELECT * FROM Orders
WHERE OrderDate >= DATEFROMPARTS(@Year, @Month, 1)
  AND OrderDate < DATEFROMPARTS(@Year, @Month + 1, 1);
```

### Comparison with Alternative Methods
```sql
-- DATEFROMPARTS (SQL Server 2012+)
WHERE OrderDate >= DATEFROMPARTS(2024, 6, 1);

-- CONVERT method (older versions)
WHERE OrderDate >= CONVERT(DATE, '2024-06-01', 120);

-- String concatenation (not recommended)
WHERE OrderDate >= CAST('2024-' + CAST(6 AS VARCHAR) + '-01' AS DATE);
```

---

## EOMONTH

### Purpose
Returns the last day of the month containing the specified date, with optional offset.

### Syntax
```sql
EOMONTH(start_date [, month_to_add])
```

### Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| start_date | DATE/DATETIME | Starting date |
| month_to_add | INT | Optional months to add (default 0) |

### Return Type
**DATE**

### Examples
```sql
-- Last day of current month
SELECT EOMONTH(GETDATE()) AS EndOfMonth;

-- Last day of specific month
SELECT EOMONTH('2024-02-15') AS FebruaryEnd;
-- Result: 2024-02-29 (leap year)

-- Last day of next month
SELECT EOMONTH(GETDATE(), 1) AS EndOfNextMonth;

-- Last day of previous month
SELECT EOMONTH(GETDATE(), -1) AS EndOfPreviousMonth;

-- Last day of same month last year
SELECT EOMONTH(GETDATE(), -12) AS EndOfMonthLastYear;
```

### Use Cases

#### Complete Month Date Range
```sql
DECLARE @TargetMonth DATE = '2024-06-15';

SELECT 
    DATEFROMPARTS(YEAR(@TargetMonth), MONTH(@TargetMonth), 1) AS MonthStart,
    EOMONTH(@TargetMonth) AS MonthEnd;
```

#### Monthly Reporting
```sql
-- Get all orders for a specific month
DECLARE @ReportMonth DATE = '2024-06-01';

SELECT 
    OrderID,
    OrderDate,
    TotalAmount
FROM Orders
WHERE OrderDate >= DATEFROMPARTS(YEAR(@ReportMonth), MONTH(@ReportMonth), 1)
  AND OrderDate <= EOMONTH(@ReportMonth);
```

#### Financial Period Calculations
```sql
-- Days remaining in month
SELECT 
    GETDATE() AS Today,
    EOMONTH(GETDATE()) AS MonthEnd,
    DATEDIFF(DAY, GETDATE(), EOMONTH(GETDATE())) AS DaysRemaining;

-- Total days in month
SELECT 
    DAY(EOMONTH(GETDATE())) AS DaysInCurrentMonth;
```

---

## LEN vs DATALENGTH

### LEN Function

**Purpose**: Returns the number of characters in a string, excluding trailing spaces.

**Syntax**:
```sql
LEN(string)
```

**Return Type**: INT

**Important Characteristics:**
- Ignores trailing spaces
- Returns NULL if input is NULL
- For VARCHAR: counts characters
- For NVARCHAR: counts characters (same as LEN for Unicode)

```sql
-- LEN examples
SELECT LEN('Hello World')      -- Result: 11
SELECT LEN('Hello World   ')   -- Result: 11 (trailing spaces ignored)
SELECT LEN('   Hello')        -- Result: 8 (leading spaces counted)
SELECT LEN(N'Hello')          -- Result: 5 (Unicode)
```

### DATALENGTH Function

**Purpose**: Returns the number of bytes used to represent any expression.

**Syntax**:
```sql
DATALENGTH(expression)
```

**Return Type**: INT

**Important Characteristics:**
- Includes trailing spaces
- Returns NULL if input is NULL
- For VARCHAR: counts bytes
- For NVARCHAR: counts bytes (2 bytes per character)

```sql
-- DATALENGTH examples
SELECT DATALENGTH('Hello')           -- Result: 5
SELECT DATALENGTH('Hello   ')         -- Result: 8 (includes trailing spaces)
SELECT DATALENGTH(N'Hello')           -- Result: 10 (Unicode = 2 bytes/char)
SELECT DATALENGTH(12345)              -- Result: 4 (INT size)
SELECT DATALENGTH(GETDATE())          -- Result: 8 (DATETIME size)
```

### Comparison Table

| Aspect | LEN | DATALENGTH |
|--------|-----|------------|
| **Unit** | Characters | Bytes |
| **Trailing spaces** | Excluded | Included |
| **Unicode handling** | Character count | Byte count (×2) |
| **Use with** | Strings | Any data type |
| **NULL handling** | Returns NULL | Returns NULL |

### Practical Examples

#### Check for Trailing Spaces
```sql
-- Find strings with trailing spaces
SELECT CustomerName
FROM Customers
WHERE DATALENGTH(CustomerName) > LEN(CustomerName);
```

#### Calculate String Size in Storage
```sql
-- Storage size of NVARCHAR column
SELECT 
    CustomerName,
    LEN(CustomerName) AS CharCount,
    DATALENGTH(CustomerName) AS ByteCount,
    DATALENGTH(CustomerName) / 2 AS CharacterCountUnicode;
```

#### Trim and Compare
```sql
-- Safe comparison (handles trailing spaces)
SELECT *
FROM Users
WHERE LEN(RTRIM(Username)) = LEN(@Input);
```

---

## LEFT Function

### Purpose
Returns the specified number of characters from the beginning (left side) of a string.

### Syntax
```sql
LEFT(string, length)
```

### Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| string | VARCHAR/NVARCHAR | Source string |
| length | INT | Number of characters to return |

### Return Type
VARCHAR or NVARCHAR (matches input type)

### Examples
```sql
-- Basic usage
SELECT LEFT('Hello World', 5)    -- Result: 'Hello'

-- Extract first name
SELECT LEFT('John Smith', 4)     -- Result: 'John'

-- With column data
SELECT LEFT(ProductName, 10) AS ShortName FROM Products;

-- When length exceeds string
SELECT LEFT('Hi', 10)            -- Result: 'Hi' (no padding)
```

### Common Use Case: Extract First Name

```sql
-- From "LastName, FirstName" format
SELECT 
    DisplayName,
    LEFT(DisplayName, CHARINDEX(' ', DisplayName) - 1) AS FirstName
FROM Users
WHERE DisplayName LIKE '% %';

-- From "FirstName LastName" format (safer version)
SELECT 
    DisplayName,
    CASE 
        WHEN CHARINDEX(' ', DisplayName) > 0 
        THEN LEFT(DisplayName, CHARINDEX(' ', DisplayName) - 1)
        ELSE DisplayName
    END AS FirstName
FROM Users;
```

### Extract Initials
```sql
SELECT 
    FirstName,
    LastName,
    LEFT(FirstName, 1) + LEFT(LastName, 1) AS Initials
FROM Employees;
-- Result: 'JD' for John Doe
```

### Safe Extraction with NULL Handling
```sql
SELECT 
    COALESCE(LEFT(ProductName, 20), 'N/A') AS ShortName
FROM Products;
```

---

## CONCAT_WS

### Purpose
**Concatenate With Separator**: Concatenates multiple strings with a specified separator between each.

### Syntax
```sql
CONCAT_WS(separator, string1, string2, ..., stringN)
```

### Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| separator | VARCHAR/NVARCHAR | Separator string placed between values |
| string1...N | Any | Values to concatenate |

### Return Type
VARCHAR or NVARCHAR

### Key Features
- **Ignores NULL values** (doesn't add extra separators)
- **Automatically handles type conversion**
- **Separator not added at beginning or end**

### Examples
```sql
-- Basic concatenation
SELECT CONCAT_WS(' ', 'Hello', 'World');     -- Result: 'Hello World'

-- Multiple values
SELECT CONCAT_WS(', ', 'Apple', 'Banana', 'Cherry');
-- Result: 'Apple, Banana, Cherry'

-- NULL handling (ignored)
SELECT CONCAT_WS(', ', 'A', NULL, 'B', NULL, 'C');
-- Result: 'A, B, C' (no extra commas for NULLs)

-- Full name construction
SELECT CONCAT_WS(' ', FirstName, MiddleName, LastName) AS FullName
FROM Users;
-- If MiddleName is NULL: 'John Smith' (not 'John  Smith')
```

### Use Cases

#### CSV Generation
```sql
-- Create CSV-style output
SELECT 
    CONCAT_WS(',', 
        CAST(UserID AS VARCHAR),
        Email,
        CONVERT(VARCHAR, CreatedDate, 120)
    ) AS CSVRow
FROM Users;
```

#### Address Formatting
```sql
SELECT 
    CONCAT_WS(', ', 
        StreetAddress,
        City,
        State,
        PostalCode,
        Country
    ) AS FormattedAddress
FROM Addresses;
-- Skips any NULL components gracefully
```

#### Dynamic URL Building
```sql
SELECT 
    CONCAT_WS('/',
        'https:',
        '',  -- Creates // after https:
        'api.example.com',
        'v1',
        'users',
        CAST(UserID AS VARCHAR)
    ) AS APIUrl
FROM Users;
```

### Comparison with CONCAT

```sql
-- CONCAT: NULL becomes empty string, no separator
SELECT CONCAT('A', NULL, 'B');           -- Result: 'AB'

-- CONCAT_WS: NULL ignored, separator added
SELECT CONCAT_WS('-', 'A', NULL, 'B');   -- Result: 'A-B'

-- Traditional method: more verbose
SELECT ISNULL(Column1 + ', ', '') + ISNULL(Column2, '')
```

---

## DATEDIFF vs DATEADD

### DATEDIFF Function

**Purpose**: Returns the difference between two dates in the specified datepart.

**Syntax**:
```sql
DATEDIFF(datepart, startdate, enddate)
```

**Dateparts**:
- YEAR (yy, yyyy), QUARTER (qq, q), MONTH (mm, m)
- DAYOFYEAR (dy, y), DAY (dd, d), WEEK (wk, ww)
- HOUR (hh), MINUTE (mi, n), SECOND (ss, s), MILLISECOND (ms)

**Important Behavior**:
DATEDIFF counts boundary crossings, not actual duration.

```sql
-- Year difference (boundary-based)
SELECT DATEDIFF(YEAR, '2022-12-31', '2023-01-01');
-- Result: 1 (crossed year boundary)
-- Actual days: 1 day

-- Month difference
SELECT DATEDIFF(MONTH, '2024-01-31', '2024-02-28');
-- Result: 1 (crossed month boundary)
```

### DATEADD Function

**Purpose**: Adds a specified time interval to a date.

**Syntax**:
```sql
DATEADD(datepart, number, date)
```

**Examples**:
```sql
-- Add days
SELECT DATEADD(DAY, 30, GETDATE());      -- 30 days from now

-- Add months
SELECT DATEADD(MONTH, 3, '2024-01-15'); -- Result: 2024-04-15

-- Subtract time
SELECT DATEADD(HOUR, -2, GETDATE());    -- 2 hours ago
```

### Converting DATEDIFF to DATEADD Pattern

**The Problem**: 
```sql
-- DATEDIFF counts year boundaries, not actual years
SELECT DATEDIFF(YEAR, '2022-12-31', '2023-01-01');  -- Returns 1
-- But only 1 day apart!
```

**DATEADD Approach for Accurate Calculation**:
```sql
-- Calculate age accurately
SELECT 
    BirthDate,
    GETDATE() AS Today,
    DATEDIFF(YEAR, BirthDate, GETDATE()) AS SimpleAge,  -- Often wrong!
    CASE 
        WHEN DATEADD(YEAR, DATEDIFF(YEAR, BirthDate, GETDATE()), BirthDate) > GETDATE()
        THEN DATEDIFF(YEAR, BirthDate, GETDATE()) - 1
        ELSE DATEDIFF(YEAR, BirthDate, GETDATE())
    END AS AccurateAge
FROM Users;
```

### Practical Examples

#### First Day of Year (DATEADD conversion)
```sql
-- Using DATEFROMPARTS (recommended)
SELECT DATEFROMPARTS(2024, 1, 1);

-- Using DATEADD pattern
SELECT DATEADD(YEAR, DATEDIFF(YEAR, 0, GETDATE()), 0);
-- 0 = 1900-01-01, finds first day of current year
```

#### First Day of Quarter
```sql
-- Using DATEADD
SELECT DATEADD(QUARTER, DATEDIFF(QUARTER, 0, GETDATE()), 0);

-- Alternative calculation
SELECT DATEFROMPARTS(
    YEAR(GETDATE()),
    ((MONTH(GETDATE()) - 1) / 3) * 3 + 1,
    1
);
```

### Summary Comparison

| Aspect | DATEDIFF | DATEADD |
|--------|----------|---------|
| **Purpose** | Calculate difference | Add/subtract interval |
| **Returns** | Integer (count) | Date/datetime |
| **Calculation** | Boundary crossings | Actual time arithmetic |
| **Use case** | Duration in units | Date manipulation |

---

## Quick Reference Card

| Function | Purpose | Example |
|----------|---------|---------|
| **DATEFROMPARTS** | Build date from parts | `DATEFROMPARTS(2024, 6, 15)` |
| **EOMONTH** | Last day of month | `EOMONTH(GETDATE())` |
| **LEN** | Character count | `LEN('Text')` |
| **DATALENGTH** | Byte size | `DATALENGTH(N'Text')` |
| **LEFT** | Extract from left | `LEFT(Name, 10)` |
| **CONCAT_WS** | Join with separator | `CONCAT_WS(', ', A, B, C)` |
| **DATEDIFF** | Date difference | `DATEDIFF(DAY, Start, End)` |
| **DATEADD** | Add interval | `DATEADD(MONTH, 1, Date)` |

---

*Source: SQL Server function documentation, T-SQL reference, and practical query optimization.*
