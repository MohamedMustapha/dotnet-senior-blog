---
title: "EF Core: Configuration and Seeding"
date: 2026-04-09
draft: false
tags: ["database", "efcore", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Data annotations get you started, Fluent API gets you to production. A practical tour of EF Core model configuration, IEntityTypeConfiguration, and the modern UseSeeding hook that replaces HasData."
---

There are two ways to configure an EF Core model, and the one most tutorials show you is the one you outgrow first. Data annotations on entity properties are fine for demos. Real applications end up with column types that do not map one-to-one to CLR types, owned types, value converters, filtered indexes, check constraints, and seed data that must land in the database before the app starts. All of that lives in the Fluent API, and as of EF Core 9 the seeding story has been rewritten.

This article walks through how to configure a non-trivial EF Core model cleanly, where to put that configuration, and how to seed reference data using the new `UseSeeding` / `UseAsyncSeeding` hooks.

## Why configuration matters more than it looks

EF Core builds a model at startup by reflecting over your `DbContext` and applying every piece of configuration it can find: conventions first, then data annotations, then Fluent API calls in `OnModelCreating`. The order matters because each step can override the previous one. When you leave everything to conventions, EF Core guesses column types, nullability, string length, and relationships from the CLR shape. Those guesses are reasonable for a demo and wrong for production the moment you care about `decimal` precision, SQL Server `rowversion`, PostgreSQL `jsonb`, or a composite unique index.

Data annotations (`[Required]`, `[MaxLength(200)]`, `[Column(TypeName = "decimal(18,2)")]`) solve the simple cases and pollute your domain classes with persistence concerns. They also cannot express half of what you need: no way to configure a value converter, no way to declare a filtered index, no way to set a composite key with ordering, no way to seed. So in any project beyond CRUD-over-one-table, you move configuration to the Fluent API, and you put it in dedicated `IEntityTypeConfiguration<T>` classes instead of letting `OnModelCreating` become a 600-line wall.

## Overview: where configuration lives

{{< mermaid >}}
graph TD
    A[DbContext] --> B[OnModelCreating]
    B --> C[ApplyConfigurationsFromAssembly]
    C --> D[OrderConfiguration : IEntityTypeConfiguration&lt;Order&gt;]
    C --> E[CustomerConfiguration : IEntityTypeConfiguration&lt;Customer&gt;]
    C --> F[ProductConfiguration : IEntityTypeConfiguration&lt;Product&gt;]
    A --> G[UseSeeding / UseAsyncSeeding]
    G --> H[Reference data: Countries, Roles, Currencies]
{{< /mermaid >}}

Three ideas to keep in mind:

1. One configuration class per aggregate root or entity. Colocate it with the entity or keep a dedicated `Configurations/` folder.
2. `OnModelCreating` becomes a single `modelBuilder.ApplyConfigurationsFromAssembly(typeof(MyDbContext).Assembly)` call.
3. Seeding is no longer done via `HasData` for anything dynamic. Use `UseSeeding` for the sync path (used by `EnsureCreated` and migrations tooling) and `UseAsyncSeeding` for runtime.

## Zoom: a realistic DbContext

Let's say we model a small e-commerce backend with orders, customers, and products.

```csharp
public sealed class ShopDbContext : DbContext
{
    public ShopDbContext(DbContextOptions<ShopDbContext> options) : base(options) { }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Country> Countries => Set<Country>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ShopDbContext).Assembly);
    }
}
```

That is the entire `OnModelCreating`. Every detail of every entity moves into its own file.

## Zoom: IEntityTypeConfiguration in practice

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");

        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
               .ValueGeneratedNever();

        builder.Property(o => o.Reference)
               .HasMaxLength(32)
               .IsRequired();

        builder.HasIndex(o => o.Reference).IsUnique();

        builder.Property(o => o.Status)
               .HasConversion<string>()
               .HasMaxLength(20);

        builder.Property(o => o.TotalAmount)
               .HasColumnType("decimal(18,2)");

        builder.Property(o => o.CreatedAt)
               .HasColumnType("timestamptz");

        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("shipping_street").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("shipping_city").HasMaxLength(100);
            addr.Property(a => a.PostalCode).HasColumnName("shipping_postal").HasMaxLength(20);
            addr.Property(a => a.CountryCode).HasColumnName("shipping_country").HasMaxLength(2);
        });

        builder.HasOne(o => o.Customer)
               .WithMany(c => c.Orders)
               .HasForeignKey(o => o.CustomerId)
               .OnDelete(DeleteBehavior.Restrict);

        builder.HasMany(o => o.Lines)
               .WithOne()
               .HasForeignKey("OrderId")
               .OnDelete(DeleteBehavior.Cascade);

        builder.Property<uint>("RowVersion").IsRowVersion();
    }
}
```

A few things are happening here that data annotations cannot express: owned type mapping with explicit column names, an enum stored as string for readability in the database, a value-generated-never primary key because we generate IDs in the domain, an explicit `DeleteBehavior` per relation, and a shadow `RowVersion` property for optimistic concurrency.

> 💡 **Info** — `HasConversion<string>()` is the modern way to persist enums as text. It survives enum reordering and is much easier to read in SQL dumps than integer codes.

> ✅ **Good practice** — Configure one entity per file. When the configuration grows, you extend the file, not `OnModelCreating`. The test for "is this clean?" is: can a new developer find the table definition of `Order` without a full-text search.

> ⚠️ **Works, but...** — Sprinkling `[Column(TypeName = "...")]` on entity properties works, but it couples the domain to the database provider. Keep annotations for truly universal rules (`[Required]` when it reflects a domain invariant) and put provider-specific types in Fluent API.

## Zoom: value converters for domain types

When your domain uses strongly-typed IDs or value objects, value converters are what keep the mapping honest.

```csharp
public readonly record struct CustomerId(Guid Value)
{
    public static CustomerId New() => new(Guid.CreateVersion7());
}

