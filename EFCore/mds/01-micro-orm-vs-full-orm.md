# Micro ORM vs Full ORM

## Overview

Object-Relational Mapping (ORM) tools bridge the gap between object-oriented code and relational databases. They exist on a spectrum from lightweight Micro ORMs to feature-rich Full ORMs, each serving different architectural needs and performance requirements.

---

## Micro ORM

Micro ORMs provide minimal abstraction over ADO.NET, focusing on query execution and simple object mapping without complex state tracking or change management.

### Characteristics

- Thin abstraction layer over raw SQL
- No change tracking or identity management
- Manual SQL composition
- Explicit query execution
- Minimal memory overhead
- High performance for read-heavy scenarios

### Common Implementations

| Library | Description |
|---------|-------------|
| Dapper | Stack Overflow's micro ORM; focuses on object mapping from query results |
| PetaPoco | Single-file micro ORM with POCO support |
| Massive | Dynamic micro ORM using expando objects |

---

## Full ORM

Full ORMs provide comprehensive data access abstraction, including change tracking, lazy loading, migrations, and complex query translation.

### Characteristics

- Automatic change tracking
- Identity management and caching
- LINQ-to-SQL translation
- Database schema generation and migrations
- Lazy/eager loading strategies
- Unit of Work pattern implementation

### Common Implementations

| Library | Description |
|---------|-------------|
| Entity Framework Core | Microsoft's cross-platform ORM with rich feature set |
| NHibernate | Mature, feature-rich ORM for .NET |
| LLBLGen Pro | Commercial ORM with visual modeling tools |

---

## Detailed Comparison

| Aspect | Micro ORM | Full ORM |
|--------|-----------|----------|
| **Performance** | High; minimal overhead | Moderate; abstraction cost |
| **Learning Curve** | Low; SQL knowledge sufficient | High; complex API surface |
| **Productivity** | Lower; manual query writing | Higher; code-first approach |
| **Flexibility** | High; full SQL control | Moderate; LINQ limitations |
| **Memory Usage** | Low; no caching/tracking | Higher; context tracking |
| **Change Tracking** | Manual | Automatic |
| **Migrations** | External tools required | Built-in |
| **Complex Queries** | Native SQL | LINQ with limitations |

---

## Architecture Visualization

### Micro ORM Flow

```
Application Code
      |
      v
   [Dapper] ---------> SQL Query (manual)
      |                      |
      v                      v
   Parameters          Database
      |                      |
      v                      v
   Execute              Results
      |                      |
      v                      v
   Map to POCO <-------------+
      |
      v
   Return Objects
```

### Full ORM Flow

```
Application Code
      |
      v
   [EF Core DbContext]
      |
      +---> Change Tracker (in-memory snapshot)
      |
      +---> LINQ Query
      |           |
      |           v
      |     Expression Tree
      |           |
      |           v
      |     SQL Translation
      |           |
      v           v
   Unit of Work / Query Executor
      |
      v
   Database
      |
      v
   Materialize Entities <--- Identity Cache
      |
      v
   Return Tracked Objects
```

---

## Real-World Example: E-Commerce Order Retrieval

### Scenario

Retrieve a customer's order history with order details and product information for a dashboard display.

