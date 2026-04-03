# Query Performance Optimization

## Table of Contents
1. [Functions in WHERE Clause](#functions-in-where-clause)
2. [JOIN vs Subquery](#join-vs-subquery)
3. [EXISTS vs IN](#exists-vs-in)
4. [Concurrency vs Parallelism](#concurrency-vs-parallelism)
5. [Separable vs Non-Separable Queries](#separable-vs-non-separable-queries)

---

## Functions in WHERE Clause

### The Problem
Using functions on indexed columns in the WHERE clause prevents index usage, forcing a full table scan.

### SARGable vs Non-SARGable

**SARGable** (Search ARGument ABLE): Queries that can use indexes efficiently.

**Non-SARGable**: Queries that cannot use indexes due to function wrapping.

### Common Anti-Patterns

#### Date Functions
```sql
-- Non-SARGable: Cannot use index on OrderDate
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2024;
SELECT * FROM Orders WHERE MONTH(OrderDate) = 6;
SELECT * FROM Orders WHERE DATEDIFF(day, OrderDate, GETDATE()) < 30;

-- SARGable: Index-friendly alternatives
SELECT * FROM Orders 
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';

SELECT * FROM Orders 
WHERE OrderDate >= DATEFROMPARTS(2024, 6, 1) 
  AND OrderDate < DATEFROMPARTS(2024, 7, 1);

SELECT * FROM Orders 
WHERE OrderDate > DATEADD(day, -30, GETDATE());
```

#### String Functions
```sql
-- Non-SARGable
SELECT * FROM Customers WHERE UPPER(LastName) = 'SMITH';
SELECT * FROM Products WHERE LEN(ProductName) > 10;
SELECT * FROM Users WHERE LEFT(Email, 5) = 'admin';

-- SARGable alternatives
SELECT * FROM Customers WHERE LastName = 'Smith' OR LastName = 'SMITH';
-- Or use computed column with index
```

#### Mathematical Operations
```sql
-- Non-SARGable
SELECT * FROM Products WHERE Price * 1.1 > 100;

-- SARGable
SELECT * FROM Products WHERE Price > 100 / 1.1;
-- Or: WHERE Price > 90.91
```

### Solution: Computed Columns with Indexes
```sql
-- Create computed column
ALTER TABLE Orders ADD OrderYear AS (YEAR(OrderDate)) PERSISTED;

-- Index the computed column
CREATE INDEX IX_Orders_Year ON Orders(OrderYear);

-- Now this uses the index
SELECT * FROM Orders WHERE OrderYear = 2024;
```

---

## JOIN vs Subquery

### Performance Reality
Modern query optimizers typically rewrite subqueries and joins into the same execution plan. The choice is often a matter of readability.

### JOIN-Based Approach
```sql
-- List users who have made posts
SELECT DISTINCT u.Id, u.DisplayName
FROM Users u
JOIN Posts p ON p.OwnerUserId = u.Id
WHERE p.PostTypeId = 2;  -- Answers
```

### Subquery-Based Approach
```sql
-- Same result using IN
SELECT u.Id, u.DisplayName
FROM Users u
WHERE u.Id IN (
    SELECT p.OwnerUserId
    FROM Posts p
    WHERE p.PostTypeId = 2
);
```

### EXISTS-Based Approach (Often Best)
```sql
-- Most efficient for existence checks
SELECT u.Id, u.DisplayName
FROM Users u
WHERE EXISTS (
    SELECT 1
    FROM Posts p
    WHERE p.OwnerUserId = u.Id
      AND p.PostTypeId = 2
);
```

### When Performance Differs

| Scenario | Better Option | Why |
|----------|---------------|-----|
| Correlated subquery with aggregation | JOIN | Single pass through data |
| Existence check only | EXISTS | Stops at first match |
| Need NULL handling | JOIN | IN/EXISTS handle NULLs differently |
| Large result sets | JOIN | Often better optimization |
| Small lookup table | Subquery | May use semi-join optimization |

### NULL Behavior Difference
```sql
-- IN returns UNKNOWN (treated as FALSE) with NULL
SELECT * FROM Users WHERE Id IN (1, 2, NULL);  -- May return nothing

-- JOIN handles NULL differently
SELECT DISTINCT u.*
FROM Users u
JOIN Posts p ON p.OwnerUserId = u.Id  -- NULLs won't match
```

---

## EXISTS vs IN

### EXISTS
- Checks for existence of rows
- Stops at first match (short-circuit)
- Handles NULLs correctly
- Generally faster for large tables

```sql
-- Efficient existence check
SELECT u.Id, u.DisplayName
FROM Users u
WHERE EXISTS (
    SELECT 1
    FROM Posts p
    WHERE p.OwnerUserId = u.Id
);
```

### IN
- Checks membership in a set
- Must evaluate entire subquery
- NULL handling can cause unexpected results
- Good for small, static lists

```sql
-- Membership test
SELECT * FROM Users WHERE Id IN (1, 2, 3, 4, 5);

-- With subquery (less efficient than EXISTS for large sets)
SELECT * FROM Users 
WHERE Id IN (SELECT OwnerUserId FROM Posts);
```

### Performance Comparison

| Metric | EXISTS | IN |
|--------|--------|-----|
| **Short-circuit** | Yes | No |
| **NULL handling** | Correct | Problematic |
| **Large tables** | Faster | Slower |
| **Empty subquery** | Fast | Fast |
| **Readability** | Good | Excellent |

### Best Practice
```sql
-- Use EXISTS for semi-joins (checking existence)
-- Use IN for small, fixed value lists
-- Use JOIN when you need data from both tables
```

---

## Concurrency vs Parallelism

### Concurrency
**Definition**: Multiple operations making progress over the same time period, alternating execution on shared resources.

**Characteristics:**
- Single-core or multi-core
- Tasks interleave (start/stop/start)
- Resource sharing requires coordination
- Managed by OS scheduler

**Database Example:**
```sql
-- Multiple transactions running concurrently
-- Transaction 1: BEGIN; SELECT * FROM Orders; COMMIT;
-- Transaction 2: BEGIN; UPDATE Orders SET...; COMMIT;
-- Scheduler alternates between them
```

### Parallelism
**Definition**: Multiple operations executing simultaneously on different processors/cores.

**Characteristics:**
- Requires multi-core hardware
- True simultaneous execution
- Divides work across threads
- Requires data partitioning

**Database Example:**
```sql
-- Query executes on multiple threads
SELECT COUNT(*) FROM LargeTable WITH (MAXDOP 4);
-- SQL Server uses 4 parallel threads
```

### Comparison Table

| Aspect | Concurrency | Parallelism |
|--------|-------------|-------------|
| **Execution** | Interleaved | Simultaneous |
| **Hardware** | Any | Multi-core required |
| **Goal** | Resource utilization | Speedup |
| **Coordination** | Locks, waits | Synchronization |
| **SQL Server** | Locking/latching | Parallel query plans |

### Parallelism in SQL Server

#### Enabling Parallelism
```sql
-- Query hint for parallelism
SELECT * FROM LargeTable
OPTION (MAXDOP 4);  -- Use up to 4 threads

-- Server-level setting
EXEC sp_configure 'max degree of parallelism', 4;
RECONFIGURE;
```

#### When Parallelism Helps
- Large table scans (millions of rows)
- Complex aggregations
- Sorting large result sets
- Hash joins on large datasets

#### When Parallelism Hurts
- Small tables (< 1000 rows)
- OLTP workloads (short transactions)
- High-concurrency systems
- Cost of coordination exceeds benefit

---

## Separable vs Non-Separable Queries

### Separable Query
A query that can be divided into independent parts that execute in parallel without dependencies.

**Characteristics:**
- No row-to-row dependencies
- Each partition can be processed independently
- Results can be combined at the end

**Example:**
```sql
-- Separable: Counting rows in partitions
SELECT COUNT(*) FROM Orders
WHERE OrderDate BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY MONTH(OrderDate);
-- Each month can be counted in parallel
```

### Non-Separable Query
A query with dependencies that prevent parallel execution or require sequential processing.

**Characteristics:**
- Dependencies between rows
- Must maintain order
- Requires global state

**Examples:**
```sql
-- Non-separable: Running totals
SELECT 
    OrderID,
    OrderDate,
    Amount,
    SUM(Amount) OVER (ORDER BY OrderDate) AS RunningTotal
FROM Orders;
-- Each row depends on previous rows
```

```sql
-- Non-separable: Recursive CTE
WITH RecursiveCategories AS (
    SELECT CategoryID, CategoryName, ParentID, 0 AS Level
    FROM Categories
    WHERE ParentID IS NULL
    
    UNION ALL
    
    SELECT c.CategoryID, c.CategoryName, c.ParentID, r.Level + 1
    FROM Categories c
    JOIN RecursiveCategories r ON c.ParentID = r.CategoryID
)
SELECT * FROM RecursiveCategories;
-- Each level depends on previous level results
```

### Parallel Query Plans

**Parallelism Operators:**
- **Distribute Streams**: Split data across threads
- **Repartition Streams**: Redistribute for join/aggregate
- **Gather Streams**: Combine results from threads

**Visual Execution Plan:**
```
SELECT * (Gather Streams)
    └── Parallelism (Repartition Streams)
            ├── Thread 1: Index Scan (Range A-M)
            ├── Thread 2: Index Scan (Range N-Z)
            └── ...
```

### Controlling Parallelism

```sql
-- Force serial execution
SELECT * FROM LargeTable
OPTION (MAXDOP 1);

-- Allow optimizer to choose
SELECT * FROM LargeTable;

-- Cost threshold for parallelism
EXEC sp_configure 'cost threshold for parallelism', 50;
-- Only parallelize if estimated cost > 50
```

### Best Practices

1. **Let optimizer decide** most of the time
2. **Limit MAXDOP** on OLTP systems (2-4)
3. **Set cost threshold** appropriately (25-50 typically)
4. **Monitor CXPACKET waits** (excessive parallelism)
5. **Use MAXDOP hints** for specific problematic queries

---

## Performance Monitoring

### Key DMVs
```sql
-- Find slow queries
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count / 1000 AS avg_duration_ms,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY avg_duration_ms DESC;
```

### Execution Plan Analysis
```sql
-- Get actual execution plan
SET STATISTICS XML ON;
GO
-- Your query here
SET STATISTICS XML OFF;
```

---

*Source: SQL Server query optimization, execution plan analysis, and database performance tuning research.*
