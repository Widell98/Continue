---
---
name: Data Rules
description: Use this when the question involves DbContext, EF Core, migrations or database queries
alwaysApply: false
---
globs: ["**/*Context.cs", "**/*Migration*.cs", "**/Migrations/*.cs", "**/*Configuration.cs"]
---

# EF Core / Data-regler

## Grundregler
- Use IDbContextFactory<AppDbContext> for async operations
- Never use lazy loading — always explicit Include()
- AsNoTracking() on all read-only queries
- Never reference DbContext in ViewModel

## DbContext pattern
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

## Entity configuration
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

## Query pattern
await using var context = await _contextFactory.CreateDbContextAsync(ct);
var orders = await context.Orders
    .AsNoTracking()
    .Include(x => x.Customer)
    .Where(x => x.Status == OrderStatus.Active)
    .OrderByDescending(x => x.CreatedAt)
    .ToListAsync(ct);

## Migrations
- Always named descriptively: AddOrderStatusColumn, CreateCustomerTable
- Never edit existing migrations — always add new
- Run: dotnet ef migrations add MigrationName
- Apply: dotnet ef database update

## Naming
- DbContext: AppDbContext eller [Module]DbContext
- Configuration: [Entity]Configuration
- Migration: [Action][Entity][Detail] ex: AddOrderStatusColumn