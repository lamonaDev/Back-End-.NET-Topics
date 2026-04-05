# Pagination Strategies

## Table of Contents
1. [Offset Pagination](#offset-pagination)
2. [Cursor (Keyset) Pagination](#cursor-keyset-pagination)
3. [Comparison & Use Cases](#comparison--use-cases)

---

## Offset Pagination

### Overview
The traditional pagination method using `OFFSET` and `LIMIT` (or `TOP` in SQL Server) to skip rows and return a subset.

### How It Works
```sql
-- Page 1 (first 20 rows)
SELECT * FROM Products
ORDER BY ProductID
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;

-- Page 2 (rows 21-40)
SELECT * FROM Products
ORDER BY ProductID
OFFSET 20 ROWS FETCH NEXT 20 ROWS ONLY;

-- Page 3 (rows 41-60)
SELECT * FROM Products
ORDER BY ProductID
OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;
```

### Syntax by Database

| Database | Syntax |
|----------|--------|
| **SQL Server** | `OFFSET x ROWS FETCH NEXT y ROWS ONLY` |
| **PostgreSQL** | `OFFSET x LIMIT y` |
| **MySQL** | `LIMIT y OFFSET x` |
| **Oracle** | `OFFSET x ROWS FETCH NEXT y ROWS ONLY` |

### Advantages

1. **Simplicity**: Easy to understand and implement
2. **Random Access**: Can jump to any page directly
3. **Total Count**: Easy to calculate total pages
4. **Universal Support**: Supported by all major databases

```sql
-- Get total pages
SELECT CEIL(COUNT(*) / 20.0) AS TotalPages FROM Products;
```

### Disadvantages

1. **Performance Degradation**: Gets slower as OFFSET increases
2. **Inconsistent Results**: New inserts/deletes shift data between pages
3. **Full Scan Requirement**: Database must scan and discard OFFSET rows
4. **Resource Intensive**: High memory and CPU usage for large offsets

### Performance Problem Explained
```sql
-- This query must:
-- 1. Scan the entire index
-- 2. Read and discard 1,000,000 rows
-- 3. Return 20 rows
SELECT * FROM Products
ORDER BY ProductID
OFFSET 1000000 ROWS FETCH NEXT 20 ROWS ONLY;
-- Execution time: ~5-10 seconds on large tables
```

### When to Use
- Small datasets (< 10,000 rows)
- Admin interfaces needing random page access
- When total page count is required
- Infrequently accessed deep pages

### Real-World Examples
- **Google Search Results**: First few pages only
- **E-commerce filters**: Initial browsing pages
- **Admin dashboards**: Limited data exploration

---

## Cursor (Keyset) Pagination

### Overview
Also called **Keyset Pagination** or **Seek Method**, uses the last seen value as a filter for the next page.

### How It Works
```sql
-- First page (no cursor needed)
SELECT * FROM Products
ORDER BY ProductID
FETCH FIRST 20 ROWS ONLY;

-- Result: last ProductID = 4521

-- Next page (use last ID as starting point)
SELECT * FROM Products
WHERE ProductID > 4521
ORDER BY ProductID
FETCH FIRST 20 ROWS ONLY;

-- Continue with new last ID...
```

### Multi-Column Sorting
```sql
-- When sorting by multiple columns
SELECT * FROM Posts
WHERE (CreatedDate < '2024-01-15' 
   OR (CreatedDate = '2024-01-15' AND PostID < 12345))
ORDER BY CreatedDate DESC, PostID DESC
FETCH FIRST 20 ROWS ONLY;
```

### Advantages

1. **Consistent Performance**: O(page size) regardless of depth
2. **Stable Results**: New rows don't affect existing pages
3. **Efficient**: Uses index seek, no row skipping
4. **No Duplicate/Missing**: Data changes between requests handled gracefully

### Performance Comparison
```sql
-- Offset (slow on deep pages)
OFFSET 1000000 ROWS  -- Must scan 1M+ rows

-- Cursor (fast at any depth)
WHERE ID > 4521     -- Direct index seek
```

### Disadvantages

1. **No Random Access**: Cannot jump to arbitrary page
2. **No Total Count**: Cannot easily determine total pages
3. **Complex Implementation**: Requires tracking last value
4. **Sorting Limitations**: Only works with deterministic sorts

### Implementation Pattern
```csharp
// Cursor-based pagination in application code
public class CursorPagination<T>
{
    public List<T> Items { get; set; }
    public string NextCursor { get; set; }  // Last seen ID
    public bool HasMore { get; set; }
}

// API Response
{
    "items": [...],
    "nextCursor": "eyJpZCI6NDUyMX0=",  // Base64 encoded cursor
    "hasMore": true
}
```

### When to Use
- Large datasets (> 100,000 rows)
- Social media feeds (infinite scroll)
- Time-series data (logs, events)
- Real-time data that changes frequently
- APIs where consistent performance matters

### Real-World Examples
- **Twitter/X Timeline**: Infinite scroll, stable feed
- **Facebook News Feed**: Consistent pagination
- **Slack Message History**: Time-based cursor
- **GitHub API**: Cursor-based for large result sets
- **Stripe API**: Uses cursors for all list endpoints

---

## Comparison & Use Cases

### Feature Comparison

| Feature | Offset | Cursor |
|---------|--------|--------|
| **Performance (deep pages)** | Poor | Excellent |
| **Performance (first pages)** | Good | Good |
| **Random page access** | Yes | No |
| **Total count available** | Yes | Difficult |
| **Result stability** | Poor | Excellent |
| **Implementation complexity** | Low | Medium |
| **Memory efficiency** | Poor | Excellent |
| **Works with any sort** | Yes | Requires unique/indexed column |

### Decision Matrix

| Scenario | Recommendation |
|----------|----------------|
| < 10,000 rows | **Offset** - simpler, fine for small data |
| Large tables, user-facing | **Cursor** - consistent performance |
| Random page navigation needed | **Offset** - only option for direct page access |
| Real-time feeds, social media | **Cursor** - stable, fast infinite scroll |
| Admin reports, data export | **Offset** - may need total count |
| API with SLA requirements | **Cursor** - predictable performance |
| Time-series data (logs, events) | **Cursor** - natural temporal ordering |

### Hybrid Approach
Some applications use both strategies:

```sql
-- First few pages: Offset (user can navigate)
-- Deep pages: Cursor (maintain performance)

-- Example: Show pages 1-5 with offset
-- Then offer "Load More" using cursor
```

### Cursor Implementation Best Practices

1. **Use Encoded Cursors**
```csharp
// Encode cursor data (page position + sort values)
var cursorData = JsonConvert.SerializeObject(new {
    lastId = lastItem.Id,
    lastDate = lastItem.CreatedDate
});
var cursor = Convert.ToBase64String(Encoding.UTF8.GetBytes(cursorData));
```

2. **Handle Sorting Edge Cases**
```sql
-- Always include unique column in sort
ORDER BY CreatedDate DESC, ID DESC  -- ID ensures determinism
```

3. **Index Strategy**
```sql
-- Composite index for cursor pagination
CREATE INDEX IX_Posts_CreatedDate_ID 
ON Posts(CreatedDate DESC, ID DESC);
```

### Database-Specific Cursor Syntax

**PostgreSQL with Row Values:**
```sql
SELECT * FROM Products
WHERE (price, id) > (100, 50)  -- Composite cursor
ORDER BY price, id
LIMIT 20;
```

**SQL Server SEEK Method:**
```sql
SELECT * FROM Products
WHERE ProductID > @LastProductID
ORDER BY ProductID
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
```

### Monitoring Query Performance

```sql
-- Check offset query performance
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Offset query
SELECT * FROM LargeTable
ORDER BY ID
OFFSET 100000 ROWS FETCH NEXT 20 ROWS ONLY;
-- Look for: high logical reads, scan counts

-- Cursor query
SELECT * FROM LargeTable
WHERE ID > 100000
ORDER BY ID
FETCH FIRST 20 ROWS ONLY;
-- Look for: low logical reads, seek operations
```

---

*Source: Database performance research, API design best practices, and real-world application patterns from Twitter, Stripe, and GitHub APIs.*
