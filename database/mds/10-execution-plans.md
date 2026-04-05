# Execution Plans Deep Dive

## Table of Contents
1. [Execution Plan Fundamentals](#execution-plan-fundamentals)
2. [Estimated vs Actual Plans](#estimated-vs-actual-execution-plans)
3. [Common Operators](#common-execution-plan-operators)
4. [Reading Execution Plans](#reading-execution-plans)
5. [Parameter Sniffing](#parameter-sniffing)

---

## Execution Plan Fundamentals

### What is an Execution Plan?
A roadmap that SQL Server creates to execute a query efficiently, showing:
- Data access methods
- Join strategies
- Filter operations
- Sorting and aggregation
- Estimated costs

### Why Execution Plans Exist
1. **SQL is declarative**: You say *what* you want, not *how*
2. **Query Optimizer** determines the most efficient way
3. Multiple equivalent queries may have different plans
4. Statistics drive plan selection

### Plan Generation Process
```
Query → Parse → Bind → Optimize → Execute
              ↓
         Query Optimizer
              ↓
    Estimates, Statistics, Cost Model
              ↓
         Execution Plan
```

### Types of Plans

| Plan Type | Description | Use Case |
|-----------|-------------|----------|
| **Estimated** | Compiled plan without execution | Query development |
| **Actual** | Plan with runtime statistics | Performance tuning |
| **Cached** | Stored in plan cache | Plan reuse |
| **Trivial** | Simple plan, no optimization | Very simple queries |

---

## Estimated vs Actual Execution Plans

### Estimated Execution Plan
**Generated at compile time**, based on:
- Table statistics
- Index information
- Cardinality estimates

**Characteristics:**
- Fast to generate
- Uses statistical estimates
- May differ from actual execution
- No actual row counts

```sql
-- Show estimated plan
SET SHOWPLAN_XML ON;
GO
SELECT * FROM Users WHERE UserID = 1;
GO
SET SHOWPLAN_XML OFF;
```

### Actual Execution Plan
**Generated during execution**, includes:
- Real row counts
- Actual execution time
- Memory grants
- Actual vs estimated differences

**Characteristics:**
- Requires query execution
- Shows true performance metrics
- Identifies estimation errors
- Essential for troubleshooting

```sql
-- Show actual plan
SET STATISTICS XML ON;
GO
SELECT * FROM Users WHERE UserID = 1;
GO
SET STATISTICS XML OFF;
```

### Key Differences

| Aspect | Estimated | Actual |
|--------|-----------|--------|
| **Row counts** | Estimated | Actual |
| **Runtime** | Predicted | Measured |
| **Requires execution** | No | Yes |
| **Memory grants** | Estimated | Actual |
| **CPU/IO** | Estimated | Actual |

### Plan Cache
SQL Server stores compiled plans for reuse:
```sql
-- View cached plans
SELECT 
    qp.query_plan,
    qs.execution_count,
    qs.total_elapsed_time / 1000.0 AS total_duration_ms
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qs.execution_count > 100;
```

---

## Common Execution Plan Operators

### Data Access Operators

#### Table Scan
**When**: No index on filter/sort columns
**Cost**: High - reads every row
**Visual**: Full table icon

```sql
-- Table Scan
SELECT * FROM Users WHERE LEN(UserName) > 5;
-- Function prevents index usage
```

#### Index Scan
**When**: Index exists but query needs many rows
**Cost**: Medium - reads entire index
**Visual**: Index icon with "Scan" label

```sql
-- Index Scan (if 90% of rows match)
SELECT * FROM Users WHERE Status = 'Active';
-- If most users are active, scan is efficient
```

#### Index Seek
**When**: Index exists and query is selective
**Cost**: Low - seeks directly to data
**Visual**: Index icon with "Seek" label

```sql
-- Index Seek
SELECT * FROM Users WHERE UserID = 12345;
-- Specific value, uses index tree
```

#### Key Lookup
**When**: Nonclustered index doesn't cover all columns
**Cost**: Additional bookmark lookup
**Visual**: Nested loops join to clustered index

```sql
-- Non-covering index
CREATE INDEX IX_Users_Status ON Users(Status);

-- Requires Key Lookup for Email
SELECT Email FROM Users WHERE Status = 'Active';
```

### Join Operators

#### Nested Loops Join
**When**: Small outer table, indexed inner table
**Cost**: Low for small datasets
**Process**: For each outer row, seek inner table

```sql
-- Nested Loops ideal for:
SELECT u.*, p.Title
FROM Users u  -- Small table
JOIN Posts p ON p.OwnerUserId = u.Id  -- Indexed
WHERE u.Reputation > 1000;  -- Few users
```

#### Hash Match Join
**When**: Large tables, no suitable indexes
**Cost**: High memory, builds hash table
**Process**: Hash build input, probe with other table

```sql
-- Hash Match for large joins
SELECT o.*, c.CustomerName
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
-- Both tables large, no selective filters
```

#### Merge Join
**When**: Both tables sorted on join key
**Cost**: Low, streaming operation
**Process**: Merge two sorted streams

```sql
-- Merge Join with sorted inputs
SELECT o.OrderID, o.OrderDate, s.ShipmentDate
FROM Orders o
JOIN Shipments s ON o.OrderID = s.OrderID
ORDER BY o.OrderID;
-- Both inputs sorted on OrderID
```

### Other Common Operators

| Operator | Purpose | When to Expect |
|----------|---------|----------------|
| **Sort** | ORDER BY, DISTINCT | No index on sort column |
| **Stream Aggregate** | COUNT, SUM, AVG | Pre-sorted data |
| **Hash Aggregate** | COUNT, SUM, AVG | Unsorted data |
| **Filter** | WHERE predicates | Post-scan filtering |
| **Compute Scalar** | Calculations | Expressions in SELECT |
| **Parallelism** | Multi-threading | Large operations |

---

## Reading Execution Plans

### Plan Flow Direction
Data flows **right to left**, top to bottom:
```
        SELECT (Result)
            |
    Nested Loops Join
       /         \
Index Seek   Key Lookup
(User)         (Email)
```

### Cost Percentages
- **Relative cost**: Percentage of total query
- **Not absolute**: Higher cost doesn't always mean slower
- **Focus on actual**: Compare estimated vs actual rows

### Warning Signs

#### Estimation Errors
```
Estimated: 100 rows
Actual: 1,000,000 rows  ← Problem!
```

#### Memory Grants
```
Granted: 4GB
Used: 50MB  ← Overestimation
```

#### Operator Warnings
- **Spills to tempdb**: Insufficient memory
- **Implicit conversion**: Data type mismatch
- **Columns without statistics**: Missing stats

### Plan Analysis Checklist
1. Check for scans vs seeks
2. Verify join types are appropriate
3. Compare estimated vs actual rows
4. Look for implicit conversions
5. Check for sort operations
6. Review memory grants
7. Identify spilling operators

---

## Parameter Sniffing

### What is Parameter Sniffing?
SQL Server creates execution plans based on the **first** parameter values it sees, which may be suboptimal for subsequent executions.

### The Problem Scenario
```sql
-- Procedure compiled with @Status = 'Active'
-- 90% of users are Active → Table Scan chosen

-- Later call with @Status = 'Admin'
-- Only 5 users are Admin → Scan is terrible!

CREATE PROCEDURE GetUsersByStatus @Status VARCHAR(20)
AS
SELECT * FROM Users WHERE Status = @Status;
```

### Symptoms
- Same procedure, vastly different performance
- Fast sometimes, slow other times
- Plans show different operators for same query

### Solutions

#### 1. OPTION (RECOMPILE)
```sql
CREATE PROCEDURE GetUsersByStatus @Status VARCHAR(20)
AS
SELECT * FROM Users 
WHERE Status = @Status
OPTION (RECOMPILE);
-- Generates new plan each execution
```

#### 2. OPTION (OPTIMIZE FOR UNKNOWN)
```sql
CREATE PROCEDURE GetUsersByStatus @Status VARCHAR(20)
AS
SELECT * FROM Users 
WHERE Status = @Status
OPTION (OPTIMIZE FOR UNKNOWN);
-- Uses average statistics, not specific values
```

#### 3. Local Variables
```sql
CREATE PROCEDURE GetUsersByStatus @Status VARCHAR(20)
AS
BEGIN
    DECLARE @LocalStatus VARCHAR(20) = @Status;
    SELECT * FROM Users WHERE Status = @LocalStatus;
    -- Optimization occurs at declaration, not runtime
END;
```

#### 4. Dynamic SQL
```sql
CREATE PROCEDURE GetUsersByStatus @Status VARCHAR(20)
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX) = 
        N'SELECT * FROM Users WHERE Status = @p1';
    EXEC sp_executesql @SQL, N'@p1 VARCHAR(20)', @Status;
    -- Plan generated each time with current parameter
END;
```

### Detecting Parameter Sniffing
```sql
-- Find plans with multiple executions and varying duration
SELECT 
    OBJECT_NAME(qs.object_id) AS ProcedureName,
    qs.execution_count,
    qs.min_elapsed_time / 1000.0 AS min_ms,
    qs.max_elapsed_time / 1000.0 AS max_ms,
    qs.avg_elapsed_time / 1000.0 AS avg_ms,
    (qs.max_elapsed_time - qs.min_elapsed_time) / 1000.0 AS variance_ms
FROM sys.dm_exec_procedure_stats qs
WHERE qs.max_elapsed_time > qs.avg_elapsed_time * 10  -- 10x variance
ORDER BY variance_ms DESC;
```

### Best Practices
1. **Monitor plan cache** for variations
2. **Use appropriate solution** for your scenario
3. **Test with representative data** during development
4. **Consider statement-level recompile** for critical queries
5. **Parameterize consistently** to maximize plan reuse

---

## Execution Plan Cache

### Viewing Cached Plans
```sql
-- Find plans in cache
SELECT 
    cp.plan_handle,
    cp.objtype,
    cp.usecounts,
    cp.size_in_bytes,
    t.text AS QueryText
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) t
WHERE cp.cacheobjtype = 'Compiled Plan'
ORDER BY cp.usecounts DESC;
```

### Clearing Plan Cache (Testing Only!)
```sql
-- Clear entire cache (DON'T DO IN PRODUCTION)
DBCC FREEPROCCACHE;

-- Clear for specific database
DBCC FLUSHPROCINDB(database_id);
```

### Plan Cache Best Practices
- Avoid ad-hoc queries (use parameterized SPs)
- Monitor for plan cache bloat
- Be cautious with OPTION (RECOMPILE) on high-frequency queries
- Consider forced parameterization for OLTP databases

---

*Source: SQL Server query execution documentation, plan analysis techniques, and performance tuning methodologies.*
