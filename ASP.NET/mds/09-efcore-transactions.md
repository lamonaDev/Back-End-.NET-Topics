# Transactions in EF Core

## Overview

Transactions are fundamental to database operations, ensuring data integrity when multiple changes must succeed or fail together. Understanding when and how to use transactions in EF Core is essential for building reliable applications.

## What is a Transaction?

A **transaction** groups multiple database operations into a single atomic unit — either *all* operations succeed (COMMIT) or *none* take effect (ROLLBACK). Transactions enforce the ACID properties:

| Property | Description |
|----------|-------------|
| **Atomicity** | All operations complete or none do — no partial state |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere with each other |
| **Durability** | Committed data is permanently saved |

## ACID Properties Visualized

```
Before Transaction:           After Successful Transaction:
┌─────────────────┐           ┌─────────────────┐
│ Account A: $100 │           │ Account A: $50  │
│ Account B: $0   │ ────────→ │ Account B: $50  │
│ Total: $100     │           │ Total: $100     │
└─────────────────┘           └─────────────────┘

If Something Fails (e.g., Account B update):
┌─────────────────┐           ┌─────────────────┐
│ Account A: $100 │           │ Account A: $100 │ ← Unchanged
│ Account B: $0   │ ────────→ │ Account B: $0   │ ← Unchanged
└─────────────────┘           └─────────────────┘
```

## Default Behavior: SaveChanges

By default, `SaveChanges()` wraps *all pending changes* in a single transaction automatically. If any SQL statement fails, the entire batch is rolled back.

```csharp
// This is implicitly transactional — all or nothing
context.Products.Add(new Product { Name = "Laptop", Price = 999 });
context.Products.Add(new Product { Name = "Mouse", Price = 29 });
await context.SaveChangesAsync(); // one transaction, two INSERTs
```

### When Implicit Transactions Work Well

- Inserting multiple related entities
- Updating parent and child records
- Deleting parent with cascade

### When You Need Explicit Transactions

- Multiple SaveChanges calls
- Mixing EF Core with raw SQL
- Long-running business processes
- Distributed transactions

## Explicit Transactions — When You Need Them

Use explicit transactions when you need **multiple `SaveChanges` calls** within one atomic operation, or when mixing EF Core with raw ADO.NET / Dapper SQL.

### Real Scenario: Place Order (Inventory + Order Creation)

When a customer places an order, you must: (1) deduct stock from inventory, (2) create the order record, (3) create order line items. If step 3 fails, you cannot leave inventory already deducted and no order existing. This requires an explicit transaction.

```csharp
public async Task<Order> PlaceOrderAsync(PlaceOrderDto dto)
{
    // Begin explicit transaction
    await using var transaction = await _context.Database.BeginTransactionAsync();

    try
    {
        // Step 1: Validate and deduct stock
        var product = await _context.Products.FindAsync(dto.ProductId)
            ?? throw new NotFoundException("Product not found");

        if (product.Stock < dto.Quantity)
            throw new BusinessException("Insufficient stock");

        product.Stock -= dto.Quantity;
        await _context.SaveChangesAsync(); // Save #1 (inside transaction)

        // Step 2: Create order
        var order = new Order
        {
            CustomerId = dto.CustomerId,
            CreatedAt = DateTime.UtcNow,
            Status = OrderStatus.Pending
        };
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(); // Save #2 (same transaction)

        // Step 3: Create order items
        var item = new OrderItem
        {
            OrderId = order.Id,
            ProductId = product.Id,
            Quantity = dto.Quantity,
            UnitPrice = product.Price
        };
        _context.OrderItems.Add(item);
        await _context.SaveChangesAsync(); // Save #3

        // All steps succeeded — commit the transaction
        await transaction.CommitAsync();
        return order;
    }
    catch
    {
        // Any failure — roll back ALL changes (stock, order, items)
        await transaction.RollbackAsync();
        throw; // re-throw so caller/middleware handles it
    }
}
```

## Transaction Isolation Levels

SQL Server and other databases support different isolation levels that control how transactions interact:

| Isolation Level | Description | Dirty Read | Non-Repeatable Read | Phantom |
|-----------------|-------------|------------|---------------------|---------|
| Read Uncommitted | Reads uncommitted data | Yes | Yes | Yes |
| Read Committed | Default — reads committed data | No | Yes | Yes |
| Repeatable Read | Locks read data | No | No | Yes |
| Serializable | Full isolation | No | No | No |
| Snapshot | Row versioning | No | No | No |

