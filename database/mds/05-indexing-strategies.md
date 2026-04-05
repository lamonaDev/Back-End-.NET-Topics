# Indexing Strategies

## Table of Contents
1. [Why Use Indexes?](#why-use-indexes)
2. [When Indexes Are Ignored](#when-indexes-are-ignored)
3. [Index Types](#index-types)
4. [Index Maintenance](#index-maintenance)

---

## Why Use Indexes

### Definition
An index is a database structure that improves the speed of data retrieval operations on a table at the cost of additional storage and write performance overhead.

### How Indexes Work
Indexes work like a book's index:
- **Without index**: Full table scan (read every page)
- **With index**: Binary tree traversal (seek directly to data)

### Benefits

| Benefit | Explanation |
|---------|-------------|
| **Faster SELECTs** | Direct data location vs full scan |
| **Efficient Sorting** | Pre-ordered data retrieval |
| **Unique Constraints** | Enforce data integrity |
| **Foreign Keys** | Speed up join operations |
| **Aggregate Functions** | Covering indexes avoid table access |

### Cost of Indexes
- **Storage**: ~20-30% additional space
- **Write operations**: Slower INSERT, UPDATE, DELETE
- **Maintenance**: Automatic rebuilds during heavy operations

### Anatomy of an Index
```
Root Page
    ├── Branch Page (A-M)
    │     ├── Leaf Page (A-F)
    │     │     ├── Row (Name: Alice, Pointer: Page 12, Slot 5)
    │     │     └── Row (Name: Bob, Pointer: Page 15, Slot 2)
    │     └── Leaf Page (G-M)
    └── Branch Page (N-Z)
          └── ...
```

---

## When Indexes Are Ignored

### 1. Implicit Data Type Conversions
```sql
-- Index on VARCHAR column won't be used
CREATE INDEX IX_Phone ON Customers(PhoneNumber);

-- Bad: Implicit conversion prevents index usage
SELECT * FROM Customers WHERE PhoneNumber = 1234567890;

-- Good: Match data types
SELECT * FROM Customers WHERE PhoneNumber = '1234567890';
```

### 2. Functions on Indexed Columns
```sql
-- Index won't be used (function wraps column)
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2024;

-- Good: Avoid function on indexed column
SELECT * FROM Orders 
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';
```

### 3. Leading Wildcards in LIKE
```sql
-- Index cannot be used
SELECT * FROM Products WHERE Name LIKE '%laptop%';

-- Index can be used
SELECT * FROM Products WHERE Name LIKE 'laptop%';
```

### 4. NOT Operators
```sql
-- Index usage limited
SELECT * FROM Products WHERE Status <> 'Active';

-- Consider: Separate indexes for common statuses
SELECT * FROM Products WHERE Status = 'Inactive';
```

### 5. Calculations on Columns
```sql
-- Index won't be used
SELECT * FROM Products WHERE Price * 1.1 > 100;

-- Good: Calculate constant first
SELECT * FROM Products WHERE Price > 90.91;
```

### 6. Low Cardinality
```sql
-- Index on boolean-like column (mostly one value)
CREATE INDEX IX_IsDeleted ON Users(IsDeleted);

-- Optimizer may choose scan if >95% are false
```

### 7. Small Tables
```sql
-- For tables with < ~1000 rows, scan is often faster
-- Index lookup overhead exceeds sequential scan benefit
```

### 8. Outdated Statistics
```sql
-- When statistics are stale, optimizer makes poor choices
-- Update statistics manually:
UPDATE STATISTICS Products WITH FULLSCAN;
```

### 9. Parameter Sniffing
```sql
-- Plan cached for specific parameter may not work for others
-- Solution: OPTION (RECOMPILE) or OPTIMIZE FOR UNKNOWN
```

### 10. Query Hints Override
```sql
-- FORCESEEK, FORCESCAN hints can prevent optimal plan
SELECT * FROM Products WITH (FORCESCAN)
WHERE ProductID = 123;
```

---

## Index Types

### Clustered Index
- **Storage**: Actual table data is physically ordered
- **One per table**: Only one clustered index allowed
- **Default**: Primary key becomes clustered if not specified

```sql
-- Create clustered index
CREATE CLUSTERED INDEX IX_Users_CreatedDate 
ON Users(CreatedDate);

-- Or as primary key
ALTER TABLE Users 
ADD CONSTRAINT PK_Users PRIMARY KEY CLUSTERED (UserID);
```

### Non-Clustered Index
- **Storage**: Separate structure with pointers to data
- **Multiple allowed**: Up to 999 per table (SQL Server)
- **Key + Included columns**: Index key + covering columns

```sql
-- Non-clustered with included columns (covering index)
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Date
ON Orders(CustomerID, OrderDate)
INCLUDE (TotalAmount, Status);
```

### Covering Index
An index that contains all columns needed for a query, eliminating the need for key lookups.

```sql
-- Query needs: CustomerID, OrderDate, TotalAmount
-- Covering index includes all three
CREATE INDEX IX_Orders_Covering
ON Orders(CustomerID, OrderDate)
INCLUDE (TotalAmount);
```

### Composite Index
Multiple columns in a single index. **Order matters** - most selective column first.

```sql
-- Good for: WHERE LastName = 'Smith' AND FirstName = 'John'
-- NOT for: WHERE FirstName = 'John' alone
CREATE INDEX IX_Users_Name 
ON Users(LastName, FirstName);
```

### Filtered Index
Index on a subset of data, smaller and more efficient.

```sql
-- Index only active users
CREATE INDEX IX_Users_Active_Email
ON Users(Email)
WHERE IsActive = 1;
```

### Columnstore Index
Optimized for data warehousing, compresses data by column.

```sql
-- Clustered columnstore for large fact tables
CREATE CLUSTERED COLUMNSTORE INDEX IX_Fact_Sales
ON Fact_Sales;
```

### Index Selection Guide

| Scenario | Index Type |
|----------|------------|
| Primary key lookups | Clustered |
| Foreign key columns | Non-clustered |
| Search/filter columns | Non-clustered |
| ORDER BY columns | Non-clustered |
| Covering queries | Include columns |
| Partial data access | Filtered index |
| Data warehouse | Columnstore |

---

## Index Maintenance

### Fragmentation Types

#### Internal Fragmentation
- Pages not full, wasted space within pages
- Caused by: INSERTs in middle, UPDATEs increasing row size
- Impact: More I/O for same data

#### External Fragmentation
- Pages out of order physically
- Caused by: INSERTs at random positions, page splits
- Impact: Sequential reads become random reads

### Measuring Fragmentation
```sql
-- Check fragmentation levels
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    name AS IndexName,
    index_type_desc,
    avg_fragmentation_in_percent,
    page_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED'
)
WHERE avg_fragmentation_in_percent > 30;
```

### REBUILD vs REORGANIZE

| Aspect | REBUILD | REORGANIZE |
|--------|---------|------------|
| **Duration** | Long | Short |
| **Locking** | Schema lock (blocking) | Minimal |
| **ONLINE** | Enterprise Edition | Always available |
| **Fragmentation threshold** | >30% | 5-30% |
| **Result** | New index structure | Defrag existing |

```sql
-- Reorganize (lightweight, online)
ALTER INDEX IX_Orders_Customer ON Orders REORGANIZE;

-- Rebuild ( Enterprise: ONLINE=ON )
ALTER INDEX IX_Orders_Customer ON Orders 
REBUILD WITH (ONLINE = ON);
```

### Maintenance Strategy
```sql
-- Weekly maintenance script
DECLARE @SchemaName NVARCHAR(128);
DECLARE @TableName NVARCHAR(128);
DECLARE @IndexName NVARCHAR(128);
DECLARE @Fragmentation FLOAT;

DECLARE IndexCursor CURSOR FOR
SELECT 
    OBJECT_SCHEMA_NAME(object_id) AS SchemaName,
    OBJECT_NAME(object_id) AS TableName,
    name AS IndexName,
    avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'SAMPLED'
)
WHERE avg_fragmentation_in_percent > 5
  AND page_count > 1000;

OPEN IndexCursor;
FETCH NEXT FROM IndexCursor INTO @SchemaName, @TableName, @IndexName, @Fragmentation;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Fragmentation < 30
    BEGIN
        -- Reorganize
        EXEC('ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REORGANIZE');
    END
    ELSE
    BEGIN
        -- Rebuild
        EXEC('ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REBUILD WITH (ONLINE = ON)');
    END
    
    FETCH NEXT FROM IndexCursor INTO @SchemaName, @TableName, @IndexName, @Fragmentation;
END;

CLOSE IndexCursor;
DEALLOCATE IndexCursor;
```

### Statistics Maintenance
```sql
-- Update all statistics
EXEC sp_updatestats;

-- Update specific table with full scan
UPDATE STATISTICS Orders WITH FULLSCAN;

-- Auto-update settings
ALTER DATABASE [MyDB] SET AUTO_UPDATE_STATISTICS ON;
ALTER DATABASE [MyDB] SET AUTO_CREATE_STATISTICS ON;
```

---

## Index Best Practices

### Design Principles
1. **Index for WHERE clauses** first
2. **Consider JOIN columns** (foreign keys)
3. **Cover ORDER BY** when possible
4. **Avoid over-indexing** - each index costs writes
5. **Include columns** instead of adding to key

### Monitoring Index Usage
```sql
-- Find unused indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    s.last_user_seek,
    s.last_user_scan
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) = 'YourTable'
ORDER BY s.user_updates DESC;
```

### Missing Indexes
```sql
-- Find potentially useful indexes
SELECT 
    mig.index_handle,
    mid.statement AS TableName,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.user_seeks,
    migs.avg_total_user_cost,
    migs.avg_user_impact
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY migs.avg_user_impact DESC;
```

---

*Source: SQL Server indexing documentation, query optimization research, and production database tuning experience.*