public sealed class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("customers");

        builder.HasKey(c => c.Id);

        builder.Property(c => c.Id)
               .HasConversion(
                   id => id.Value,
                   value => new CustomerId(value))
               .ValueGeneratedNever();

        builder.Property(c => c.Email)
               .HasMaxLength(256)
               .IsRequired();

        builder.HasIndex(c => c.Email).IsUnique();
    }
}
```

This keeps `CustomerId` as a first-class domain type and still maps to a plain `uuid` column. EF Core 8 introduced primitive collections and improved support for complex types, so value objects no longer require you to flatten them manually.

> 💡 **Info** — `Guid.CreateVersion7()` is available since .NET 9 and produces time-ordered GUIDs, which index much better than random v4 GUIDs as clustered or primary keys.

## Zoom: seeding with UseSeeding and UseAsyncSeeding

EF Core 9 introduced `UseSeeding` and `UseAsyncSeeding`, and they finally give you a place to put seed logic that runs both at design time (when you call `dotnet ef database update`) and at runtime (when your app calls `context.Database.MigrateAsync()`), without the limitations of `HasData`.

`HasData` stores seed rows in migrations as literal `INSERT` statements. That works for truly static reference data, breaks the moment the seed row depends on anything dynamic (hashed password, computed foreign key, configuration), and makes every change to a seed row produce a migration. For anything beyond a fixed list of country codes, you want `UseSeeding`.

```csharp
services.AddDbContext<ShopDbContext>(options =>
{
    options.UseNpgsql(connectionString)
           .UseSeeding((context, _) =>
           {
               var shop = (ShopDbContext)context;
               if (!shop.Countries.Any())
               {
                   shop.Countries.AddRange(
                       new Country { Code = "FR", Name = "France" },
                       new Country { Code = "BE", Name = "Belgium" },
                       new Country { Code = "CH", Name = "Switzerland" });
                   shop.SaveChanges();
               }
           })
           .UseAsyncSeeding(async (context, _, ct) =>
           {
               var shop = (ShopDbContext)context;
               if (!await shop.Countries.AnyAsync(ct))
               {
                   shop.Countries.AddRange(
                       new Country { Code = "FR", Name = "France" },
                       new Country { Code = "BE", Name = "Belgium" },
                       new Country { Code = "CH", Name = "Switzerland" });
                   await shop.SaveChangesAsync(ct);
               }
           });
});
```

Both hooks run when you call `context.Database.EnsureCreatedAsync()` or when the EF tooling bootstraps a database. At runtime, the async variant runs on the thread pool; at design time, the sync variant runs from the tooling. EF Core does not call them for you on every application startup: you trigger seeding explicitly, usually right after `MigrateAsync()` in your startup pipeline.

```csharp
public static async Task MigrateAndSeedAsync(this IServiceProvider services)
{
    await using var scope = services.CreateAsyncScope();
    var db = scope.ServiceProvider.GetRequiredService<ShopDbContext>();
    await db.Database.MigrateAsync();
    await db.Database.EnsureCreatedAsync();
}
```

> ✅ **Good practice** — Keep seeding idempotent. Every seed block must check "does this row already exist?" before inserting. Restarting the app, rerunning migrations in CI, or a pod restarting in Kubernetes should all be safe.

> ❌ **Never do** — Do not seed user-facing data (customers, orders, test fixtures) through `UseSeeding`. That hook is for reference data only: countries, currencies, roles, permissions, lookup tables. Test data belongs to test fixtures, not to the `DbContext` configuration.

> ⚠️ **Works, but...** — If you still rely on `HasData` in a legacy codebase, it continues to work. The migration cost to move to `UseSeeding` is low and buys you the ability to seed anything computed, including rows that depend on environment variables or existing foreign keys.

## Zoom: conventions you usually want to override

A few small changes at the model level pay off across the whole context:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<string>()
                        .HaveMaxLength(512);

    configurationBuilder.Properties<decimal>()
                        .HavePrecision(18, 2);

    configurationBuilder.Properties<DateTime>()
                        .HaveColumnType("timestamptz");
}
```

