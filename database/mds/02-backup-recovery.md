# Backup & Recovery in SQL Server

## Table of Contents
1. [.bak Files](#bak-files-in-sql-server)
2. [.bak vs .bacpac](#bak-vs-bacpac)
3. [.bak vs Exporting Data](#bak-backup-vs-exporting-data)
4. [Backup Types](#backup-types)
5. [Restore Strategies](#restore-strategies)

---

## .bak Files in SQL Server

### Definition
A `.bak` file is a SQL Server native backup file format that stores complete database backups, including schema, data, indexes, and transaction log information.

### What Contains a .bak File
- **Database schema**: Tables, views, stored procedures, functions
- **Data**: All row data from user tables
- **Indexes**: Both clustered and non-clustered indexes
- **Transaction log**: Point-in-time recovery information
- **Permissions**: Users, roles, and security settings
- **Database settings**: Compatibility level, recovery model, options

### Backup Types Supported
1. **Full Backup**: Complete database copy
2. **Differential Backup**: Changes since last full backup
3. **Transaction Log Backup**: Transaction log records since last log backup

### Creating a .bak File
```sql
-- Full database backup
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH FORMAT, COMPRESSION;

-- Differential backup
BACKUP DATABASE [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Diff.bak'
WITH DIFFERENTIAL;

-- Transaction log backup
BACKUP LOG [MyDatabase]
TO DISK = 'C:\Backups\MyDatabase_Log.trn';
```

### Restoring from .bak
```sql
-- Restore full backup
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH REPLACE;

-- Restore with differential
RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Full.bak'
WITH NORECOVERY;

RESTORE DATABASE [MyDatabase]
FROM DISK = 'C:\Backups\MyDatabase_Diff.bak'
WITH RECOVERY;
```

---

## .bak vs .bacpac

### .bacpac Definition
A `.bacpac` file is a data-tier application package containing database schema and data in a portable format, designed for migration between SQL Server instances and Azure SQL Database.

### Key Differences

| Aspect | .bak | .bacpac |
|--------|------|---------|
| **Format** | SQL Server native binary | Open ZIP-based format |
| **Content** | Complete database (including unused space) | Schema + data only |
| **Size** | Larger (includes empty pages) | Smaller (compressed data) |
| **Cross-platform** | SQL Server only | SQL Server + Azure SQL |
| **Log backup** | Supported | Not supported |
| **Filegroups** | Preserved | Not supported |
| **Permissions** | Included | Must be scripted separately |
| **Point-in-time** | Yes (with log backups) | No (snapshot only) |

### When to Use .bak
- Same SQL Server version restores
- Point-in-time recovery required
- Full database cloning needed
- Transaction log backups part of strategy
- Complex filegroup configurations

### When to Use .bacpac
- Migrating to Azure SQL Database
- Schema-only deployments
- Smaller transfer size needed
- Deploying DAC (Data-tier Applications)
- Version-agnostic migrations

### Creating a .bacpac
```sql
-- Export via SSMS or sqlpackage.exe
sqlpackage.exe /Action:Export \
  /SourceConnectionString:"Server=.;Database=MyDB;..." \
  /TargetFile:"C:\Backups\MyDB.bacpac"
```

---

## .bak Backup vs Exporting Data

### Data Export Methods
| Method | Tool | Output | Use Case |
|--------|------|--------|----------|
| **BCP** | Command-line | Flat files | Bulk data transfer |
| **SSIS** | Visual Studio | Various formats | ETL processes |
| **Export Wizard** | SSMS | CSV, Excel, etc. | Ad-hoc exports |
| **SELECT INTO** | SQL | New table | Data subset copies |
| **Generate Scripts** | SSMS | SQL scripts | Schema + data |

### Comparison Table

| Feature | .bak Backup | Data Export |
|---------|-------------|-------------|
| **Completeness** | Full database | Selected objects/data |
| **Recovery** | Full point-in-time recovery | Manual reconstruction |
| **Speed** | Fast (page-level) | Slower (row-level) |
| **Size** | Compressed binary | Varies by format |
| **Portability** | SQL Server specific | Universal formats |
| **Indexes** | Preserved | Must recreate |
| **Permissions** | Preserved | Not included |
| **Automation** | Built-in scheduling | Custom scripts |

### Use Case Scenarios

**Choose .bak when:**
- Disaster recovery planning
- Regular backup strategy
- Database cloning for testing
- Rapid restoration needed
- Complete environment preservation

**Choose Export when:**
- Sharing data with non-SQL Server systems
- Data warehousing/ETL
- Specific table/data subsets needed
- Cross-platform data exchange
- Application data feeds

---

## Backup Types

### Full Backup
Captures the entire database at a point in time.
```sql
BACKUP DATABASE [DatabaseName]
TO DISK = 'Path\Full.bak'
WITH COMPRESSION, CHECKSUM;
```

**Characteristics:**
- Base for differential and log backups
- Longest backup time
- Largest backup size
- Required for any recovery strategy

### Differential Backup
Captures only data changed since the last full backup.
```sql
BACKUP DATABASE [DatabaseName]
TO DISK = 'Path\Diff.bak'
WITH DIFFERENTIAL, COMPRESSION;
```

**Characteristics:**
- Smaller than full backup
- Faster than full backup
- Chains to last full backup
- Accumulates changes (each diff since last full)

### Transaction Log Backup
Captures all log records since the last log backup.
```sql
BACKUP LOG [DatabaseName]
TO DISK = 'Path\Log.trn';
```

**Characteristics:**
- Enables point-in-time recovery
- Required in Full/Bulk-Logged recovery models
- Truncates inactive log portions
- Can be very frequent (every minute)

### Copy-Only Backup
Special backup that doesn't interrupt the differential chain.
```sql
BACKUP DATABASE [DatabaseName]
TO DISK = 'Path\Copy.bak'
WITH COPY_ONLY;
```

### File/Filegroup Backup
Backs up specific filegroups for large databases.
```sql
BACKUP DATABASE [DatabaseName]
FILEGROUP = 'ReadOnlyData'
TO DISK = 'Path\FG.bak';
```

---

## Restore Strategies

### Restore Chain
The sequence of backups required to restore a database to a point in time.

**Typical Chain:**
1. Full backup (RESTORE WITH NORECOVERY)
2. Differential backup (optional, WITH NORECOVERY)
3. Log backups in sequence (WITH NORECOVERY)
4. Final log backup (WITH RECOVERY)

### Point-in-Time Recovery
Restoring to a specific moment using log backups.
```sql
RESTORE DATABASE [MyDB]
FROM DISK = 'Full.bak'
WITH NORECOVERY;

RESTORE LOG [MyDB]
FROM DISK = 'Log1.trn'
WITH NORECOVERY;

RESTORE LOG [MyDB]
FROM DISK = 'Log2.trn'
WITH STOPAT = '2024-01-15 14:30:00', RECOVERY;
```

### Tail-Log Backup
Backing up the active transaction log before restore (to capture latest transactions).
```sql
BACKUP LOG [MyDB]
TO DISK = 'TailLog.trn'
WITH NORECOVERY;
-- Allows restore to point of failure
```

### RPO vs RTO

| Metric | Definition | Typical Target |
|--------|------------|----------------|
| **RPO** (Recovery Point Objective) | Maximum acceptable data loss | 15 minutes, 1 hour |
| **RTO** (Recovery Time Objective) | Maximum acceptable downtime | 1 hour, 4 hours |

### Recovery Models

| Model | Log Behavior | Use Case |
|-------|--------------|----------|
| **Simple** | Auto-truncated | Development, data warehouses |
| **Full** | Log backups required | Production OLTP |
| **Bulk-Logged** | Minimal logging for bulk ops | Bulk operations with Full model |

### Best Practices
1. **Test restores regularly** - Backups are only valid if restorable
2. **Use CHECKSUM** - Validates backup integrity
3. **Backup to multiple locations** - On-site and off-site
4. **Monitor backup history** - Alert on failures
5. **Document recovery procedures** - Time is critical during disasters
6. **Secure backup files** - Encrypt sensitive backups

---

*Source: SQL Server documentation, research compilation from Microsoft Learn and industry best practices.*
