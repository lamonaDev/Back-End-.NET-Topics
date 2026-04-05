# NoSQL Database Fundamentals

## Table of Contents
1. [What is NoSQL?](#what-is-nosql)
2. [Types of NoSQL Databases](#types-of-nosql-databases)
3. [SQL vs NoSQL](#sql-vs-nosql-comparison)
4. [Data Modeling in NoSQL](#data-modeling-in-nosql)

---

## What is NoSQL

### Definition
NoSQL (Not Only SQL) refers to database systems that provide mechanisms for storage and retrieval of data that differ from the traditional tabular relations used in relational databases.

### Key Characteristics
| Characteristic | Description |
|----------------|-------------|
| **Schema Flexibility** | Dynamic schemas, not predefined |
| **Horizontal Scaling** | Scale out across multiple servers |
| **High Availability** | Built-in replication and failover |
| **Performance** | Optimized for specific access patterns |
| **Polyglot Persistence** | Use multiple databases for different needs |

### Problems NoSQL Solves
1. **Massive scale**: Billions of rows, petabytes of data
2. **High throughput**: Millions of operations per second
3. **Flexible data**: Varied and evolving data structures
4. **Geographic distribution**: Global data replication
5. **Rapid development**: Agile schema evolution

### CAP Theorem
In distributed systems, you can only guarantee **two** of:
- **C**onsistency: All nodes see same data
- **A**vailability: Every request receives response
- **P**artition tolerance: System works despite network failures

| Database Type | CAP Focus |
|---------------|-----------|
| Relational | CP (Consistency + Partition) |
| MongoDB | CP with tunable consistency |
| Cassandra | AP (Availability + Partition) |
| Redis | CP |

---

## Types of NoSQL Databases

### 1. Document Stores
**Examples**: MongoDB, Couchbase, Amazon DocumentDB

**Characteristics**:
- Store data as documents (JSON, BSON)
- Self-contained records with nested structures
- Flexible schema within collections

**Structure**:
```json
{
    "_id": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "addresses": [
        {
            "type": "home",
            "city": "New York",
            "zip": "10001"
        }
    ],
    "preferences": {
        "newsletter": true,
        "theme": "dark"
    }
}
```

**Best For**:
- Content management
- User profiles
- Catalogs
- Mobile applications

**When to Use**:
- Data has hierarchical structure
- Schema evolves frequently
- Document represents complete entity

### 2. Key-Value Stores
**Examples**: Redis, Amazon DynamoDB, Riak

**Characteristics**:
- Simplest NoSQL type
- Key maps to arbitrary value
- Extremely fast lookups
- Schema-less value storage

**Structure**:
```
Key: user:123:profile
Value: {"name":"John","age":30}

Key: session:abc123
Value: {"userId":"123","expires":"2024-06-01"}
```

**Best For**:
- Caching
- Session management
- Shopping carts
- Real-time leaderboards

**When to Use**:
- Simple data access patterns
- Need extremely low latency
- Data fits simple key-value model

### 3. Wide-Column Stores (Column-Family)
**Examples**: Cassandra, HBase, Google Bigtable

**Characteristics**:
- Store data in columns, not rows
- Columns grouped into families
- Excellent for write-heavy workloads
- Distributed by design

**Structure**:
```
Row Key: user123
└── Column Family: profile
    ├── name: "John Doe"
    ├── age: 30
    └── email: "john@example.com"
└── Column Family: activity
    ├── last_login: "2024-06-15"
    └── login_count: 150
```

**Best For**:
- Time-series data
- Write-heavy applications
- Event logging
- Messaging systems

**When to Use**:
- Massive write throughput
- Distributed across data centers
- Need linear scalability

### 4. Graph Databases
**Examples**: Neo4j, Amazon Neptune, ArangoDB

**Characteristics**:
- Store data as nodes and relationships
- Relationships are first-class citizens
- Optimized for traversing connections

**Structure**:
```
(:Person {name: "John"})-[:FRIENDS_WITH {since: "2020"}]->(:Person {name: "Jane"})
(:Person {name: "John"})-[:WORKS_AT]->(:Company {name: "TechCorp"})
```

**Best For**:
- Social networks
- Recommendation engines
- Fraud detection
- Network topology

**When to Use**:
- Data is highly interconnected
- Relationships matter more than entities
- Need fast traversal queries

---

## SQL vs NoSQL Comparison

### Data Model

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Structure** | Tables with rows/columns | Documents, key-value, columns, graphs |
| **Schema** | Rigid, predefined | Flexible, dynamic |
| **Relationships** | Foreign keys, JOINs | Embedded or application-level |
| **Normalization** | Encouraged | Denormalization common |

### Query Language

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Standard** | ANSI SQL (universal) | Database-specific |
| **Complex queries** | JOINs, subqueries, aggregations | Simpler, often limited |
| **Ad-hoc queries** | Excellent | Varies by type |
| **Aggregation** | SQL GROUP BY | MapReduce or aggregation pipelines |

### Scalability

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Vertical scaling** | Primary method | Supported |
| **Horizontal scaling** | Complex (sharding, replication) | Built-in, easier |
| **Sharding** | Manual | Automatic |
| **Replication** | Master-slave, synchronous | Master-master, asynchronous |

### ACID Properties

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Atomicity** | Yes | Varies (eventual consistency) |
| **Consistency** | Strong | Tunable (BASE model) |
| **Isolation** | Multiple levels | Limited |
| **Durability** | Yes | Varies |

### Use Case Comparison

| Use Case | Best Choice |
|----------|-------------|
| Banking transactions | SQL (ACID required) |
| Real-time analytics | NoSQL (Column store) |
| Content management | NoSQL (Document store) |
| User sessions | NoSQL (Key-value) |
| Social graph | NoSQL (Graph database) |
| Inventory management | SQL (consistency critical) |
| IoT sensor data | NoSQL (Time-series) |

---

## Data Modeling in NoSQL

### Query-Driven Design
Unlike SQL's "normalize everything" approach, NoSQL modeling is driven by how you'll query the data.

**SQL Thinking**:
```sql
-- Separate tables, join at query time
Users (user_id, name, email)
Orders (order_id, user_id, date, total)
OrderItems (item_id, order_id, product_id, qty)
```

**NoSQL Thinking**:
```json
// Embed related data for single-document access
{
    "_id": "user123",
    "name": "John",
    "email": "john@example.com",
    "orders": [
        {
            "order_id": "ord456",
            "date": "2024-06-15",
            "total": 150.00,
            "items": [
                {"product": "Widget", "qty": 2, "price": 50.00},
                {"product": "Gadget", "qty": 1, "price": 50.00}
            ]
        }
    ]
}
```

### Embedding vs Referencing

#### Embedding (Denormalization)
Store related data together in one document.

**Pros**:
- Single read retrieves all data
- No joins needed
- Atomic updates to related data

**Cons**:
- Data duplication
- Larger documents
- Harder to update shared data

**When to Embed**:
- "Contains" relationship (order contains items)
- Data accessed together
- One-to-few relationships

#### Referencing (Normalization)
Store related data separately, link by ID.

**Pros**:
- No data duplication
- Smaller documents
- Independent updates

**Cons**:
- Multiple queries needed
- Application-level joins
- Inconsistency possible

**When to Reference**:
- Many-to-many relationships
- Frequently changing shared data
- Unbounded one-to-many (unlimited items)

### One-to-Many Modeling

#### Bounded (Few) - Embed
```json
{
    "_id": "user123",
    "addresses": [
        {"type": "home", "city": "NYC"},
        {"type": "work", "city": "LA"}
    ]
}
// Users rarely have more than a few addresses
```

#### Unbounded (Many) - Reference
```json
// User document
{
    "_id": "user123",
    "name": "John"
}

// Separate posts collection
{
    "_id": "post789",
    "user_id": "user123",
    "content": "Hello world"
}
// Users can have unlimited posts
```

### Many-to-Many Without Joins

**SQL Approach**:
```sql
-- Junction table
StudentCourses (student_id, course_id)
```

**NoSQL Approaches**:

1. **Array of References**:
```json
// Student document
{
    "_id": "student123",
    "name": "John",
    "courses": ["courseA", "courseB", "courseC"]
}

// Application fetches each course
```

2. **Two-way References**:
```json
// Student
{"_id": "s123", "courses": ["c1", "c2"]}

// Course
{"_id": "c1", "students": ["s123", "s456"]}
```

3. **Embedding (if bounded)**:
```json
{
    "_id": "student123",
    "enrolled_courses": [
        {"course_id": "c1", "name": "Math", "grade": "A"},
        {"course_id": "c2", "name": "Science", "grade": "B"}
    ]
}
```

### Indexing in NoSQL

| Database | Indexing Strategy |
|----------|-------------------|
| **MongoDB** | Single field, compound, text, geospatial |
| **Cassandra** | Primary key, clustering columns |
| **Redis** | Key-based only (no secondary indexes) |
| **Neo4j** | Indexes on labels and properties |

### Best Practices Summary

1. **Design for queries**: Model data based on how you'll read it
2. **Embed when possible**: Reduce round trips
3. **Reference unbounded relationships**: Avoid unbounded document growth
4. **Accept duplication**: Trade storage for performance
5. **Plan for denormalization**: Update strategies for duplicated data
6. **Understand consistency**: Choose based on your requirements

---

*Source: NoSQL database documentation, distributed systems theory, and modern data architecture practices.*
