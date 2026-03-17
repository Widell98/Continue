---
name: Data Rules
description: >
  Använd denna när frågan handlar om DbContext, EF Core, migrationer,
  databasfrågor, IDbContextFactory, entity-konfigurationer, Include,
  AsNoTracking, LINQ mot databasen eller databasschema.
alwaysApply: false
globs:
  - "**/*Context.cs"
  - "**/*Migration*.cs"
  - "**/Migrations/*.cs"
  - "**/*Configuration.cs"
---

# EF Core / Data-regler

## Grundregler
- Använd `IDbContextFactory<AppDbContext>` för async-operationer
- Aldrig lazy loading — alltid explicit `Include()`
- `AsNoTracking()` på alla läs-frågor
- DbContext refereras aldrig i ViewModel

## DbContext-mönster
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

## Entity-konfiguration
```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(x => x.Id);
        builder.Property(x => x.OrderNumber).IsRequired().HasMaxLength(50);
        builder.HasOne(x => x.Customer)
               .WithMany(x => x.Orders)
               .HasForeignKey(x => x.CustomerId);
    }
}
```

## Frågemönster
```csharp
await using var context = await _contextFactory.CreateDbContextAsync(ct);
var orders = await context.Orders
    .AsNoTracking()
    .Include(x => x.Customer)
    .Where(x => x.Status == OrderStatus.Active)
    .OrderByDescending(x => x.CreatedAt)
    .ToListAsync(ct);
```

## Migrationer
- Alltid beskrivande namn: `AddOrderStatusColumn`, `CreateCustomerTable`
- Redigera aldrig befintliga migrationer — lägg alltid till nya
- Skapa: `dotnet ef migrations add MigrationName`
- Applicera: `dotnet ef database update`

## Namngivning
| Typ | Mönster | Exempel |
|-----|---------|---------|
| DbContext | `AppDbContext` eller `[Module]DbContext` | `AppDbContext` |
| Konfiguration | `[Entity]Configuration` | `OrderConfiguration` |
| Migration | `[Action][Entity][Detail]` | `AddOrderStatusColumn` |
