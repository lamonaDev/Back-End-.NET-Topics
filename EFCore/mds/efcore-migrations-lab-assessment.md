# EF Core 10 - Migrations Lab Assessment

A hands-on assessment covering real-world migration scenarios, team collaboration edge cases, and database lifecycle management in a Console Application context.

---

## Assessment Overview

| Attribute | Value |
|-----------|-------|
| **Technology** | .NET 10, EF Core 10 |
| **Application Type** | Console Application |
| **Scenarios** | 13 |
| **Total Points** | 100 |
| **Time Limit** | 60 Minutes |
| **Constraints** | No Relationships, No Fluent API |

---

## Table of Contents

1. [Initial Setup - 3 Table Project](#question-1-initial-setup)
2. [Pending Changes and Database Sync](#question-2-pending-changes)
3. [Migration Execution Methods](#question-3-migration-execution)
4. [Accident Scenarios - Deleted Migrations Folder](#question-4-accident-scenarios)
5. [Migration File Deletion](#question-5-migration-deletion)
6. [Reverting Multiple Migrations](#question-6-reverting-migrations)
7. [Development Revert Strategies](#question-7-revert-strategies)
8. [Multiple DbContexts](#question-8-multiple-contexts)
9. [Empty Migrations Research](#question-9-empty-migrations)
10. [SQL Script Generation](#question-10-sql-scripts)
11. [History Table Recovery](#question-11-history-recovery)
12. [Team Git Conflicts](#question-12-git-conflicts)
13. [Pull Before Migration](#question-13-pull-routine)

---

## Question 1: Initial Setup

### Scenario
Create a new Console App (.NET 10) project. Install NuGet packages:
- `Microsoft.EntityFrameworkCore.SqlServer`
- `Microsoft.EntityFrameworkCore.Tools`
- `Microsoft.EntityFrameworkCore.Design`

### Models (No Relationships, No Annotations)

```csharp
// Student.cs
public class Student {
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public DateTime EnrolledAt { get; set; }
}

// Course.cs
public class Course {
    public int Id { get; set; }
    public string Title { get; set; }
    public int DurationHours { get; set; }
}

// Instructor.cs
public class Instructor {
    public int Id { get; set; }
    public string FullName { get; set; }
    public string Specialty { get; set; }
}

// AppDbContext.cs
public class AppDbContext : DbContext {
    public DbSet<Student> Students { get; set; }
    public DbSet<Course> Courses { get; set; }
    public DbSet<Instructor> Instructors { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlServer(
            "Server=.;Database=MigrationLabDb;Trusted_Connection=True;TrustServerCertificate=True");
}
```

### Task
Run `Add-Migration InitialCreate` then `Update-Database`.

### Questions and Solutions

**Q1a: What files were generated and what is inside each one?**

Three files are generated:

1. **Migration File (e.g., `20240101120000_InitialCreate.cs`)**: Contains `Up()` method to create tables and `Down()` method to drop them.
2. **Designer File (e.g., `20240101120000_InitialCreate.Designer.cs`)**: Contains migration metadata and model state at the time of creation.
3. **ModelSnapshot (`AppDbContextModelSnapshot.cs`)**: Represents the current model state used for future diff comparisons.

**Operation Sequence:**
```
Add-Migration InitialCreate
    |
    +--> Generates 3 files
            |
            +--> Update-Database
                    |
                    +--> Creates __EFMigrationsHistory table
                    +--> Executes Up() methods
                    +--> Records migration in history table
```

**Q1b: What is the ModelSnapshot.cs file and why does EF Core need it?**

The `ModelSnapshot.cs` represents the current state of the model as EF Core understands it. When `Add-Migration` runs, EF compares the current model against this snapshot to determine what changed and generate the migration diff. Without it, EF cannot know the previous state and cannot generate accurate incremental migrations.

**Q1c: What is the __EFMigrationsHistory table? Who creates it? What does it store?**

- **Purpose**: Tracks which migrations have been applied to the database
- **Creator**: EF Core automatically creates this table when `Update-Database` runs for the first time
- **Contents**: Stores migration IDs and EF Core product version for every applied migration
- **Function**: EF uses it to prevent re-running already-applied migrations

---

## Question 2: Pending Changes and Database Sync

### Scenario
Database is already created with all 3 tables. Add new properties to the Student class:

```csharp
// Add to Student class:
public string PhoneNumber { get; set; }
public bool IsActive { get; set; }
```

**Important**: Do NOT run `Add-Migration` yet. Go straight to `Update-Database`.

### Questions and Solutions

**Q2a: What happened when you ran Update-Database with un-migrated model changes?**

Nothing changes in the database. `Update-Database` only applies existing pending migrations from the Migrations folder. Since no new `Add-Migration` was run for the new properties, EF Core has no migration file describing the new columns. The database remains unchanged.

**Key Principle**: `Update-Database` does NOT scan your model classes directly. It only reads and applies migration files.

**Q2b: Now run Add-Migration AddStudentFields followed by Update-Database. What changed?**

**Operation Sequence:**
```
Modified Model (PhoneNumber, IsActive)
    |
    +--> Add-Migration AddStudentFields
            |
            +--> EF compares current model vs snapshot
            +--> Generates migration with AddColumn calls
            +--> Updates ModelSnapshot
                    |
                    +--> Update-Database
                            |
                            +--> Reads new migration file
                            +--> Executes Up() - adds columns
                            +--> Records migration in history
```

**Q2c: What is the key lesson about the order of commands?**

The correct workflow is always:

```
Modify Model --> Add-Migration --> Update-Database
```

`Update-Database` only applies migration files that already exist. You must always create a migration first to capture the model changes, then apply it.

---

## Question 3: Migration Execution Methods

### Scenario
A teammate wants to replace the `Update-Database` CLI call with automatic migration in code:

```csharp
using var db = new AppDbContext();
await db.Database.MigrateAsync();
```

### Questions and Solutions

**Q3a: What is the difference between Update-Database (CLI) and MigrateAsync() / Migrate() (code)?**

| Aspect | Update-Database (CLI) | MigrateAsync() / Migrate() (Code) |
|--------|----------------------|-----------------------------------|
| Timing | Design-time (developer-controlled) | Runtime (automatic on app start) |
| Execution | Manual via Package Manager Console or dotnet CLI | Called from Program.cs |
| Control | Explicit, intentional | Automatic, implicit |
| Typical Use | Development, deployment pipelines | Some development scenarios |

**Q3b: What is the difference between MigrateAsync() and Migrate()? Which should you prefer?**

- **MigrateAsync()**: Asynchronous version using async/await. Does not block the thread during database operations.
- **Migrate()**: Synchronous version. Blocks the calling thread until completion.

**Recommendation**: In modern .NET with async-first design, `MigrateAsync()` should be preferred to avoid blocking the startup pipeline.

**Q3c: Should MigrateAsync() be used in production? What are the risks?**

**Opinion**: Using `MigrateAsync()` in a Console App for production is generally risky.

**Risks:**
1. If the migration has a bug, the app crashes on startup
2. Console Apps may be scheduled to run many times a day, causing repeated migration attempts
3. No opportunity for DBA review before schema changes
4. Difficult to rollback if issues occur

**Best Practice**: Use a dedicated migration step via CLI (`dotnet ef database update`) in the deployment pipeline, not inside the application code.

---

## Question 4: Accident Scenarios - Deleted Migrations Folder

### Scenario A: Migrations Not Applied
Added a migration `AddCourseDescription` but have not run `Update-Database` yet. Accidentally deleted the entire Migrations folder.

### Questions and Solutions

**Q4a: What is the impact? What should you do to recover?**

**Impact**: Since the migrations were never applied to the database, the database schema is still in sync with the previous state. The migration files can be safely regenerated.

**Recovery Steps:**
1. Run `Add-Migration` again (e.g., `Add-Migration InitialCreate` or the appropriate name)
2. This regenerates the Migrations folder from scratch
3. The ModelSnapshot is gone, so EF starts fresh
4. Run `Update-Database` to apply

**Q4b: Migrations Already Applied - What happens when you run Add-Migration?**

**Scenario**: Migrations WERE applied to the database. After deleting the Migrations folder, run `Add-Migration NewMigration`.

**What Happens:**
- Since the ModelSnapshot is gone, EF Core treats the current model as entirely new
- The new migration generates an `Up()` that tries to CREATE all tables from scratch
- Running `Update-Database` will fail because those tables already exist

**The Danger**: Crash or potential data loss from attempting to recreate existing tables.

**Recovery Steps:**
1. Add the migration to recreate the snapshot
2. Manually empty the `Up()` method and keep `Down()` empty too
3. This creates a baseline migration that syncs the snapshot without touching schema
4. Future migrations will work correctly

---

## Question 5: Migration File Deletion

### Scenario
Ran `Add-Migration AddPhoneField`. Before running `Update-Database`, manually deleted the migration `.cs` file (did NOT use `Remove-Migration`).

### Questions and Solutions

**Q5a: What happens now? What is out of sync?**

The `ModelSnapshot` was already updated to include the new property (`PhoneNumber`) when `Add-Migration` ran. Now:
- The migration file is gone
- The snapshot still reflects the new state

**Problem**: When you try to run `Add-Migration` again for the same change, EF thinks the change is already captured (because snapshot is up to date) and may generate an empty migration.

**Q5b: What is the correct command to remove an unapplied migration?**

**Correct Command**: `Remove-Migration`

**Why**: It removes the migration file AND rolls back the ModelSnapshot to its previous state, keeping everything consistent. Manual deletion only removes the file but leaves the snapshot updated, causing the mismatch.

---

## Question 6: Reverting Multiple Migrations

### Scenario
You have 4 migrations, all applied to the database:
- **M1**: InitialCreate
- **M2**: AddStudentFields
- **M3**: AddCourseDescription
- **M4**: AddInstructorPhone

You realize M3 and M4 were mistakes. Want to revert both.

### Questions and Solutions

**Q6a: Write the exact commands to revert both M3 and M4**

**Command:**
```powershell
Update-Database AddStudentFields
# OR
Update-Database M2
```

**What Happens Internally:**
1. EF calls the `Down()` method of M4 first (reverse order)
2. Then calls the `Down()` method of M3
3. Each `Down()` undoes what the `Up()` did (drops columns, tables, etc.)
4. The `__EFMigrationsHistory` table rows for M3 and M4 are deleted

**Operation Sequence:**
```
Update-Database M2
    |
    +--> EF reads migration history
    +--> Identifies M4 and M3 need rollback
    +--> Executes Down() for M4
    +--> Executes Down() for M3
    +--> Removes M4 and M3 from history table
    +--> Database now matches M2 state
```

**Q6b: After reverting, what do you do with the M3 and M4 migration files?**

You can now use `Remove-Migration` to remove M4 first, then `Remove-Migration` again to remove M3.

**Important**: Do NOT manually delete the files. Use `Remove-Migration` to ensure the snapshot is also properly reverted.

---

## Question 7: Development Revert Strategies

### Scenario
Team debate on reverting the last migration:
- Teammate A: "Add a new migration that manually reverses the change"
- Teammate B: "Run `Update-Database PreviousMigrationName` to execute Down()"

### Questions and Solutions

**Q7a: Which approach is correct for development?**

**Teammate B is correct** for development scenarios.

**Correct Command:**
```powershell
Update-Database PreviousMigrationName
```

**Full Process:**
1. Run `Update-Database PreviousMigrationName` to trigger `Down()`
2. Then use `Remove-Migration` to delete the migration file
3. This reverts the snapshot as well
4. Clean and uses already-written `Down()` logic

**Q7b: When might Teammate A's approach (creating a reversal migration) make sense?**

In **production deployments**, running `Down()` migrations is risky because it may delete columns or tables containing real customer data.

**Alternative Approach**:
- Write a new forward migration that explicitly reverses the unwanted change
- For example, drop the column that was added
- This is safer because:
  - It moves migration history forward
  - Can be reviewed in a PR
  - Does not require connecting to live DB interactively

---

## Question 8: Multiple DbContexts

### Scenario
Project has two DbContexts in the same project: `AppDbContext` (main data) and `LogDbContext` (audit logs). Run `Add-Migration AddNewTable` without specifying `-Context`.

### Questions and Solutions

**Q8a: What happens? What error or behavior does EF Core show?**

EF Core throws an error:
```
More than one DbContext was found. Specify which one to use.
```

The migration is not created. EF requires explicit context specification when multiple contexts exist.

**Q8b: Write the correct command for AppDbContext. Should each context have its own Migrations folder?**

**Correct Command:**
```powershell
Add-Migration AddNewTable -Context AppDbContext
```

**Separate Folders**: Yes, each context should have its own Migrations folder to avoid conflicts.

**Configure Output Directory:**
```powershell
Add-Migration AddNewTable -Context AppDbContext -OutputDir Migrations/App
```

This keeps migrations isolated per context and prevents file collisions.

---

## Question 9: Empty Migrations Research

### Scenario
Added a new property `public string Address { get; set; }` to the Student class. Ran `Add-Migration AddStudentAddress`. The generated migration has empty `Up()` and `Down()` methods.

### Questions and Solutions

**Q9a: List at least 3 reasons why EF Core may generate an empty migration**

1. **File Not Saved**: Added the property to the class but forgot to save the file before running `Add-Migration`
2. **Snapshot Already Updated**: The property was already captured in the ModelSnapshot from a previous `Add-Migration` that was not cleaned up
3. **Unsupported Type**: In a Console App, the property type is not supported or mappable by EF Core
4. **Entity Not Registered**: The entity is not registered as a DbSet in AppDbContext, so EF Core ignores it
5. **Ignored Property**: The change was made to a property EF Core does not track (static, computed, or explicitly ignored)

**Q9b: Is an empty migration always a mistake?**

No, an empty migration can be intentional.

**Use Case**: After recovering from a deleted Migrations folder where the database already has the correct schema, generate an empty migration as a baseline to re-sync the ModelSnapshot with the database without touching the schema. It acts as a marker in migration history.

---

## Question 10: SQL Script Generation

### Scenario
Console App deployed to production. DBA requires a SQL file with all changes for review before execution via SSMS.

### Questions and Solutions

**Q10a: What command generates a SQL script from migrations?**

**Command (Package Manager Console):**
```powershell
Script-Migration -From InitialCreate -To AddStudentFields -Output ./scripts/migration.sql -Idempotent
```

**Command (dotnet CLI):**
```bash
dotnet ef migrations script InitialCreate AddStudentFields -o ./scripts/migration.sql --idempotent
```

**Key Options:**
- `-From`: Starting migration (optional, defaults to first)
- `-To`: Ending migration
- `-Output`: File path to save the script
- `-Idempotent`: Generates script safe to run multiple times

**Q10b: What does the -Idempotent flag do and why is it important?**

The `-Idempotent` flag generates a SQL script that:
- Checks the `__EFMigrationsHistory` table before applying each migration
- Skips migrations that are already applied
- Can be safely run multiple times without errors

**Why Important for Production:**
- DBAs may need to run the script multiple times
- Can be applied across different environments without modification
- Prevents duplicate operation errors
- Allows for safer automated deployment scripts

---

## Question 11: History Table Recovery

### Scenario
Junior developer drops the `__EFMigrationsHistory` table via SSMS. Database still has all real tables (Students, Courses, Instructors). Team tries to run `Update-Database`.

### Questions and Solutions

**Q11a: What happens when Update-Database is run?**

1. EF Core tries to re-create the `__EFMigrationsHistory` table (it always ensures it exists)
2. EF checks which migrations have been applied — finding none recorded
3. EF tries to run ALL migrations from scratch, starting with `InitialCreate`
4. **Failure**: Errors occur because tables (Students, Courses, Instructors) already exist

**Q11b: How do you recover?**

**Recovery Steps:**
1. Manually re-insert the migration records into `__EFMigrationsHistory` using SQL INSERT statements
2. For each previously applied migration, insert the migration ID and EF Core product version
3. Example:
```sql
INSERT INTO __EFMigrationsHistory (MigrationId, ProductVersion)
VALUES ('20240101120000_InitialCreate', '10.0.0');
```
4. After that, `Update-Database` will see them as already applied and skip them

---

## Question 12: Team Git Conflicts

### Scenario
Team of 3 working on same Console App. Main branch has migrations up to `M2_AddStudentFields`. Both Ahmed and Sara pull main and start working on separate feature branches.

**What Happened:**
- Ahmed (branch: `feature/add-phone`): Adds `PhoneNumber` to Student, runs `Add-Migration M3_AddPhone`, pushes
- Sara (branch: `feature/add-grade`): Adds `Grade` to Student, runs `Add-Migration M3_AddGrade`, pushes
- Both merged into main. Now main has two M3 migrations with conflicting ModelSnapshots.

### Questions and Solutions

**Q12a: What exactly is the conflict?**

Both migrations were built from the same M2 snapshot state. Now there are:
- Two migration files both named M3 (or with overlapping timestamps)
- Two different versions of ModelSnapshot — one including PhoneNumber, one including Grade

**The Real Problem**: There can only be one `ModelSnapshot.cs` file. Git will show a merge conflict in `ModelSnapshot.cs` that cannot be auto-resolved correctly.

**Q12b: Walk through the exact steps to resolve this conflict**

**Resolution Steps:**
1. Decide whose migration goes first (e.g., Ahmed's `M3_AddPhone`)
2. Resolve the `ModelSnapshot.cs` merge conflict manually — the snapshot must reflect the state after both changes
3. Sara's migration (`M3_AddGrade`) must be renamed to `M4_AddGrade` and regenerated:
   - Sara runs `Remove-Migration`
   - Sara pulls main (with Ahmed's M3)
   - Sara runs `Add-Migration M4_AddGrade` again
4. The snapshot now reflects both changes
5. Push and run `Update-Database`

**Q12c: What team workflow would prevent this conflict?**

**Prevention Strategies:**
1. Only one person owns migrations at a time — communicate before adding
2. Always pull latest main before running `Add-Migration`
3. Use short-lived feature branches and merge frequently
4. Create migrations only on main/develop branch, not on feature branches
5. Use branch protection rules to require PR review before merging migration changes

---

## Question 13: Pull Before Migration

### Scenario
Pulled latest code from GitHub. Teammate pushed 3 new migrations while you were away. Running the Console App produces:
```
"The model backing the context has changed since the database was created."
```

### Questions and Solutions

**Q13a: What caused this error?**

Your local database was last updated before the 3 new migrations were applied. The ModelSnapshot in the code reflects the 3 new migrations, but the local database schema is behind. EF Core detects the mismatch between the model state and the database structure.

**Q13b: What is the exact command and what should be part of your routine?**

**Command:**
```powershell
Update-Database
```

**Team Routine:**
After every `git pull`, check if there are new migration files. If so, run `Update-Database` immediately before starting development. Some teams document this in PR descriptions or README.

**Q13c: Different situation — you added your own migration BEFORE pulling**

**Problem**: Your local migration was built from an older ModelSnapshot that does not include your teammate's migration. Now there is a conflict — your migration and your teammate's migration both branch from the same snapshot base.

**Fix Steps:**
1. Run `Remove-Migration` to delete your local unapplied migration (reverts the snapshot)
2. Run `Update-Database` to apply the pulled migrations from your teammate
3. Re-run `Add-Migration` with your change on top of the updated snapshot
4. Push your migration

**Result**: Linear migration history with consistent snapshot.

---

## Command Reference

### Essential Commands

| Command | Purpose |
|---------|---------|
| `Add-Migration <Name>` | Creates a new migration based on model changes |
| `Update-Database` | Applies all pending migrations to the database |
| `Update-Database <MigrationName>` | Reverts to a specific migration |
| `Remove-Migration` | Removes the last unapplied migration |
| `Script-Migration` | Generates SQL script from migrations |

### Command Options

| Option | Description |
|--------|-------------|
| `-Context <Name>` | Specifies which DbContext to use |
| `-OutputDir <Path>` | Specifies where to place migration files |
| `-Idempotent` | Generates script safe to run multiple times |

### Runtime Methods

| Method | Description |
|--------|-------------|
| `db.Database.Migrate()` | Synchronous migration execution |
| `db.Database.MigrateAsync()` | Asynchronous migration execution (preferred) |

---

## Key Principles Summary

1. **Correct Order**: Modify Model -> Add-Migration -> Update-Database
2. **Snapshot Importance**: Never manually delete migration files; use Remove-Migration
3. **Team Workflow**: Always pull latest before creating migrations
4. **Production Safety**: Use CLI commands in deployment pipelines, not MigrateAsync()
5. **Recovery**: If history table is lost, manually restore history records
6. **Git Conflicts**: Resolve by re-creating migrations on top of latest main

---

## Assessment Criteria Summary

| Question | Points | Key Focus |
|----------|--------|-----------|
| Q1 | 8 pts | Initial setup, migration files, history table |
| Q2 | 8 pts | Pending changes, command order |
| Q3 | 7 pts | CLI vs runtime, async vs sync |
| Q4 | 8 pts | Accident recovery |
| Q5 | 6 pts | Remove-Migration importance |
| Q6 | 8 pts | Reverting migrations |
| Q7 | 6 pts | Dev vs production strategies |
| Q8 | 5 pts | Multiple DbContexts |
| Q9 | 6 pts | Empty migrations research |
| Q10 | 6 pts | SQL scripts |
| Q11 | 6 pts | History table recovery |
| Q12 | 10 pts | Git conflicts |
| Q13 | 10 pts | Pull routines |
| **Total** | **100 pts** | |

---

*Document generated from EF Core 10 Migrations Lab Assessment*
*Source: Console Application scenario-based quiz*
