# Query Optimization Techniques

## Table of Contents
1. [Adaptive Join](#adaptive-join)
2. [Hash Spill vs Sort Spill](#hash-spill--sort-spill)
3. [OPTION (RECOMPILE)](#option-recompile)
4. [N+1 Query Problem](#n1-query-problem)
5. [OPENQUERY & OPENROWSET](#openquery--openrowset)

---

## Adaptive Join

### Overview
Adaptive Join is a SQL Server feature that delays the choice between Hash Join and Nested Loops until runtime, based on actual row counts.

### Why Adaptive Join Exists
Traditional query plans choose join type at compile time based on estimates. If estimates are wrong:
- **Overestimate**: Nested Loops chosen for large datasets (slow)
- **Underestimate**: Hash Join chosen for small datasets (unnecessary overhead)

### How It Works
```
Branch 1: If row count < threshold → Nested Loops
Branch 2: If row count >= threshold → Hash Match
          ↓
    [Adaptive Join operator decides at runtime]
```

### Requirements
- SQL Server 2017+
- Compatibility level 140+
- Batch mode execution
- Columnstore index involved OR
- Database setting: `SET QUERY_OPTIMIZER_HOTFIXES = ON`

### Enabling Adaptive Join
```sql
-- Check compatibility level
SELECT name, compatibility_level 
FROM sys.databases 
WHERE name = 'YourDatabase';

-- Set to SQL Server 2017+
ALTER DATABASE YourDatabase SET COMPATIBILITY_LEVEL = 140;

-- Enable batch mode on rowstore (SQL 2019+)
ALTER DATABASE YourDatabase 
SET BATCH_MODE_ON_ROWSTORE = ON;
```

### Identifying Adaptive Join in Plans
```sql
-- Look for "Adaptive Join" operator in execution plan
-- Shows both branches with estimated vs actual row counts

EXPLAIN ANALYZE
SELECT o.OrderID, c.CustomerName
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate >= '2024-01-01';
```

### When Adaptive Join Helps
- **Volatile data**: Statistics frequently outdated
- **Parameter sensitivity**: Different parameters yield different cardinalities
- **Complex queries**: Multiple estimates compound errors

---

## Hash Spill & Sort Spill

### What is a Spill?
When an operation requires more memory than granted, SQL Server spills data to tempdb (disk), dramatically slowing performance.

### Hash Spill

#### When It Happens
- Hash Join needs more memory than allocated
- Build side of hash join exceeds memory grant

#### Execution Plan Signs
```
Hash Match (Warning: Columns with Unequal Lengths)
    └── Warning: The query processor ... granted 1234 KB...
```

#### Causes
1. Underestimated row counts
2. Large tables with no filter
3. Many concurrent hash operations
4. Memory pressure system-wide

#### Prevention
```sql
-- 1. Update statistics
UPDATE STATISTICS LargeTable WITH FULLSCAN;

-- 2. Query hint for memory grant
SELECT * FROM LargeTable
OPTION (MIN_GRANT_PERCENT = 10, MAX_GRANT_PERCENT = 50);

-- 3. Batch mode for better estimates
CREATE COLUMNSTORE INDEX IX_CS ON LargeTable(Col1, Col2);
```

### Sort Spill

#### When It Happens
- ORDER BY requires more memory than granted
- Sort operator overflows memory

#### Execution Plan Signs
```
Sort (Warning: Columns with Unequal Lengths)
    └── Warning: Operation caused spill...
```

#### Prevention
```sql
-- 1. Covering index for sort
CREATE INDEX IX_Orders_Date ON Orders(OrderDate) INCLUDE (OrderID, Total);

-- 2. Limit result set
SELECT TOP 1000 * FROM Orders ORDER BY OrderDate;

-- 3. Avoid unnecessary sorts
-- Let optimizer use index order
```

### Monitoring Spills
```sql
-- Find queries with spills
SELECT 
    qs.plan_handle,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text,
    qs.total_spills,
    qs.last_spills
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qs.total_spills > 0
ORDER BY qs.total_spills DESC;
```

---

## OPTION (RECOMPILE)

### Purpose
Forces query optimizer to generate a new execution plan on every execution, discarding any cached plan.

### When to Use
1. **Parameter sniffing issues**: Plans optimized for one parameter are terrible for others
2. **Volatile data**: Statistics change frequently
3. **Complex queries**: Plan generation cost < execution cost
4. **Dynamic SQL**: Different shapes each execution

### Syntax
```sql
-- Stored procedure with recompile
CREATE PROCEDURE GetOrdersByDate @StartDate DATE, @EndDate DATE
AS
BEGIN
    SELECT * FROM Orders 
    WHERE OrderDate BETWEEN @StartDate AND @EndDate
    OPTION (RECOMPILE);
END;
```

### Statement-Level Recompile
```sql
-- Only recompile specific statement
CREATE PROCEDURE MixedWorkload
AS
BEGIN
    -- This uses cached plan
    SELECT * FROM StaticTable WHERE ID = 1;
    
    -- This recompiles every time
    SELECT * FROM VolatileTable 
    WHERE DateColumn >= DATEADD(day, -7, GETDATE())
    OPTION (RECOMPILE);
END;
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Optimal plan for each execution | CPU overhead of compilation |
| Solves parameter sniffing | Increased plan cache pressure |
| Accurate statistics every time | Not for high-frequency queries |
| Handles dynamic predicates | Compilation time measurable |

### Best Practices
```sql
-- Use for low-frequency, critical queries
-- Example: Report generation that runs hourly
CREATE PROCEDURE GenerateReport @StartDate DATE
AS
SELECT * FROM ComplexJoins
WHERE Date >= @StartDate
OPTION (RECOMPILE);

-- DON'T use for OLTP queries running 1000s of times/minute
```

---

## N+1 Query Problem

### The Problem
ORMs often execute one query for parent records, then N queries for related records.

```csharp
// N+1 Anti-pattern
var users = db.Users.ToList();           // Query 1: SELECT * FROM Users (100 users)
foreach (var user in users)
{
    Console.WriteLine(user.Orders.Count); // Query 2-101: SELECT * FROM Orders WHERE UserID = ?
}
// Total: 101 queries!
```

### Solutions

#### 1. Eager Loading (Include)
```csharp
// Single query with JOIN
var users = db.Users
    .Include(u => u.Orders)
    .ToList();
// Generates: SELECT ... FROM Users JOIN Orders...
```

#### 2. Projection (Select)
```csharp
// Load only needed data
var userSummaries = db.Users
    .Select(u => new {
        u.Name,
        OrderCount = u.Orders.Count()
    })
    .ToList();
```

#### 3. Batch Loading
```csharp
// Load users, then load all orders in one query
var users = db.Users.ToList();
var userIds = users.Select(u => u.Id).ToList();
var allOrders = db.Orders
    .Where(o => userIds.Contains(o.UserId))
    .ToList();
// Total: 2 queries
```

#### 4. Stored Procedure
```sql
-- Single query with aggregated data
CREATE PROCEDURE GetUsersWithOrderCounts
AS
BEGIN
    SELECT 
        u.UserID,
        u.UserName,
        COUNT(o.OrderID) AS OrderCount
    FROM Users u
    LEFT JOIN Orders o ON u.UserID = o.UserID
    GROUP BY u.UserID, u.UserName;
END;
```

### Detection
```sql
-- Find applications doing N+1
-- Look for repeated similar queries with different parameters
SELECT 
    SUBSTRING(text, (statement_start_offset/2)+1, 100) AS query_snippet,
    COUNT(*) AS execution_count
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)
WHERE text LIKE '%SELECT%FROM Orders%WHERE UserID%'
GROUP BY SUBSTRING(text, (statement_start_offset/2)+1, 100)
HAVING COUNT(*) > 1000;
```

---

## OPENQUERY & OPENROWSET

### OPENQUERY

#### Purpose
Executes a query on a linked server using the remote server's query processor.

#### Syntax
```sql
OPENQUERY(linked_server, 'query')
```

#### Example
```sql
-- Query data from linked Oracle server
SELECT * 
FROM OPENQUERY(ORACLE_PROD, 'SELECT * FROM HR.EMPLOYEES WHERE DEPT_ID = 10');

-- Join with local table
SELECT e.*, d.DepartmentName
FROM OPENQUERY(ORACLE_PROD, 'SELECT * FROM HR.EMPLOYEES') AS e
JOIN Departments d ON e.DEPT_ID = d.DeptID;
```

#### Advantages
- Remote server processes WHERE clauses
- Can use remote indexes
- Reduced data transfer

### OPENROWSET

#### Purpose
Ad-hoc connection to remote data source without pre-configured linked server.

#### Syntax
```sql
OPENROWSET(provider, connection_string, query)
```

#### Example
```sql
-- Bulk import from file
SELECT * 
FROM OPENROWSET(
    BULK 'C:\Data\Import.csv',
    FORMATFILE = 'C:\Data\ImportFormat.xml'
) AS data;

-- Query remote SQL Server
SELECT *
FROM OPENROWSET(
    'SQLNCLI',
    'Server=RemoteServer;Trusted_Connection=yes;',
    'SELECT * FROM SalesDB.dbo.Orders'
);
```

### Comparison

| Aspect | OPENQUERY | OPENROWSET |
|--------|-----------|------------|
| **Linked Server Required** | Yes | No |
| **Security** | Uses linked server credentials | Ad-hoc credentials |
| **Performance** | Better (uses remote processing) | May pull all data |
| **Flexibility** | Limited to linked servers | Any connection string |

### Security Considerations
```sql
-- OPENROWSET requires specific permissions
-- Must enable: sp_configure 'Ad Hoc Distributed Queries', 1

-- Safer approach: Use linked servers with stored credentials
-- Avoid ad-hoc connections in production
```

### When to Use

**OPENQUERY**:
- Regular cross-server queries
- Joining remote and local data
- Leveraging remote server processing

**OPENROWSET**:
- One-time data imports
- Bulk operations
- Dynamic connection requirements

### Best Practice
```sql
-- Use OPENQUERY for joins to push predicates
-- BAD: Pulls all data, filters locally
SELECT * FROM OPENQUERY(REMOTE, 'SELECT * FROM HugeTable')
WHERE DateColumn > '2024-01-01';

-- GOOD: Filters on remote server
SELECT * FROM OPENQUERY(REMOTE, 
    'SELECT * FROM HugeTable WHERE DateColumn > ''2024-01-01''');
```

---

## Summary Table

| Technique | Problem Solved | Trade-off |
|-----------|--------------|-----------|
| **Adaptive Join** | Suboptimal join choice | Requires batch mode |
| **Reduce Spills** | Memory pressure | Requires more memory |
| **OPTION (RECOMPILE)** | Parameter sniffing | CPU overhead |
| **Eager Loading** | N+1 queries | Larger initial query |
| **OPENQUERY** | Remote data access | Requires linked server |

---

*Source: SQL Server query optimization documentation, ORM best practices, and distributed query patterns.*
