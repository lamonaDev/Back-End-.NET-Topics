# .NET Backend Engineering: Advanced Deep Dives
 
A comprehensive technical repository documenting advanced C# mechanics, .NET internals, and database optimization strategies. This project serves as a structured knowledge base for building scalable, high-performance backend systems.
 
---
 
## Table of Contents
 
- [1. C# Fundamentals](#1-c-fundamentals)
  - [1.1 Error Handling](#11-error-handling)
  - [1.2 Language Fundamentals](#12-language-fundamentals)
  - [1.3 Type System](#13-type-system)
  - [1.4 Strings](#14-strings)
  - [1.5 DateTime](#15-datetime)
  - [1.6 Memory Management](#16-memory-management)
  - [1.7 Best Practices](#17-best-practices)
- [2. Object-Oriented Programming](#2-object-oriented-programming)
  - [2.1 Encapsulation](#21-encapsulation)
  - [2.2 Abstraction](#22-abstraction)
  - [2.3 Inheritance](#23-inheritance)
  - [2.4 Polymorphism](#24-polymorphism)
  - [2.5 Class Relationships](#25-class-relationships)
  - [2.6 Interview Questions](#26-interview-questions)
- [3. Advanced C# & Concurrency](#3-advanced-c--concurrency)
  - [3.1 Language Features](#31-language-features)
  - [3.2 Asynchronous Programming](#32-asynchronous-programming)
  - [3.3 Threading & Synchronization](#33-threading--synchronization)
  - [3.4 Memory & Performance](#34-memory--performance)
- [4. LINQ](#4-linq)
- [5. Database Architecture](#5-database-architecture)
  - [5.1 Design & Structure](#51-design--structure)
  - [5.2 Performance & Optimization](#52-performance--optimization)
  - [5.3 Concurrency & Reliability](#53-concurrency--reliability)
  - [5.4 Advanced Topics](#54-advanced-topics)
- [6. EF Core](#6-ef-core)
  - [6.1 Micro ORM vs Full ORM](#61-micro-orm-vs-full-orm)
  - [6.2 Expression Trees](#62-expression-trees)
  - [6.3 Migrations vs Scaffolding](#63-migrations-vs-scaffolding)
  - [6.4 DbContext Deep Dive](#64-dbcontext-deep-dive)
  - [6.5 Dependency Injection](#65-dependency-injection)
  - [6.6 Connection Pool Exhaustion](#66-connection-pool-exhaustion)
  - [6.7 Migrations Lab Assessment](#67-migrations-lab-assessment)
  - [6.8 DbSet, Concurrency & Relationships](#68-dbset-concurrency-and-relationships)
  - [6.9 Advanced Mapping](#69-advanced-mapping)
  - [6.10 Query Performance](#610-query-performance)
  - [6.11 Query Translation & Debugging](#611-query-translation-and-debugging)
  - [6.12 LINQ, Expressions & Functions](#612-linq-expressions-and-functions)
  - [6.13 Interceptors & Raw SQL](#613-interceptors-and-raw-sql)
  - [6.14 .NET Ecosystem Tools](#614-dotnet-ecosystem-tools)
- [7. Quality Assurance](#7-quality-assurance)
  - [7.1 Unit Testing](#71-unit-testing)
  - [7.2 Mocking](#72-mocking)
- [8. Project Overview](#8-project-overview)
---
 
## 1. C# Fundamentals
 
Core language mechanics and foundational concepts.
 
### 1.1 Error Handling
 
| Document | Path |
|----------|------|
| [Error Handling](./C%23_Basics/mds/01-error-handling.md) | `C#_Basics/mds/01-error-handling.md` |
 
Exception handling patterns, try-catch-finally blocks, custom exception types, and defensive programming techniques.
 
[Back to Top](#table-of-contents)
 
### 1.2 Language Fundamentals
 
| Document | Path |
|----------|------|
| [Language Fundamentals](./C%23_Basics/mds/02-language-fundamentals.md) | `C#_Basics/mds/02-language-fundamentals.md` |
 
Core syntax, operators, control flow structures, and statement-level constructs.
 
[Back to Top](#table-of-contents)
 
### 1.3 Type System
 
| Document | Path |
|----------|------|
| [Type System](./C%23_Basics/mds/03-type-system.md) | `C#_Basics/mds/03-type-system.md` |
 
Value types vs reference types, boxing and unboxing operations, nullable value types, type inference, and the unified type system.
 
[Back to Top](#table-of-contents)
 
### 1.4 Strings
 
| Document | Path |
|----------|------|
| [Strings](./C%23_Basics/mds/04-strings.md) | `C#_Basics/mds/04-strings.md` |
 
String manipulation, StringBuilder for performance, string interpolation, formatting options, and encoding considerations.
 
[Back to Top](#table-of-contents)
 
### 1.5 DateTime
 
| Document | Path |
|----------|------|
| [DateTime](./C%23_Basics/mds/05-datetime.md) | `C#_Basics/mds/05-datetime.md` |
 
Date and time handling, TimeZoneInfo, DateTimeOffset for timezone-aware operations, parsing, and formatting strategies.
 
[Back to Top](#table-of-contents)
 
### 1.6 Memory Management
 
| Document | Path |
|----------|------|
| [Memory Management](./C%23_Basics/mds/06-memory.md) | `C#_Basics/mds/06-memory.md` |
 
Stack vs heap allocation, garbage collection fundamentals, object lifetime management, and memory leak detection.
 
[Back to Top](#table-of-contents)
 
### 1.7 Best Practices
 
| Document | Path |
|----------|------|
| [Best Practices](./C%23_Basics/mds/07-best-practices.md) | `C#_Basics/mds/07-best-practices.md` |
 
Coding standards, naming conventions, maintainable code patterns, and professional development guidelines.
 
[Back to Top](#table-of-contents)
 
---
 
## 2. Object-Oriented Programming
 
The four pillars of OOP and advanced object-oriented design concepts.
 
### 2.1 Encapsulation
 
| Document | Path |
|----------|------|
| [Encapsulation](./OOP/mds/01-encapsulation.md) | `OOP/mds/01-encapsulation.md` |
 
Access modifiers, properties with getters and setters, data hiding principles, immutability patterns, and information hiding.
 
[Back to Top](#table-of-contents)
 
### 2.2 Abstraction
 
| Document | Path |
|----------|------|
| [Abstraction](./OOP/mds/02-abstraction.md) | `OOP/mds/02-abstraction.md` |
 
Abstract classes, interfaces, hiding implementation complexity, and defining contracts for components.
 
[Back to Top](#table-of-contents)
 
### 2.3 Inheritance
 
| Document | Path |
|----------|------|
| [Inheritance](./OOP/mds/03-inheritance.md) | `OOP/mds/03-inheritance.md` |
 
Base and derived classes, method overriding, inheritance hierarchies, and the Liskov Substitution Principle.
 
[Back to Top](#table-of-contents)
 
### 2.4 Polymorphism
 
| Document | Path |
|----------|------|
| [Polymorphism](./OOP/mds/04-polymorphism.md) | `OOP/mds/04-polymorphism.md` |
 
Method overloading, method overriding, runtime polymorphism (virtual/override), compile-time polymorphism, and dynamic dispatch.
 
[Back to Top](#table-of-contents)
 
### 2.5 Class Relationships
 
| Document | Path |
|----------|------|
| [Class Relationships](./OOP/mds/05-class-relationships.md) | `OOP/mds/05-class-relationships.md` |
 
Association, aggregation, composition, dependency injection, and understanding coupling vs cohesion.
 
[Back to Top](#table-of-contents)
 
### 2.6 Interview Questions
 
| Document | Path |
|----------|------|
| [Interview Questions](./OOP/mds/06-interview-questions.md) | `OOP/mds/06-interview-questions.md` |
 
Common OOP interview questions and answers with detailed explanations and code examples.
 
[Back to Top](#table-of-contents)
 
---
 
## 3. Advanced C# & Concurrency
 
High-performance programming, asynchronous patterns, threading, and resource management.
 
### 3.1 Language Features
 
#### Advanced Generics
 
| Document | Path |
|----------|------|
| [Advanced Generics](./Advanced_C%23/mds/01-advanced-generics.md) | `Advanced_C#/mds/01-advanced-generics.md` |
 
Generic constraints (where clauses), covariance and contravariance, generic methods, performance considerations, and type parameter patterns.
 
[Back to Top](#table-of-contents)
 
#### Events & Observer Pattern
 
| Document | Path |
|----------|------|
| [Events & Observer Pattern](./Advanced_C%23/mds/02-events-observer-pattern.md) | `Advanced_C#/mds/02-events-observer-pattern.md` |
 
Event handling mechanisms, the Observer design pattern, event-driven architecture, and weak event patterns.
 
[Back to Top](#table-of-contents)
 
#### Delegates
 
| Document | Path |
|----------|------|
| [Delegates](./Advanced_C%23/mds/17-delegates.md) | `Advanced_C#/mds/17-delegates.md` |
 
Func, Action, and Predicate delegates, anonymous methods, lambda expressions, multicast delegates, and closure captures.
 
[Back to Top](#table-of-contents)
 
### 3.2 Asynchronous Programming
 
#### Threading & Concurrency
 
| Document | Path |
|----------|------|
| [Threading & Concurrency](./Advanced_C%23/mds/03-threading-concurrency-async.md) | `Advanced_C#/mds/03-threading-concurrency-async.md` |
 
Thread lifecycle, Task-based Asynchronous Pattern (TAP), synchronization contexts, and async/await fundamentals.
 
[Back to Top](#table-of-contents)
 
#### IAsyncEnumerable
 
| Document | Path |
|----------|------|
| [IAsyncEnumerable](./Advanced_C%23/mds/07_IAsyncEnumerable.md) | `Advanced_C#/mds/07_IAsyncEnumerable.md` |
 
Async streams, yield return in async contexts, consuming async enumerables, and memory-efficient streaming.
 
[Back to Top](#table-of-contents)
 
#### Async/Await Deep Dive
 
| Document | Path |
|----------|------|
| [Async/Await Deep Dive](./Advanced_C%23/mds/21-async-await-deep-dive.md) | `Advanced_C#/mds/21-async-await-deep-dive.md` |
 
Compiler-generated state machines, ConfigureAwait behavior, async best practices, exception handling in async code, and value tasks.
 
[Back to Top](#table-of-contents)
 
#### Parallel.ForEachAsync
 
| Document | Path |
|----------|------|
| [Parallel.ForEachAsync](./Advanced_C%23/mds/06_Parallel_ForEachAsync.md) | `Advanced_C#/mds/06_Parallel_ForEachAsync.md` |
 
Parallel async processing, degree of parallelism control, throttling strategies, and batch processing patterns.
 
[Back to Top](#table-of-contents)
 
#### File I/O Async
 
| Document | Path |
|----------|------|
| [File I/O Async](./Advanced_C%23/mds/10_File_IO_Async.md) | `Advanced_C#/mds/10_File_IO_Async.md` |
 
Asynchronous file operations, buffering strategies, FileStream options, and I/O completion ports.
 
[Back to Top](#table-of-contents)
 
#### Channels
 
| Document | Path |
|----------|------|
| [Channels](./Advanced_C%23/mds/12_Channels.md) | `Advanced_C#/mds/12_Channels.md` |
 
Producer-consumer patterns, bounded and unbounded channels, ChannelReader/ChannelWriter, and backpressure handling.
 
[Back to Top](#table-of-contents)
 
### 3.3 Threading & Synchronization
 
#### Thread.Join
 
| Document | Path |
|----------|------|
| [Thread.Join](./Advanced_C%23/mds/05_Thread_Join.md) | `Advanced_C#/mds/05_Thread_Join.md` |
 
Thread coordination mechanisms, waiting for thread completion, timeout handling, and thread synchronization primitives.
 
[Back to Top](#table-of-contents)
 
#### Monitor, Mutex, Semaphore
 
| Document | Path |
|----------|------|
| [Monitor, Mutex, Semaphore](./Advanced_C%23/mds/08_Monitor_Mutex_Semaphore.md) | `Advanced_C#/mds/08_Monitor_Mutex_Semaphore.md` |
 
Lock statements (Monitor.Enter/Exit), Mutex for cross-process synchronization, Semaphore and SemaphoreSlim for resource limiting.
 
[Back to Top](#table-of-contents)
 
#### Interlocked
 
| Document | Path |
|----------|------|
| [Interlocked](./Advanced_C%23/mds/09_Interlocked.md) | `Advanced_C#/mds/09_Interlocked.md` |
 
Atomic operations, lock-free programming, CompareExchange patterns, and memory barrier semantics.
 
[Back to Top](#table-of-contents)
 
#### Thread Pool Starvation
 
| Document | Path |
|----------|------|
| [Thread Pool Starvation](./Advanced_C%23/mds/14_Thread_Pool_Starvation.md) | `Advanced_C#/mds/14_Thread_Pool_Starvation.md` |
 
Detection of thread pool exhaustion, prevention strategies, min/max thread configuration, and avoiding sync-over-async.
 
[Back to Top](#table-of-contents)
 
### 3.4 Memory & Performance
 
#### Performance & Memory
 
| Document | Path |
|----------|------|
| [Performance & Memory](./Advanced_C%23/mds/04-performance-memory.md) | `Advanced_C#/mds/04-performance-memory.md` |
 
Profiling techniques, memory diagnostics, performance counters, BenchmarkDotNet usage, and optimization strategies.
 
[Back to Top](#table-of-contents)
 
#### Queue, Stack, Span
 
| Document | Path |
|----------|------|
| [Queue, Stack, Span](./Advanced_C%23/mds/19-queue-stack-span.md) | `Advanced_C#/mds/19-queue-stack-span.md` |
 
Span\<T\> and Memory\<T\> for stack-safe buffers, stackalloc, ArrayPool\<T\>, and low-allocation patterns.
 
[Back to Top](#table-of-contents)
 
#### Concurrent Collections
 
| Document | Path |
|----------|------|
| [Concurrent Collections](./Advanced_C%23/mds/13_Concurrent_Collections.md) | `Advanced_C#/mds/13_Concurrent_Collections.md` |
 
ConcurrentDictionary, ConcurrentQueue, ConcurrentBag, BlockingCollection, and thread-safe collection patterns.
 
[Back to Top](#table-of-contents)
 
#### IDisposable & Using
 
| Document | Path |
|----------|------|
| [IDisposable & Using](./Advanced_C%23/mds/11_IDisposable_Using.md) | `Advanced_C#/mds/11_IDisposable_Using.md` |
 
Resource cleanup patterns, deterministic disposal, finalizers, SafeHandle, and the Dispose pattern implementation.
 
[Back to Top](#table-of-contents)
 
#### List Deep Dive
 
| Document | Path |
|----------|------|
| [List Deep Dive](./Advanced_C%23/mds/18-list-deep-dive.md) | `Advanced_C#/mds/18-list-deep-dive.md` |
 
List\<T\> internals, capacity management, growth algorithms, trimming excess, and performance characteristics.
 
[Back to Top](#table-of-contents)
 
#### SortedList, Dictionary, HashSet
 
| Document | Path |
|----------|------|
| [SortedList, Dictionary, HashSet](./Advanced_C%23/mds/20-sortedlist-dictionary-hashset.md) | `Advanced_C#/mds/20-sortedlist-dictionary-hashset.md` |
 
Hash table implementations, Dictionary internals, hash code algorithms, collision resolution, and custom equality comparers.
 
[Back to Top](#table-of-contents)
 
---
 
## 4. LINQ
 
Language Integrated Query and data transformation strategies.
 
| Document | Path |
|----------|------|
| [LINQ Deep Dive](./LINQ/mds/linq-deep-dive.md) | `LINQ/mds/linq-deep-dive.md` |
 
Query syntax vs method syntax, deferred vs immediate execution, standard query operators, LINQ providers (LINQ to Objects, LINQ to SQL), and expression trees.
 
[Back to Top](#table-of-contents)
 
---
 
## 5. Database Architecture
 
SQL optimization, design patterns, and reliability strategies for relational databases.
 
### 5.1 Design & Structure
 
#### Database Design & Normalization
 
| Document | Path |
|----------|------|
| [Database Design & Normalization](./database/mds/01-database-design-normalization.md) | `database/mds/01-database-design-normalization.md` |
 
First through Fifth Normal Forms (1NF-5NF), denormalization strategies, schema design principles, and entity-relationship modeling.
 
[Back to Top](#table-of-contents)
 
#### Data Validation & Constraints
 
| Document | Path |
|----------|------|
| [Data Validation & Constraints](./database/mds/03-data-validation-constraints.md) | `database/mds/03-data-validation-constraints.md` |
 
CHECK constraints, UNIQUE constraints, FOREIGN KEY relationships, DEFAULT values, triggers for validation, and domain integrity.
 
[Back to Top](#table-of-contents)
 
#### Pagination Strategies
 
| Document | Path |
|----------|------|
| [Pagination Strategies](./database/mds/04-pagination-strategies.md) | `database/mds/04-pagination-strategies.md` |
 
OFFSET/FETCH pagination, keyset pagination (seek method), cursor-based pagination, performance comparison, and implementation patterns.
 
[Back to Top](#table-of-contents)
 
### 5.2 Performance & Optimization
 
#### Indexing Strategies
 
| Document | Path |
|----------|------|
| [Indexing Strategies](./database/mds/05-indexing-strategies.md) | `database/mds/05-indexing-strategies.md` |
 
B-tree index structure, clustered vs non-clustered indexes, composite indexes, index selectivity, and index maintenance.
 
[Back to Top](#table-of-contents)
 
#### Advanced Indexes
 
| Document | Path |
|----------|------|
| [Advanced Indexes](./database/mds/09-advanced-indexes.md) | `database/mds/09-advanced-indexes.md` |
 
Covering indexes, filtered indexes (conditional indexes), indexed views, full-text search indexes, and columnstore indexes.
 
[Back to Top](#table-of-contents)
 
#### Query Performance
 
| Document | Path |
|----------|------|
| [Query Performance](./database/mds/06-query-performance.md) | `database/mds/06-query-performance.md` |
 
Query tuning fundamentals, SARGable queries, parameter sniffing, statistics maintenance, and cardinality estimation.
 
[Back to Top](#table-of-contents)
 
#### Query Optimization
 
| Document | Path |
|----------|------|
| [Query Optimization](./database/mds/13-query-optimization.md) | `database/mds/13-query-optimization.md` |
 
Query rewrite strategies, plan guides, query hints, optimization techniques for complex queries, and recursive CTEs.
 
[Back to Top](#table-of-contents)
 
#### Execution Plans
 
| Document | Path |
|----------|------|
| [Execution Plans](./database/mds/10-execution-plans.md) | `database/mds/10-execution-plans.md` |
 
Reading graphical and XML execution plans, plan operators, cost estimation, index seeks vs scans, and plan cache analysis.
 
[Back to Top](#table-of-contents)
 
### 5.3 Concurrency & Reliability
 
#### Locking & Concurrency
 
| Document | Path |
|----------|------|
| [Locking & Concurrency](./database/mds/08-locking-concurrency.md) | `database/mds/08-locking-concurrency.md` |
 
Transaction isolation levels (Read Uncommitted to Serializable), locking hints, deadlock detection and resolution, row versioning (SNAPSHOT isolation).
 
[Back to Top](#table-of-contents)
 
#### Backup & Recovery
 
| Document | Path |
|----------|------|
| [Backup & Recovery](./database/mds/02-backup-recovery.md) | `database/mds/02-backup-recovery.md` |
 
Full backups, differential backups, transaction log backups, point-in-time recovery strategies, and disaster recovery planning.
 
[Back to Top](#table-of-contents)
 
### 5.4 Advanced Topics
 
#### SQL Functions
 
| Document | Path |
|----------|------|
| [SQL Functions](./database/mds/07-sql-functions.md) | `database/mds/07-sql-functions.md` |
 
Scalar functions, table-valued functions (inline and multi-statement), window functions (ROW_NUMBER, RANK, LEAD, LAG), and aggregate functions.
 
[Back to Top](#table-of-contents)
 
#### Security & Vulnerabilities
 
| Document | Path |
|----------|------|
| [Security & Vulnerabilities](./database/mds/11-security-vulnerabilities.md) | `database/mds/11-security-vulnerabilities.md` |
 
SQL injection prevention, parameterized queries, encryption (TDE, column-level), row-level security, and least privilege principles.
 
[Back to Top](#table-of-contents)
 
#### NoSQL Fundamentals
 
| Document | Path |
|----------|------|
| [NoSQL Fundamentals](./database/mds/12-nosql-fundamentals.md) | `database/mds/12-nosql-fundamentals.md` |
 
Document databases (MongoDB), key-value stores (Redis), column-family stores (Cassandra), graph databases (Neo4j), and CAP theorem.
 
[Back to Top](#table-of-contents)
 
---
 
## 6. EF Core
 
Entity Framework Core deep dives covering ORM patterns, expression trees, migrations, DbContext internals, and performance optimization.
 
### 6.1 Micro ORM vs Full ORM
 
| Document | Path |
|----------|------|
| [Micro ORM vs Full ORM](./EFCore/mds/01-micro-orm-vs-full-orm.md) | `EFCore/mds/01-micro-orm-vs-full-orm.md` |
 
Comprehensive comparison between Micro ORMs (Dapper) and Full ORMs (EF Core). Architecture differences, performance characteristics, decision matrix, and hybrid approach recommendations with real-world e-commerce examples.
 
[Back to Top](#table-of-contents)
 
### 6.2 Expression Trees
 
| Document | Path |
|----------|------|
| [Expression Trees](./EFCore/mds/02-expression-trees.md) | `EFCore/mds/02-expression-trees.md` |
 
Expression tree structure and traversal, LINQ-to-SQL translation pipeline, dynamic query builder implementation, specification pattern, and client vs server evaluation pitfalls.
 
[Back to Top](#table-of-contents)
 
### 6.3 Migrations vs Scaffolding
 
| Document | Path |
|----------|------|
| [Migrations vs Scaffolding](./EFCore/mds/03-migrations-vs-scaffolding.md) | `EFCore/mds/03-migrations-vs-scaffolding.md` |
 
Code-first migrations vs database-first scaffolding workflows, migration file structure, team development strategies, merge conflict resolution, and decision framework for approach selection.
 
[Back to Top](#table-of-contents)
 
### 6.4 DbContext Deep Dive
 
| Document | Path |
|----------|------|
| [DbContext Deep Dive](./EFCore/mds/04-dbcontext-deep-dive.md) | `EFCore/mds/04-dbcontext-deep-dive.md` |
 
DbContext lifecycle management, configuration architecture, change tracking mechanisms, multi-tenant SaaS implementation, performance optimization with context pooling and compiled queries.
 
[Back to Top](#table-of-contents)
 
### 6.5 Dependency Injection
 
| Document | Path |
|----------|------|
| [Dependency Injection](./EFCore/mds/05-dependency-injection.md) | `EFCore/mds/05-dependency-injection.md` |
 
DI registration patterns, repository and Unit of Work patterns, enterprise application architecture, multi-context registration, factory pattern for background services, and integration testing setup.
 
[Back to Top](#table-of-contents)
 
### 6.6 Connection Pool Exhaustion
 
| Document | Path |
|----------|------|
| [Connection Pool Exhaustion](./EFCore/mds/06-connection-pooling.md) | `EFCore/mds/06-connection-pooling.md` |
 
Connection pool mechanics, exhaustion causes and detection, long-running query analysis, mitigation strategies, circuit breaker pattern, and monitoring with health checks and metrics collection.
 
[Back to Top](#table-of-contents)
 
### 6.7 Migrations Lab Assessment
 
| Document | Path |
|----------|------|
| [Migrations Lab Assessment](./EFCore/mds/efcore-migrations-lab-assessment.md) | `EFCore/mds/efcore-migrations-lab-assessment.md` |
 
Comprehensive hands-on assessment covering real-world migration scenarios, team collaboration edge cases, database lifecycle management, accident recovery, Git conflict resolution, and SQL script generation for production deployments.
 
[Back to Top](#table-of-contents)
 
### 6.8 DbSet Concurrency and Relationships
 
| Document | Path |
|----------|------|
| [DbSet Concurrency and Relationships](./EFCore/mds/07_Dbset_Concurrency.md) | `EFCore/mds/07_Dbset_Concurrency.md` |
 
DbSet\<TEntity\> generic patterns, optimistic concurrency with RowVersion/Timestamp tokens, handling DbUpdateConcurrencyException, and InverseProperty attribute for multiple navigation relationships.
 
[Back to Top](#table-of-contents)
 
### 6.9 Advanced Mapping
 
| Document | Path |
|----------|------|
| [Advanced Mapping](./EFCore/mds/08_Advanced_Mapping.md) | `EFCore/mds/08_Advanced_Mapping.md` |
 
ValueConverter for custom type transformations, temporal tables (IsTemporal) for automatic history tracking, and global query filters (HasQueryFilter) for soft-delete and multi-tenancy patterns.
 
[Back to Top](#table-of-contents)
 
### 6.10 Query Performance
 
| Document | Path |
|----------|------|
| [Query Performance](./EFCore/mds/09_Query_Performance.md) | `EFCore/mds/09_Query_Performance.md` |
 
AsNoTracking vs AsNoTrackingWithIdentityResolution, SingleQuery vs SplitQuery for collection includes, bulk insert batching with SQL parameterization, and compiled queries for hot path optimization.
 
[Back to Top](#table-of-contents)
 
### 6.11 Query Translation and Debugging
 
| Document | Path |
|----------|------|
| [Query Translation and Debugging](./EFCore/mds/10_Query_Translation.md) | `EFCore/mds/10_Query_Translation.md` |
 
Client vs server evaluation boundaries, untranslatable LINQ patterns, ToQueryString for SQL preview, TagWith for query correlation, EnableDetailedErrors for exception diagnostics, and logging configuration with LogLevel filtering.
 
[Back to Top](#table-of-contents)
 
### 6.12 LINQ Expressions and Functions
 
| Document | Path |
|----------|------|
| [LINQ Expressions and Functions](./EFCore/mds/11_LINQ_Expressions.md) | `EFCore/mds/11_LINQ_Expressions.md` |
 
Expression trees structure and dynamic building, EF.Functions (Like, DateDiff) for database-specific operations, and Regular Expressions limitations (not translatable to SQL) with workaround patterns.
 
[Back to Top](#table-of-contents)
 
### 6.13 Interceptors and Raw SQL
 
| Document | Path |
|----------|------|
| [Interceptors and Raw SQL](./EFCore/mds/12_Interceptors_RawSQL.md) | `EFCore/mds/12_Interceptors_RawSQL.md` |
 
SaveChangesInterceptor for audit columns, connection resiliency with retry policies, raw SQL execution with FromSqlRaw/FromSqlInterpolated, migrations best practices, and IEntityTypeConfiguration patterns.
 
[Back to Top](#table-of-contents)
 
### 6.14 DotNET Ecosystem Tools
 
| Document | Path |
|----------|------|
| [DotNET Ecosystem Tools](./EFCore/mds/13_DotNet_Ecosystem.md) | `EFCore/mds/13_DotNet_Ecosystem.md` |
 
Central Package Management (CPM) for version consistency across solutions, and Bogus library for realistic test data generation with datasets for commerce, finance, and personal data.
 
[Back to Top](#table-of-contents)
 
---
 
## 7. Quality Assurance
 
Modern testing methodologies for enterprise applications.
 
### 7.1 Unit Testing
 
| Document | Path |
|----------|------|
| [Unit Testing Fundamentals](./Advanced_C%23/mds/15_Unit_Testing.md) | `Advanced_C#/mds/15_Unit_Testing.md` |
 
xUnit, NUnit, and MSTest frameworks, test organization, Arrange-Act-Assert pattern, data-driven tests, and code coverage analysis.
 
[Back to Top](#table-of-contents)
 
### 7.2 Mocking
 
| Document | Path |
|----------|------|
| [Mocking Techniques](./Advanced_C%23/mds/16_Mocking.md) | `Advanced_C#/mds/16_Mocking.md` |
 
Moq and NSubstitute frameworks, dependency injection for testability, test doubles (stubs, mocks, fakes), and mocking best practices.
 
[Back to Top](#table-of-contents)
 
---
 
## 8. Project Overview
 
### Learning Paths
 
**Beginner Path:**
C# Fundamentals → OOP Pillars → LINQ → Database Design & Normalization → EF Core Migrations
 
**Intermediate Path:**
Advanced Generics → Delegates → Async/Await Deep Dive → Concurrent Collections → EF Core DbContext → Indexing Strategies
 
**Expert Path:**
Thread Pool Starvation → Interlocked → Span/Memory → Execution Plans → EF Core Connection Pooling → Query Optimization
 
**Database Specialist Path:**
Normalization → Indexing Strategies → Query Optimization → Locking & Concurrency → Backup & Recovery → EF Core Performance
 
**Full Stack .NET Path:**
C# Fundamentals → OOP → Database Design → EF Core (Migrations, DbContext, DI) → Async/Await → Testing
 
### Repository Statistics
 
- Total Documents: 57+ technical deep-dives
- Primary Categories: 7 major domains
- C# & .NET Topics: 30 documents
- Database Topics: 13 documents
- EF Core Topics: 14 documents
- Testing Topics: 2 documents
### Project Objectives
 
**Under-the-Hood Understanding:** Moving beyond syntax to understand CLR behavior, memory allocation patterns, and runtime internals.
 
**Concurrency Mastery:** Navigating complex threading scenarios, avoiding deadlocks and race conditions, and building responsive applications.
 
**Database Excellence:** Bridging the gap between application code and data storage through optimized SQL, efficient LINQ providers, and robust schema design.
 
**Production-Ready Testing:** Building reliable, maintainable test suites that enable confident refactoring and continuous delivery.
 
### Author
 
**Ayman Mohamed** — .NET Full Stack Engineer  
Contributor: Built with professional guidance from Senior Engineer Abdelrahman
 
---
[Back to Top](#table-of-contents)
