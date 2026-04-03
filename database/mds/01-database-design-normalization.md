# Database Design & Normalization

## Table of Contents
1. [Primary Keys: Number vs GUID](#primary-keys-number-vs-guid)
2. [BCNF (Boyce-Codd Normal Form)](#bcnf-boyce-codd-normal-form)
3. [Fourth Normal Form (4NF)](#fourth-normal-form-4nf)
4. [Fifth Normal Form (5NF)](#fifth-normal-form-5nf)

---

## Primary Keys: Number vs GUID

### Overview
Choosing between numeric (auto-incrementing integers) and GUID (Globally Unique Identifiers) primary keys is a fundamental database design decision with significant performance and architectural implications.

### Number (Auto-Incrementing Integer)

**Advantages:**
- **Compact storage**: 4-8 bytes vs 16 bytes for GUID
- **Sequential access**: Physically ordered inserts minimize page fragmentation
- **Fast lookups**: Smaller index size improves cache efficiency
- **Human-readable**: Easier for debugging and support
- **Lower memory usage**: Better buffer pool utilization

**Disadvantages:**
- **Not globally unique**: Difficult in distributed systems
- **Sequential ID exposure**: Can reveal business volume (ID 1,000,000 = ~1M records)
- **Merge conflicts**: Complex when consolidating databases
- **Hotspot contention**: Last page contention in high-insert scenarios

### GUID (Globally Unique Identifier)

**Advantages:**
- **Global uniqueness**: No coordination needed across distributed systems
- **Security**: Non-sequential IDs don't reveal table size
- **Merge-friendly**: Easy database consolidation
- **Pre-generation**: IDs can be created before insertion

**Disadvantages:**
- **Storage overhead**: 16 bytes vs 4-8 bytes
- **Fragmentation**: Random inserts cause severe page fragmentation
- **Performance**: Larger indexes, more memory pressure
- **Non-sequential**: Poor cache locality

### NEWSEQUENTIALID
SQL Server's solution to GUID fragmentation. Generates sequentially ordered GUIDs while maintaining global uniqueness. Reduces fragmentation while keeping the benefits of GUIDs.

### UUID Version 7
Modern standard (RFC 9562) that embeds a timestamp in the UUID, ensuring natural sort order by time. Combines global uniqueness with sequential characteristics for better database performance.

### Best Practice Recommendation
| Scenario | Recommended Key |
|----------|----------------|
| Single database, single application | BIGINT IDENTITY |
| High-insert OLTP | INT/BIGINT with application-level sharding |
| Distributed/microservices | UUID v7 or NEWSEQUENTIALID |
| Security-sensitive sequential exposure | GUID/UUID |
| Multi-region active-active | UUID v7 |

---

## BCNF (Boyce-Codd Normal Form)

### Definition
BCNF is a stricter version of 3NF. A table is in BCNF if for every functional dependency X → Y, X must be a superkey (uniquely identifies each row).

### Formal Definition
For every functional dependency X → Y:
- X must be a superkey, OR
- Y is a prime attribute (part of any candidate key)

### Comparison with 3NF
- **3NF**: Allows transitive dependencies if the dependent is prime
- **BCNF**: Eliminates all dependencies on non-superkeys
- Every BCNF table is in 3NF, but not vice versa

### Example Violation
```
Student(StudentID, Subject, Professor)
- Professor → Subject (each professor teaches one subject)
- StudentID, Subject → Professor

Violation: Professor determines Subject, but Professor is not a superkey.
```

### Solution
Split into two tables:
```sql
ProfessorSubject(Professor, Subject)
StudentEnrollment(StudentID, Professor)
```

### When to Stop at 3NF
- When the functional dependency preserves important information
- When decomposition would lose critical join dependencies
- When query performance is more important than the dependency anomaly

---

## Fourth Normal Form (4NF)

### Definition
A table is in 4NF if it is in BCNF and contains no multi-valued dependencies (MVDs) other than trivial ones.

### Multi-Valued Dependency
X →→ Y (X multi-determines Y) when:
- For each X value, there exists a well-defined set of Y values
- This set is independent of other attributes in the table

### Example Violation
```
EmployeeSkills(EmployeeID, Skill, Language)
- Employee 1 knows SQL and Python
- Employee 1 speaks English and Arabic

Problem: Skills and Languages are independent multi-valued attributes.
Storing all combinations creates redundancy.
```

### 4NF Violation Pattern
When one row implies multiple independent collections:
- Employee has multiple skills
- Employee speaks multiple languages
- Skills don't determine languages and vice versa

### Solution
Decompose into separate tables:
```sql
EmployeeSkills(EmployeeID, Skill)
EmployeeLanguages(EmployeeID, Language)
```

### When to Apply 4NF
- When attributes are truly independent multi-valued properties
- When the combination produces significant redundancy
- For entity-attribute-value patterns

---

## Fifth Normal Form (5NF)

### Definition
Also known as **Projected-Join Normal Form (PJ/NF)**. A table is in 5NF if it is in 4NF and cannot be decomposed into smaller tables without loss of information (lossless join decomposition is not possible without losing semantics).

### Purpose
Addresses **join dependencies** that aren't implied by candidate keys. Prevents cases where a table can be reconstructed from projections but shouldn't be decomposed.

### When Decomposition Loses Information
```
SupplierProjectPart(Supplier, Project, Part)
- Supplier A supplies Part P to Project J1
- Supplier A supplies to Project J2
- Supplier B also supplies Part P to Project J1

If we decompose into:
- SupplierProject(Supplier, Project)
- SupplierPart(Supplier, Part)
- ProjectPart(Project, Part)

We can reconstruct the original, but we lose the constraint that 
a supplier must supply a specific part to work on a specific project.
```

### 5NF Requirement
If a table can be decomposed and reconstructed via join, it should only remain decomposed if the join dependencies reflect actual business constraints.

### Practical Application
5NF is rarely needed in practice. Most real-world databases operate effectively at 3NF or BCNF. Apply 5NF when:
- Complex many-to-many-to-many relationships exist
- Decomposition appears possible but loses semantic constraints
- Working with temporal or versioned data with complex constraints

---

## Summary Table

| Normal Form | Addresses | Key Constraint |
|-------------|-----------|----------------|
| 1NF | Repeating groups | Atomic values |
| 2NF | Partial dependencies | Full key dependency |
| 3NF | Transitive dependencies | No non-key dependencies |
| BCNF | All key dependencies | Only superkey dependencies |
| 4NF | Multi-valued dependencies | Independent MVDs separated |
| 5NF | Join dependencies | Lossless join preservation |

---

*Source: Research compilation from database design principles and SQL Server documentation.*
