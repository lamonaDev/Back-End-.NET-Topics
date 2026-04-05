# Advanced Indexing Concepts

## Table of Contents
1. [Columnstore vs Rowstore](#columnstore-vs-rowstore-index)
2. [Full-Text Index](#full-text-index)
3. [Hash Index](#hash-index)
4. [Bitmap Index](#bitmap-index)
5. [NEWSEQUENTIALID](#newsequentialid)

---

## Columnstore vs Rowstore Index

### Rowstore Index (Traditional)
**Storage**: Data organized and stored by rows.

**Structure**:
```
Row 1: [ID=1, Name=John, Age=25, City=NYC]
Row 2: [ID=2, Name=Jane, Age=30, City=LA]
Row 3: [ID=3, Name=Bob, Age=35, City=Chicago]
```

**Best For:**
- OLTP workloads
- Point lookups (single row queries)
- Small range scans
- Frequent updates
- Queries selecting most columns

**Characteristics:**
- Excellent for seeking individual rows
- Efficient for write operations
- B-tree structure for fast lookups
- Data stored in pages (8KB each)

### Columnstore Index
**Storage**: Data organized and stored by columns.

**Structure**:
```
ID Column:    [1, 2, 3, 4, 5...]
Name Column:  [John, Jane, Bob, Alice...]
Age Column:   [25, 30, 35, 28...]
City Column:  [NYC, LA, Chicago, Boston...]
```

**Best For:**
- Data warehousing (OLAP)
- Aggregations (SUM, AVG, COUNT)
- Analytical queries
- Large table scans
- Queries selecting few columns

**Characteristics:**
- Highly compressed (10x typical)
- Batch mode processing (900+ rows at once)
- Poor for single-row lookups
- Read-optimized, slower for updates

### Comparison Table

| Aspect | Rowstore | Columnstore |
|--------|----------|-------------|
| **Best workload** | OLTP | OLAP/Data Warehousing |
| **Storage** | By rows | By columns |
| **Compression** | Minimal | 10-30x typical |
| **Point queries** | Excellent | Poor |
| **Aggregations** | Good | Excellent |
| **Updates** | Fast | Slow (rebuild delta store) |
| **Memory usage** | Lower | Higher (decompressed) |
| **Batch processing** | Row by row | 900+ rows at once |

### Creating Columnstore Index
```sql
-- Nonclustered columnstore
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_CS_Sales
ON Sales (ProductID, Quantity, Price, SaleDate);

-- Clustered columnstore (SQL Server 2014+)
CREATE CLUSTERED COLUMNSTORE INDEX IX_CCS_Sales
ON Sales;
```

### When to Use Each

**Rowstore:**
- Transaction processing systems
- E-commerce checkout flows
- User management
- Any system with frequent updates

**Columnstore:**
- Reporting databases
- Data marts
- Historical analytics
- Large fact tables

---

## Full-Text Index

### Purpose
Enables complex searches on text data including:
- Word or phrase searching
- Inflectional forms (run, ran, running)
- Proximity searches (words near each other)
- Weighted rankings

### Creating Full-Text Index
```sql
-- Step 1: Create full-text catalog
CREATE FULLTEXT CATALOG ProductCatalog AS DEFAULT;

-- Step 2: Create unique index (required)
CREATE UNIQUE INDEX IX_Products_ProductID ON Products(ProductID);

-- Step 3: Create full-text index
CREATE FULLTEXT INDEX ON Products
(
    ProductName LANGUAGE English,
    Description LANGUAGE English
)
KEY INDEX IX_Products_ProductID
ON ProductCatalog
WITH CHANGE_TRACKING AUTO;
```

### Full-Text Queries
```sql
-- Simple match
SELECT * FROM Products
WHERE CONTAINS(Description, 'laptop');

-- Multiple words (AND implied)
SELECT * FROM Products
WHERE CONTAINS(Description, 'laptop AND computer');

-- Phrase search
SELECT * FROM Products
WHERE CONTAINS(Description, '"gaming laptop"');

-- Inflectional forms
SELECT * FROM Products
WHERE CONTAINS(Description, 'FORMSOF(INFLECTIONAL, run)');
-- Finds: run, ran, running, runs

-- Proximity search
SELECT * FROM Products
WHERE CONTAINS(Description, 'laptop NEAR gaming');
```

### FREETEXT for Natural Language
```sql
-- Broader matching using thesaurus
SELECT * FROM Products
WHERE FREETEXT(Description, 'portable computer');
-- Matches: laptop, notebook, netbook, computer
```

### Ranking Results
```sql
SELECT 
    ProductID,
    ProductName,
    KEY_TBL.RANK
FROM Products
INNER JOIN FREETEXTTABLE(Products, Description, 'wireless mouse') AS KEY_TBL
    ON Products.ProductID = KEY_TBL.[KEY]
ORDER BY KEY_TBL.RANK DESC;
```

---

## Hash Index

### Purpose
Optimized for point lookups using a hash function on the key value.

### Characteristics
- Hash function computes bucket location
- O(1) lookup time for equality searches
- Cannot support range queries
- No sorting capability

### In-Memory OLTP (SQL Server)
```sql
-- Hash index for memory-optimized table
CREATE TABLE Cache_Users
(
    UserID INT NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 100000),
    UserName NVARCHAR(100),
    LastAccessed DATETIME
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- Create additional hash index
CREATE NONCLUSTERED HASH INDEX IX_Hash_Username
ON Cache_Users(UserName)
WITH (BUCKET_COUNT = 50000);
```

### Bucket Count Considerations
```sql
-- Bucket count should be 1.5-2x expected unique values
-- Too few: Long chains, slower lookups
-- Too many: Wasted memory

-- Formula: BUCKET_COUNT = (ExpectedRows * 2)
```

### When to Use Hash Indexes
- Exact equality lookups (`=`)
- Memory-optimized tables
- High-concurrency scenarios
- No need for range queries

---

## Bitmap Index

### Purpose
Space-efficient index for columns with low cardinality (few distinct values).

### Characteristics
- One bitmap per distinct value
- Each bit represents a row
- Highly compressed for low-cardinality data
- Excellent for data warehousing

### When to Use
| Cardinality | Recommendation |
|-------------|----------------|
| High (>1000 unique) | B-tree index |
| Medium (100-1000) | Consider filtered indexes |
| Low (<100) | Bitmap index candidate |

### Common Low-Cardinality Columns
- Gender
- Status (Active/Inactive)
- State/Country
- Product Category
- Boolean flags

### SQL Server Implementation
SQL Server doesn't have native bitmap indexes like Oracle, but columnstore uses bitmap compression internally.

```sql
-- Use columnstore for bitmap-like benefits
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_CS_Users
ON Users (Status, Gender, Country);

-- Or filtered indexes for specific values
CREATE INDEX IX_Users_Active ON Users(UserID) 
WHERE Status = 'Active';
```

---

## NEWSEQUENTIALID

### Purpose
Generates GUIDs that are sequential (like integers) while maintaining global uniqueness.

### The GUID Fragmentation Problem
Standard GUIDs (`NEWID()`) are random:
```sql
-- NEWID() produces scattered values
-- Page 1: [A1, Z9, B2, Y8...]
-- Page 2: [M5, P3, K7...]
-- Result: Constant page splits, fragmentation
```

### NEWSEQUENTIALID Solution
```sql
-- Sequential GUIDs
-- Page 1: [A1, A2, A3, A4...]
-- Page 2: [B1, B2, B3...]
-- Result: Efficient clustered index inserts
```

### Creating Sequential GUID Column
```sql
-- Default constraint with NEWSEQUENTIALID
CREATE TABLE SequentialUsers
(
    UserID UNIQUEIDENTIFIER DEFAULT NEWSEQUENTIALID() PRIMARY KEY,
    UserName NVARCHAR(100),
    CreatedAt DATETIME DEFAULT GETDATE()
);
```

### Comparison: NEWID vs NEWSEQUENTIALID

| Aspect | NEWID() | NEWSEQUENTIALID() |
|--------|---------|-------------------|
| **Uniqueness** | Guaranteed | Guaranteed |
| **Sequential** | No | Yes |
| **Index fragmentation** | Severe | Minimal |
| **Security** | Unpredictable | Slightly predictable |
| **Use case** | Security tokens | Primary keys |
| **Insert performance** | Slow | Fast |

### When to Use Each

**NEWID()**:
- Session tokens
- API keys
- Anywhere unpredictability is desired

**NEWSEQUENTIALID()**:
- Primary keys in replicated systems
- GUID clustering key needed
- High-insert OLTP systems

### UUID Version 7 (Modern Alternative)
The new standard embeds a timestamp:
```csharp
// .NET 9+ supports UUID v7
Guid.NewGuid(); // UUID v7 - time-ordered
```

### Summary: GUID as Primary Key

| Approach | Index Type | Recommendation |
|----------|------------|----------------|
| Random GUID | Non-clustered | Use if GUID required |
| Sequential GUID | Clustered | NEWSEQUENTIALID or UUID v7 |
| Integer surrogate | Clustered | Best for performance |

---

## Index Type Selection Guide

| Use Case | Recommended Index |
|----------|-------------------|
| OLTP, point lookups | Rowstore (B-tree) |
| Data warehouse, analytics | Columnstore |
| Text search | Full-Text |
| Exact match, in-memory | Hash |
| Low cardinality filtering | Filtered index |
| Distributed systems | Sequential GUID |

---

*Source: SQL Server indexing documentation, in-memory OLTP features, and data warehousing best practices.*
