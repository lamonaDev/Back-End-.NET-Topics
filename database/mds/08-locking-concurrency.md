# Locking & Concurrency Control

## Table of Contents
1. [SQL Server Lock Types](#sql-server-lock-types)
2. [Update Lock (U)](#update-lock-u)
3. [Intent Locks](#intent-locks)
4. [Isolation Levels](#isolation-levels)
5. [Blocking vs Deadlocks](#blocking-vs-deadlocks)

---

## SQL Server Lock Types

### Overview
Locks are mechanisms SQL Server uses to ensure data integrity and manage concurrent access to database resources.

### Lock Granularity Levels
- **Row**: Individual row locks
- **Page**: 8KB page locks
- **Extent**: 64KB extent locks
- **Table**: Entire table locks
- **Database**: Database-level locks

### Basic Lock Modes

| Lock Mode | Symbol | Description |
|-----------|--------|-------------|
| **Shared** | S | Read lock, allows concurrent reads |
| **Exclusive** | X | Write lock, blocks all other access |
| **Update** | U | Read-for-write lock, prevents deadlocks |

### Shared Lock (S)

**Purpose**: Allows concurrent reading while preventing modifications.

**Characteristics:**
- Acquired for SELECT operations
- Multiple shared locks can coexist
- Blocks exclusive locks
- Released after statement completes (default)

```sql
-- Transaction 1 acquires Shared lock
BEGIN TRANSACTION;
SELECT * FROM Products WHERE ProductID = 1;
-- Shared lock held on row 1

-- Transaction 2 can also read (Shared lock compatible)
SELECT * FROM Products WHERE ProductID = 1;

-- Transaction 3 cannot update (Exclusive blocked)
UPDATE Products SET Price = 100 WHERE ProductID = 1;  -- Waits

COMMIT TRANSACTION;
```

### Exclusive Lock (X)

**Purpose**: Prevents any access to data being modified.

**Characteristics:**
- Acquired for INSERT, UPDATE, DELETE
- Only one exclusive lock allowed
- Blocks all other locks (S, U, X)
- Held until transaction commits

```sql
BEGIN TRANSACTION;
UPDATE Products SET Price = Price * 1.1;
-- Exclusive locks on all updated rows
-- No other transaction can read or write these rows
COMMIT TRANSACTION;
```

### Lock Compatibility Matrix

|  | S | U | X |
|--|---|---|---|
| **S** | Compatible | Compatible | Conflict |
| **U** | Compatible | Conflict | Conflict |
| **X** | Conflict | Conflict | Conflict |

---

## Update Lock (U)

### Purpose
Prevents deadlocks in read-modify-write scenarios.

### The Deadlock Problem
```sql
-- Without Update locks, this can deadlock:

-- Transaction 1                    -- Transaction 2
SELECT * FROM Products              SELECT * FROM Products
WHERE ProductID = 1;                WHERE ProductID = 2;
-- (Shared lock on 1)               -- (Shared lock on 2)

UPDATE Products                     UPDATE Products
SET Price = 100                     SET Price = 200
WHERE ProductID = 2;                WHERE ProductID = 1;
-- Waits for T2's lock              -- Waits for T1's lock
-- DEADLOCK!
```

### Update Lock Solution
```sql
-- Using UPDLOCK hint
BEGIN TRANSACTION;
SELECT * FROM Products WITH (UPDLOCK)
WHERE ProductID = 1;
-- Update lock acquired (blocks other Update/Exclusive)

UPDATE Products SET Price = 100 WHERE ProductID = 1;
-- Lock escalated to Exclusive
COMMIT TRANSACTION;
```

### When Update Locks Are Used
- Serializable transactions
- Cursors with optimistic concurrency
- Explicit UPDLOCK hints
- Automatic deadlock prevention

---

## Intent Locks

### Purpose
Intent locks indicate that a transaction intends to place locks on resources lower in the hierarchy.

### Why Intent Locks Exist
Without intent locks, checking for conflicting locks would require scanning every row, page, and table. Intent locks allow efficient conflict detection at higher levels.

### Intent Lock Types

| Intent Lock | Symbol | Description |
|-------------|--------|-------------|
| **Intent Shared** | IS | Intent to place Shared locks on sub-resources |
| **Intent Exclusive** | IX | Intent to place Exclusive locks on sub-resources |
| **Shared with Intent Exclusive** | SIX | Shared lock on resource + intent for Exclusive on sub-resources |

### Intent Lock Hierarchy
```
Database (IX)
└── Table (IX)
    └── Page (IX)
        └── Row (X)  -- Actual modification
```

### Example Scenario
```sql
-- Transaction acquires intent locks:
UPDATE Users SET Status = 'Active' WHERE UserID = 100;

-- Lock hierarchy:
-- Database: IX (Intent Exclusive)
-- Table Users: IX (Intent Exclusive)
-- Page containing row: IX
-- Row 100: X (Exclusive)
```

### Intent Lock Compatibility

|  | IS | IX | S | SIX | X |
|--|----|----|---|-----|---|
| **IS** | Yes | Yes | Yes | Yes | No |
| **IX** | Yes | Yes | No | No | No |
| **S** | Yes | No | Yes | No | No |
| **SIX** | Yes | No | No | No | No |
| **X** | No | No | No | No | No |

### Viewing Current Locks
```sql
-- Query current locks
SELECT 
    resource_type,
    resource_description,
    request_mode,
    request_status,
    request_session_id
FROM sys.dm_tran_locks
WHERE request_status = 'GRANT';
```

---

## Isolation Levels

### Overview
Isolation levels control the degree to which transactions are isolated from each other, trading consistency for concurrency.

### The Four Isolation Levels

| Level | Dirty Read | Non-Repeatable | Phantom |
|-------|------------|----------------|---------|
| **READ UNCOMMITTED** | Possible | Possible | Possible |
| **READ COMMITTED** | Prevented | Possible | Possible |
| **REPEATABLE READ** | Prevented | Prevented | Possible |
| **SERIALIZABLE** | Prevented | Prevented | Prevented |

### Read Uncommitted
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Can read uncommitted data (dirty reads)
SELECT * FROM Accounts WITH (NOLOCK);
-- Fastest, least safe
```

### Read Committed (Default)
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Only reads committed data
-- Shared locks released after statement
-- May read data twice with different values
```

### Repeatable Read
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Shared locks held until transaction ends
-- Prevents non-repeatable reads
-- Still allows phantom rows
```

### Serializable
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Range locks prevent phantom reads
-- Strictest isolation
-- Lowest concurrency
```

### Isolation Level Examples

#### Lost Update Problem
```sql
-- Initial balance: $1000

-- Transaction 1                  -- Transaction 2
BEGIN TRANSACTION;                 BEGIN TRANSACTION;
SELECT Balance FROM Accounts        SELECT Balance FROM Accounts
WHERE AccountID = 1;  -- $1000      WHERE AccountID = 1;  -- $1000

UPDATE Accounts                     UPDATE Accounts
SET Balance = 1000 + 100            SET Balance = 1000 + 200
WHERE AccountID = 1;                WHERE AccountID = 1;
-- $1100                            -- $1200 (OVERWRITES!)
COMMIT;                             COMMIT;

-- Result: Only $1200, lost $100 deposit!
```

#### Solution with Serializable
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT Balance FROM Accounts WHERE AccountID = 1;
-- Acquires range lock, blocks other transactions

UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 1;
COMMIT;
```

### Snapshot Isolation
```sql
-- Enable at database level
ALTER DATABASE MyDB SET ALLOW_SNAPSHOT_ISOLATION ON;

-- Use in transaction
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
-- Reads consistent point-in-time snapshot
-- No locks on readers!
COMMIT;
```

---

## Blocking vs Deadlocks

### Blocking

**Definition**: One transaction holds a lock that another transaction needs, causing the second to wait.

**Characteristics:**
- Normal behavior
- Resolves when first transaction completes
- Can cause performance issues if excessive
- Can be monitored and analyzed

```sql
-- Transaction 1 (blocks)
BEGIN TRANSACTION;
UPDATE Products SET Price = 100 WHERE ProductID = 1;
-- Exclusive lock held...
-- COMMIT not executed yet

-- Transaction 2 (blocked)
SELECT * FROM Products WHERE ProductID = 1;
-- Waits for Transaction 1 to release lock
```

### Detecting Blocking
```sql
-- Find blocking queries
SELECT 
    blocking.session_id AS BlockingSession,
    blocked.session_id AS BlockedSession,
    blocking_sql.text AS BlockingSQL,
    blocked_sql.text AS BlockedSQL,
    wait.wait_duration_ms / 1000.0 AS WaitSeconds
FROM sys.dm_exec_requests blocked
JOIN sys.dm_exec_sessions blocking ON blocked.blocking_session_id = blocking.session_id
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) blocked_sql
CROSS APPLY sys.dm_exec_sql_text(blocking.most_recent_sql_handle) blocking_sql
JOIN sys.dm_os_waiting_tasks wait ON blocked.session_id = wait.session_id;
```

### Resolving Blocking
1. **Short transactions**: Keep transactions brief
2. **Proper indexing**: Reduce lock duration
3. **Read Committed Snapshot**: Eliminate read blocking
4. **Timeout settings**: Set appropriate timeouts

### Deadlocks

**Definition**: Two or more transactions permanently block each other, each waiting for resources the other holds.

**The Classic Pattern:**
```
Transaction A holds Resource 1, needs Resource 2
Transaction B holds Resource 2, needs Resource 1
DEADLOCK - Neither can proceed
```

**Deadlock Example:**
```sql
-- Transaction A                      -- Transaction B
BEGIN TRANSACTION;                      BEGIN TRANSACTION;
UPDATE Accounts SET...                UPDATE Inventory SET...
WHERE AccountID = 1;                    WHERE ProductID = 100;
-- Holds lock on 1
                                       -- Holds lock on 100
UPDATE Inventory SET...                UPDATE Accounts SET...
WHERE ProductID = 100;                  WHERE AccountID = 1;
-- Needs lock on 100                    -- Needs lock on 1
-- Waits for B                         -- Waits for A
-- DEADLOCK! Both rollback            -- SQL Server kills one
```

### SQL Server Deadlock Resolution
1. Detects deadlock automatically
2. Chooses **victim** based on cost estimate
3. Kills victim transaction with error 1205
4. Victim must retry

### Deadlock Prevention Strategies

#### 1. Consistent Ordering
```sql
-- Always access tables in same order
-- All transactions: Accounts first, then Inventory

BEGIN TRANSACTION;
UPDATE Accounts SET... WHERE AccountID = 1;      -- First
UPDATE Inventory SET... WHERE ProductID = 100;  -- Second
COMMIT;
```

#### 2. Minimize Transaction Duration
```sql
-- Don't hold locks unnecessarily
BEGIN TRANSACTION;
-- Do ALL reads first
-- Do ALL updates together
-- COMMIT immediately
```

#### 3. Use Row-Level Locking
```sql
-- Force row locks
UPDATE Products WITH (ROWLOCK)
SET Price = 100 WHERE ProductID = 1;
```

#### 4. Retry Logic
```sql
-- Application-level retry
int retryCount = 0;
while (retryCount < 3)
{
    try {
        ExecuteTransaction();
        break;
    }
    catch (SqlException ex) when (ex.Number == 1205)
    {
        retryCount++;
        Thread.Sleep(100 * retryCount);
    }
}
```

### Monitoring Deadlocks
```sql
-- Extended Events trace for deadlocks
CREATE EVENT SESSION [DeadlockMonitor] ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.ring_buffer;
GO
ALTER EVENT SESSION [DeadlockMonitor] ON SERVER STATE = START;
```

---

## Best Practices Summary

| Practice | Benefit |
|----------|---------|
| Keep transactions short | Reduce blocking duration |
| Use appropriate isolation | Balance consistency/performance |
| Access resources in order | Prevent deadlocks |
| Enable Read Committed Snapshot | Eliminate reader blocking |
| Add retry logic | Handle deadlocks gracefully |
| Monitor lock waits | Identify bottlenecks |

---

*Source: SQL Server locking documentation, transaction isolation research, and concurrency control best practices.*
