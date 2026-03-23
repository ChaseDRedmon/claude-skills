# EF Core Patterns Reference

Best practices for database access using Entity Framework Core without repository abstractions.

## Table of Contents
1. [Direct DbContext Usage](#direct-dbcontext-usage)
2. [Query Patterns](#query-patterns)
3. [Command Patterns](#command-patterns)
4. [Configuration](#configuration)
5. [Migrations](#migrations)
6. [Testing with EF Core](#testing-with-ef-core)

---

## Direct DbContext Usage

EF Core already implements both the Repository and Unit of Work patterns.
Do NOT wrap it in additional abstractions.

### Why No Repository Layer
```csharp
// ❌ Pointless abstraction — adds complexity, hides EF features
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct);
    Task<User?> GetByEmailAsync(string email, CancellationToken ct);
    Task AddAsync(User user, CancellationToken ct);
    Task SaveChangesAsync(CancellationToken ct);
}

public class UserRepository : IUserRepository
{
    private readonly AppDbContext _db;
    
    public async Task<User?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _db.Users.FindAsync(new object[] { id }, ct);

    public async Task<User?> GetByEmailAsync(string email, CancellationToken ct)
        => await _db.Users.SingleOrDefaultAsync(u => u.Email == email, ct);

    // Every new query requires a new interface method + implementation
    // Every custom query requires breaking the abstraction or adding yet another method
}
```

```csharp
// ✅ Direct DbContext — simple, powerful, full access to LINQ-to-SQL
public sealed class GetUserHandler(AppDbContext db)
{
    public async Task<UserResponse?> Handle(GetUserQuery query, CancellationToken ct)
    {
        return await db.Users
            .AsNoTracking()
            .Where(u => u.Id == query.UserId)
            .Select(u => new UserResponse(u.Id, u.Email, u.DisplayName))
            .SingleOrDefaultAsync(ct);
    }
}
```

### DbContext Registration
```csharp
// Program.cs
services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString)
           .UseSnakeCaseNamingConvention());
```

---

## Query Patterns

### Always Use AsNoTracking for Reads
```csharp
// ✅ No change tracking overhead for queries
var users = await db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .ToListAsync(ct);
```

### Always Project to DTOs
```csharp
// ✅ Only loads needed columns, runs as efficient SQL
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new OrderListItem(
        o.Id,
        o.CreatedAt,
        o.Total,
        o.Status.ToString(),
        o.Items.Count
    ))
    .ToListAsync(ct);

// ❌ Loads entire entity graph then maps in memory
var orders = await db.Orders
    .Include(o => o.Items)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);
var dtos = orders.Select(o => MapToDto(o)).ToList();
```

### Avoiding N+1 Queries
```csharp
// ❌ N+1 — executes 1 query for orders + N queries for items
var orders = await db.Orders.ToListAsync(ct);
foreach (var order in orders)
{
    var items = await db.OrderItems
        .Where(i => i.OrderId == order.Id)
        .ToListAsync(ct);
}

// ✅ Option 1: Eager loading
var orders = await db.Orders
    .Include(o => o.Items)
    .ToListAsync(ct);

// ✅ Option 2: Projection (preferred — no tracking, minimal data)
var orders = await db.Orders
    .Select(o => new
    {
        o.Id,
        o.Total,
        Items = o.Items.Select(i => new { i.ProductName, i.Quantity })
    })
    .ToListAsync(ct);
```

### Pagination
```csharp
public sealed record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize
);

var totalCount = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .CountAsync(ct);

var items = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .OrderByDescending(o => o.CreatedAt)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .Select(o => new OrderListItem(o.Id, o.CreatedAt, o.Total))
    .ToListAsync(ct);

return new PagedResult<OrderListItem>(items, totalCount, page, pageSize);
```

### Exists / Any Checks
```csharp
// ✅ Efficient — stops at first match, returns bool
var emailExists = await db.Users.AnyAsync(u => u.Email == email, ct);

// ❌ Loads entire entity just to check existence
var user = await db.Users.FirstOrDefaultAsync(u => u.Email == email, ct);
if (user is not null) { /* exists */ }
```

---

## Command Patterns

### Create
```csharp
public sealed class CreateOrderHandler(AppDbContext db)
{
    public async Task<Guid> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = command.CustomerId,
            CreatedAt = DateTime.UtcNow,
            Status = OrderStatus.Pending
        };

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

### Update
```csharp
public sealed class UpdateOrderHandler(AppDbContext db)
{
    public async Task Handle(UpdateOrderCommand command, CancellationToken ct)
    {
        var order = await db.Orders.SingleAsync(o => o.Id == command.OrderId, ct);

        order.ShippingAddress = command.NewAddress;
        order.UpdatedAt = DateTime.UtcNow;

        await db.SaveChangesAsync(ct);
    }
}
```

### Delete
```csharp
public sealed class DeleteOrderHandler(AppDbContext db)
{
    public async Task Handle(DeleteOrderCommand command, CancellationToken ct)
    {
        var order = await db.Orders.SingleAsync(o => o.Id == command.OrderId, ct);
        db.Orders.Remove(order);
        await db.SaveChangesAsync(ct);
    }
}
```

### Bulk Operations
```csharp
// ✅ EF Core 7+ ExecuteUpdateAsync — no entity loading
await db.Users
    .Where(u => u.LastLoginAt < cutoffDate)
    .ExecuteUpdateAsync(u => u.SetProperty(x => x.IsActive, false), ct);

// ✅ EF Core 7+ ExecuteDeleteAsync
await db.AuditLogs
    .Where(l => l.CreatedAt < retentionDate)
    .ExecuteDeleteAsync(ct);
```

---

## Configuration

### Entity Configuration (Fluent API)
```csharp
// Infrastructure/Database/Configurations/OrderConfiguration.cs
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Email).HasMaxLength(256).IsRequired();
        builder.HasIndex(o => o.Email).IsUnique();
        builder.Property(o => o.Status).HasConversion<string>();

        builder.HasMany(o => o.Items)
               .WithOne()
               .HasForeignKey(i => i.OrderId)
               .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### DbContext
```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options)
    : DbContext(options)
{
    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

## Migrations

### Creating Migrations
```bash
dotnet ef migrations add CreateUsersTable -o Infrastructure/Database/Migrations
```

### Migration Best Practices
- Name migrations descriptively: `AddEmailIndexToUsers`, not `Migration001`
- Review generated SQL before applying to production
- Never modify a migration that has been applied to any environment
- Use `HasData()` sparingly — prefer seed scripts for complex data

---

## Testing with EF Core

### Integration Tests with Real Database
```csharp
// Prefer real database (TestContainers) over InMemory provider
public sealed class CreateUserHandlerTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder().Build();
    private AppDbContext _db = null!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_postgres.GetConnectionString())
            .Options;
        _db = new AppDbContext(options);
        await _db.Database.MigrateAsync();
    }

    [Fact]
    public async Task Handle_ValidCommand_CreatesUser()
    {
        var handler = new CreateUserHandler(_db);
        var command = new CreateUserCommand("test@example.com", "Password123!");

        var userId = await handler.Handle(command, CancellationToken.None);

        var user = await _db.Users.SingleAsync(u => u.Id == userId);
        Assert.Equal("test@example.com", user.Email);
    }

    public async Task DisposeAsync() => await _postgres.DisposeAsync();
}
```

### Why Not InMemory Provider
- Doesn't support transactions, constraints, or indexes
- Doesn't translate LINQ to SQL (masks N+1 queries)
- Gives false confidence — tests pass but production fails
- Use SQLite for fast tests, PostgreSQL/SQL Server containers for accuracy