`ConfigureConventions` (EF Core 6+) lets you set defaults that apply to every property of a given CLR type. It cuts the amount of per-entity boilerplate dramatically and makes the "I forgot to set decimal precision" warning disappear.

## Wrap-up

You now have the configuration story that holds up in production: Fluent API over annotations, one `IEntityTypeConfiguration<T>` per entity, value converters for strongly-typed IDs and value objects, and the modern `UseSeeding` / `UseAsyncSeeding` hooks for reference data. You can drop the 600-line `OnModelCreating` and replace it with a single `ApplyConfigurationsFromAssembly` call, and you can stop bloating your migrations with `HasData` `INSERT` statements.

The next piece of the puzzle is migrations themselves: how to author them, how to review them, and how to avoid the classic traps when the database is already in production.

## Related articles

- [Data Access in .NET: Repository Pattern, or Not?](/en/posts/layer-focused-repository-or-not/)
- [Application Layer: CQS and CQRS](/en/posts/layer-focused-cqs-cqrs/)

## References

- [EF Core: Modeling](https://learn.microsoft.com/en-us/ef/core/modeling/)
- [EF Core: IEntityTypeConfiguration](https://learn.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.ientitytypeconfiguration-1)
- [EF Core 9: UseSeeding and UseAsyncSeeding](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew#database-seeding)
- [EF Core: Value Conversions](https://learn.microsoft.com/en-us/ef/core/modeling/value-conversions)
- [EF Core: Pre-convention model configuration](https://learn.microsoft.com/en-us/ef/core/modeling/bulk-configuration#pre-convention-configuration)
