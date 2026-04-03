# Database Research Documentation

This repository contains comprehensive, professional documentation covering database concepts, SQL Server features, and NoSQL fundamentals. Each topic is explained with practical examples and best practices.

## Table of Contents

### Database Design & Fundamentals
1. **[Database Design & Normalization](01-database-design-normalization.md)**
   - Primary Keys: Number vs GUID
   - BCNF, 4NF, 5NF

2. **[NoSQL Fundamentals](12-nosql-fundamentals.md)**
   - Document, Key-Value, Column-Family, Graph databases
   - Data modeling patterns

### Backup & Recovery
3. **[Backup & Recovery](02-backup-recovery.md)**
   - .bak vs .bacpac files
   - Backup types (Full, Differential, Log)
   - RPO/RTO strategies

### Data Validation & Constraints
4. **[Data Validation & Constraints](03-data-validation-constraints.md)**
   - NO ACTION vs RESTRICT
   - Password & Email validation patterns
   - LIKE operator limitations

### Query Performance
5. **[Pagination Strategies](04-pagination-strategies.md)**
   - Offset vs Cursor pagination
   - Use cases and comparisons

6. **[Query Performance Optimization](06-query-performance.md)**
   - Functions in WHERE clause
   - JOIN vs Subquery vs EXISTS
   - Concurrency vs Parallelism

7. **[Query Optimization Techniques](13-query-optimization.md)**
   - Adaptive Join
   - Hash/Sort spills
   - N+1 Query Problem

### Indexing
8. **[Indexing Strategies](05-indexing-strategies.md)**
   - When indexes are ignored
   - Clustered vs Non-clustered
   - Covering and Filtered indexes

9. **[Advanced Indexing](09-advanced-indexes.md)**
   - Columnstore vs Rowstore
   - Full-Text indexes
   - Hash and Bitmap indexes

### SQL Functions
10. **[SQL Functions Reference](07-sql-functions.md)**
    - DATEFROMPARTS, EOMONTH
    - LEN vs DATALENGTH
    - CONCAT_WS, LEFT

### Concurrency & Locking
11. **[Locking & Concurrency Control](08-locking-concurrency.md)**
    - Lock types (Shared, Exclusive, Update)
    - Intent locks
    - Isolation levels
    - Deadlock prevention

### Execution Plans & Performance
12. **[Execution Plans Deep Dive](10-execution-plans.md)**
    - Estimated vs Actual plans
    - Common operators
    - Parameter sniffing

### Security
13. **[Security Vulnerabilities](11-security-vulnerabilities.md)**
    - SQL Injection
    - XSS, CSRF
    - Password security

## Quick Reference

### File Mapping to Original Research

| Source File | Documentation File |
|-------------|-------------------|
| 3-Search.txt (Normalization) | 01-database-design-normalization.md |
| 5-Search.txt (Validation) | 03-data-validation-constraints.md |
| 6-Search.txt (Pagination) | 04-pagination-strategies.md |
| 7-Search.txt (Indexing/Functions) | 05-indexing-strategies.md, 07-sql-functions.md |
| 8-Search.txt (Locking/Joins) | 06-query-performance.md, 08-locking-concurrency.md |
| 9.html (Advanced Topics) | 09-advanced-indexes.md, 10-execution-plans.md |
| 10-Task.html (Performance) | 06-query-performance.md, 10-execution-plans.md |
| 11-12-Task.txt (Isolation) | 08-locking-concurrency.md |
| 13-research.html (Security/NoSQL) | 11-security-vulnerabilities.md, 12-nosql-fundamentals.md, 02-backup-recovery.md |

## How to Use This Documentation

1. **Learning Path**: Start with fundamentals (01, 05) and progress to advanced topics (09, 10)
2. **Reference**: Use as quick lookup for specific SQL functions or patterns
3. **Interview Prep**: Each file contains key concepts commonly asked in technical interviews
4. **Project Work**: Practical examples ready for adaptation to real scenarios

## Key Concepts Summary

### Performance
- **SARGable queries** - Write queries that can use indexes
- **Covering indexes** - Include all needed columns in index
- **Avoid functions** on indexed columns in WHERE clauses
- **Use EXISTS** over IN for existence checks

### Design
- **Normalization** - Balance between redundancy and integrity
- **Partitioning** - Split large tables for manageability
- **Guid keys** - Use sequential GUIDs (NEWSEQUENTIALID/UUIDv7) over random GUIDs

### Concurrency
- **Isolation levels** - Choose appropriate level for your consistency needs
- **Lock hints** - Use only when necessary
- **Deadlock handling** - Implement retry logic in applications

### Security
- **Parameterized queries** - Never concatenate user input
- **Least privilege** - Application accounts with minimal permissions
- **Encryption** - Encrypt connections and sensitive data

## Additional Resources

- [SQL Server Documentation](https://docs.microsoft.com/sql/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Redis Documentation](https://redis.io/documentation)

---

*Compiled from research assignments and coursework materials. Last updated: April 2026*