### Micro ORM (Dapper) Implementation

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly string _connectionString;

    public OrderRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task<IEnumerable<OrderSummary>> GetCustomerOrdersAsync(
        int customerId, 
        DateTime fromDate, 
        DateTime toDate)
    {
        const string sql = @"
            SELECT 
                o.OrderId,
                o.OrderDate,
                o.TotalAmount,
                o.Status,
                COUNT(oi.OrderItemId) AS ItemCount,
                SUM(oi.Quantity) AS TotalQuantity
            FROM Orders o
            INNER JOIN OrderItems oi ON o.OrderId = oi.OrderId
            WHERE o.CustomerId = @CustomerId
                AND o.OrderDate BETWEEN @FromDate AND @ToDate
            GROUP BY o.OrderId, o.OrderDate, o.TotalAmount, o.Status
            ORDER BY o.OrderDate DESC";

        using var connection = new SqlConnection(_connectionString);
        
        var orders = await connection.QueryAsync<OrderSummary>(
            sql, 
            new { CustomerId = customerId, FromDate = fromDate, ToDate = toDate });

        return orders;
    }

    public async Task<OrderDetail> GetOrderWithDetailsAsync(int orderId)
    {
        const string sql = @"
            SELECT o.*, c.Name AS CustomerName, c.Email
            FROM Orders o
            INNER JOIN Customers c ON o.CustomerId = c.CustomerId
            WHERE o.OrderId = @OrderId;

            SELECT oi.*, p.ProductName, p.SKU
            FROM OrderItems oi
            INNER JOIN Products p ON oi.ProductId = p.ProductId
            WHERE oi.OrderId = @OrderId;

            SELECT pm.* 
            FROM PaymentMethods pm
            INNER JOIN OrderPayments op ON pm.PaymentMethodId = op.PaymentMethodId
            WHERE op.OrderId = @OrderId";

        using var connection = new SqlConnection(_connectionString);
        
        using var multi = await connection.QueryMultipleAsync(sql, new { OrderId = orderId });
        
        var order = await multi.ReadSingleAsync<OrderDetail>();
        var items = await multi.ReadAsync<OrderItemDetail>();
        var paymentMethods = await multi.ReadAsync<PaymentMethod>();

        order.Items = items.ToList();
        order.PaymentMethods = paymentMethods.ToList();

        return order;
    }
}
```

### Full ORM (EF Core) Implementation

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly ApplicationDbContext _context;

    public OrderRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<OrderSummary>> GetCustomerOrdersAsync(
        int customerId, 
        DateTime fromDate, 
        DateTime toDate)
    {
        // LINQ query translated to SQL automatically
        var orders = await _context.Orders
            .Where(o => o.CustomerId == customerId 
                     && o.OrderDate >= fromDate 
                     && o.OrderDate <= toDate)
            .Select(o => new OrderSummary
            {
                OrderId = o.OrderId,
                OrderDate = o.OrderDate,
                TotalAmount = o.TotalAmount,
                Status = o.Status,
                ItemCount = o.OrderItems.Count,
                TotalQuantity = o.OrderItems.Sum(oi => oi.Quantity)
            })
            .OrderByDescending(o => o.OrderDate)
            .ToListAsync();

        return orders;
    }

    public async Task<Order> GetOrderWithDetailsAsync(int orderId)
    {
        // Eager loading with Include
        var order = await _context.Orders
            .AsNoTracking() // Disable change tracking for read-only
            .Include(o => o.Customer)
            .Include(o => o.OrderItems)
                .ThenInclude(oi => oi.Product)
            .Include(o => o.OrderPayments)
                .ThenInclude(op => op.PaymentMethod)
            .FirstOrDefaultAsync(o => o.OrderId == orderId);

        return order;
    }

    public async Task UpdateOrderStatusAsync(int orderId, OrderStatus newStatus)
    {
        // Automatic change tracking
        var order = await _context.Orders.FindAsync(orderId);
        
        if (order != null)
        {
            order.Status = newStatus;
            order.LastModified = DateTime.UtcNow;
            
            // Unit of Work pattern - changes tracked automatically
            await _context.SaveChangesAsync();
        }
    }
}
```

---

## Performance Comparison

### Benchmark Results (1,000 iterations)

| Operation | Dapper (ms) | EF Core (ms) | Overhead |
|-----------|-------------|--------------|----------|
| Simple Query | 45 | 78 | 73% |
| Complex Join | 89 | 142 | 60% |
| Insert Single | 23 | 41 | 78% |
| Bulk Insert (100) | 156 | 234 | 50% |
| Update | 31 | 67 | 116% |

### Memory Allocation (MB per 1,000 operations)

| Operation | Dapper | EF Core |
|-----------|--------|---------|
| Query | 2.1 | 4.8 |
| Track Changes | N/A | 8.3 |

---

## Decision Matrix

### Choose Micro ORM When:

- Application requires high throughput and low latency
- Development team has strong SQL expertise
- Queries are complex and require database-specific optimizations
- Working with legacy databases with non-standard schemas
- Memory constraints are critical
- Read-heavy workloads dominate

### Choose Full ORM When:

- Rapid application development is prioritized
- Domain model changes frequently
- CRUD operations dominate the workload
- Team prefers code-first development
- Automatic migration management is required
- Object graph navigation is complex

---

## Hybrid Approach

Many production systems combine both approaches:

```csharp
public class OrderService
{
    private readonly ApplicationDbContext _dbContext;    // EF Core for writes
    private readonly IOrderQueryRepository _queryRepo;   // Dapper for reads

    public async Task ProcessOrderAsync(OrderDto orderDto)
    {
        // Use EF Core for transactional writes
        var order = new Order { /* mapping */ };
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();
    }

    public async Task<IEnumerable<OrderSummary>> GetDashboardDataAsync()
    {
        // Use Dapper for complex read queries
        return await _queryRepo.GetOrderSummariesAsync();
    }
}
```

---

## Summary

Micro ORMs and Full ORMs represent trade-offs between control and convenience. Micro ORMs excel in performance-critical scenarios requiring precise SQL control, while Full ORMs accelerate development through abstraction and automation. The optimal choice depends on application requirements, team expertise, and performance constraints.
