---
name: Service Rules  
description: Use this when the question involves services, repositories, interfaces or dependency injection
alwaysApply: false
globs: ["**/*Service.cs", "**/*Repository.cs", "**/*Interface.cs", "**/I*.cs"]
---

# Service / Repository-regler

## Grundregler
- All services behind interfaces: IOrderService, ICustomerService
- ViewModel never touches DbContext directly
- Services registered in DI at startup — never new'd manually
- All service methods async with CancellationToken

## Interface pattern
public interface IOrderService
{
    Task<IReadOnlyList<Order>> GetOrdersAsync(CancellationToken ct = default);
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task CreateAsync(Order order, CancellationToken ct = default);
    Task UpdateAsync(Order order, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}

## Implementation pattern
public class OrderService : IOrderService
{
    private readonly IDbContextFactory<AppDbContext> _contextFactory;

    public OrderService(IDbContextFactory<AppDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task<IReadOnlyList<Order>> GetOrdersAsync(CancellationToken ct = default)
    {
        await using var context = await _contextFactory.CreateDbContextAsync(ct);
        return await context.Orders
            .AsNoTracking()
            .OrderByDescending(x => x.CreatedAt)
            .ToListAsync(ct);
    }
}

## DI registration
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<ICustomerService, CustomerService>();
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

## Naming
- Interface: I[Name]Service
- Implementation: [Name]Service
- Repository interface: I[Name]Repository
- Repository implementation: [Name]Repository