### Setting Isolation Level in EF Core

```csharp
await using var transaction = await _context.Database.BeginTransactionAsync(
    isolationLevel: System.Data.IsolationLevel.Serializable);
```

## Explicit vs SaveChanges Default

| Aspect | Default SaveChanges | Explicit Transaction |
|--------|---------------------|----------------------|
| Scope | Single SaveChanges call | Multiple SaveChanges calls |
| Control | Automatic | Manual BeginTransaction / Commit / Rollback |
| Mixed providers | EF only | Can include raw SQL / Dapper |
| Savepoints | Not available | Supported in SQL Server and PostgreSQL |
| Use when | Single batch of related entities | Multi-step business workflows |

## Transaction with Raw SQL

```csharp
public async Task ImportUsersAsync(List<UserDto> users)
{
    await using var transaction = await _context.Database.BeginTransactionAsync();
    
    try
    {
        // EF Core operation
        await _context.Users.AddAsync(newUser);
        await _context.SaveChangesAsync();
        
        // Raw SQL operation
        await _context.Database.ExecuteSqlRawAsync(
            "UPDATE Stats SET UserCount = UserCount + 1");
        
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

## Savepoints

Savepoints allow you to partially rollback a transaction:

```csharp
await using var transaction = await _context.Database.BeginTransactionAsync();

try
{
    // First operation
    await _context.Orders.AddAsync(order1);
    await _context.SaveChangesAsync();
    await transaction.SaveAsync("order1_saved"); // Savepoint
    
    try
    {
        // Second operation
        await _context.Orders.AddAsync(order2);
        await _context.SaveChangesAsync();
    }
    catch
    {
        // Rollback to savepoint, not the whole transaction
        await transaction.RollbackAsync("order1_saved");
    }
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Performance Considerations

> **Performance Note**: Keep transactions as **short as possible**. Long-running transactions hold database locks, increasing blocking and deadlock risk under concurrent load. Never perform HTTP calls or user I/O inside a transaction.

### Best Practices

1. **Keep transactions short** — Do validation before the transaction
2. **Avoid user interaction** — Don't wait for input inside a transaction
3. **No HTTP calls** — Network calls can fail or timeout
4. **Use appropriate isolation level** — Don't use Serializable unless needed
5. **Index properly** — Locking on unindexed tables is expensive

### Locking Example

```csharp
// Without proper isolation: Possible Race Condition
product.Stock -= dto.Quantity;
// If another request reads Stock between read and write,
// both might see enough stock, both deduct, resulting in negative stock

// With proper locking (SELECT FOR UPDATE):
var product = await _context.Products
    .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK) WHERE Id = {0}", dto.ProductId)
    .FirstOrDefaultAsync();
    
product.Stock -= dto.Quantity;
await _context.SaveChangesAsync();
```

## Distributed Transactions

When working with multiple databases:

```csharp
// Requires MSDTC (Microsoft Distributed Transaction Coordinator)
await using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.Serializable });

try
{
    await _context1.Database.ExecuteSqlRawAsync("INSERT...");
    await _context2.Database.ExecuteSqlRawAsync("INSERT...");
    
    scope.Complete();
}
catch
{
    // Automatic rollback
}
```

Note: Distributed transactions require additional infrastructure (MSDTC) and have performance overhead. Consider using eventual consistency patterns instead when possible.

## Error Handling Patterns

```csharp
public async Task<Order> PlaceOrderAsync(PlaceOrderDto dto)
{
    await using var transaction = await _context.Database.BeginTransactionAsync();
    
    try
    {
        // ... business logic ...
        await transaction.CommitAsync();
        return order;
    }
    catch (DbUpdateException ex)
    {
        await transaction.RollbackAsync();
        
        // Handle specific database errors
        if (ex.InnerException is SqlException sqlEx && sqlEx.Number == 1205)
        {
            // Deadlock victim — retry logic here
            throw new RetryableException("Please retry your order", ex);
        }
        
        throw;
    }
}
```

## References

- [Microsoft Docs — EF Core Transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions)
- [EF Core — Handling Concurrency Conflicts](https://learn.microsoft.com/en-us/ef/core/saving/concurrency)