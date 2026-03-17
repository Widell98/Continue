---
name: Service Rules
description: >
  Använd denna när frågan handlar om services, repositories, interfaces,
  dependency injection, DI-registrering, IOrderService, service-lager,
  CancellationToken i service-metoder eller hur ViewModel kommunicerar
  med backend/databas via ett service-lager.
alwaysApply: false
globs:
  - "**/*Service.cs"
  - "**/*Repository.cs"
  - "**/I*Service.cs"
  - "**/I*Repository.cs"
---

# Service / Repository-regler

## Grundregler
- Alla tjänster bakom interface: `IOrderService`, `ICustomerService`
- ViewModel rör aldrig DbContext direkt
- Tjänster registreras i DI vid uppstart — new:as aldrig manuellt
- Alla service-metoder är async med `CancellationToken`

## Interface-mönster
```csharp
public interface IOrderService
{
    Task<IReadOnlyList<Order>> GetOrdersAsync(CancellationToken ct = default);
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task CreateAsync(Order order, CancellationToken ct = default);
    Task UpdateAsync(Order order, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}
```

## Implementationsmönster
```csharp
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
```

## DI-registrering
```csharp
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<ICustomerService, CustomerService>();
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

## Namngivning
| Typ | Mönster | Exempel |
|-----|---------|---------|
| Interface | `I[Name]Service` | `IOrderService` |
| Implementation | `[Name]Service` | `OrderService` |
| Repository interface | `I[Name]Repository` | `IOrderRepository` |
| Repository implementation | `[Name]Repository` | `OrderRepository` |
