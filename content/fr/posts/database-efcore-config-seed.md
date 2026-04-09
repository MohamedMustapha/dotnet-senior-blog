---
title: "EF Core : Configuration et Seed"
date: 2026-04-09
draft: false
tags: ["database", "efcore", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Les data annotations suffisent pour démarrer, la Fluent API est ce qui tient en production. Tour pratique de la configuration de modèle EF Core, d'IEntityTypeConfiguration et du nouveau hook UseSeeding qui remplace HasData."
---

Hello tous le monde, aujourd'hui on va explorer **la configuration de modèle EF Core et le seeding des données de référence**.

Il existe deux manières de configurer un modèle EF Core, et celle que la plupart des tutoriels montrent est celle qu'on finit par abandonner en premier. Les data annotations sur les propriétés d'entités suffisent pour une démo. Les applications réelles finissent avec des types de colonnes qui ne se mappent pas un-pour-un aux types CLR, des owned types, des value converters, des index filtrés, des contraintes check, et des données de référence qui doivent exister en base avant le démarrage de l'application. Tout ça vit dans la Fluent API, et depuis EF Core 9 l'histoire du seeding a été réécrite.

Cet article montre comment configurer proprement un modèle EF Core non trivial, où ranger cette configuration, et comment seeder les données de référence avec les nouveaux hooks `UseSeeding` / `UseAsyncSeeding`.

## Le contexte : pourquoi la configuration compte plus qu'il n'y paraît

Imaginons que nous ayons un `DbContext` laissé aux conventions. EF Core construit le modèle au démarrage en reflétant sur le contexte et en appliquant, dans l'ordre, les conventions, puis les data annotations, puis les appels Fluent API dans `OnModelCreating`. Chaque étape peut écraser la précédente. Quand on laisse tout aux conventions, EF Core devine les types de colonnes, la nullabilité, la longueur des strings et les relations à partir de la forme CLR. Ces devinettes sont raisonnables pour une démo, et fausses pour la production dès qu'on tient à la précision des `decimal`, au `rowversion` SQL Server, au `jsonb` PostgreSQL ou à un index unique composite.

Les data annotations (`[Required]`, `[MaxLength(200)]`, `[Column(TypeName = "decimal(18,2)")]`) résolvent les cas simples et polluent les classes du domaine avec des préoccupations de persistance. Et elles ne savent pas exprimer la moitié de ce dont on a besoin : pas de value converter, pas d'index filtré, pas de clé composite ordonnée, pas de seed. Donc dans tout projet qui dépasse le CRUD sur une table, on déplace la configuration vers la Fluent API, et on la range dans des classes dédiées `IEntityTypeConfiguration<T>` plutôt que de laisser `OnModelCreating` devenir un mur de 600 lignes.

## Vue d'ensemble : où vit la configuration

{{< mermaid >}}
graph TD
    A[DbContext] --> B[OnModelCreating]
    B --> C[ApplyConfigurationsFromAssembly]
    C --> D[OrderConfiguration : IEntityTypeConfiguration&lt;Order&gt;]
    C --> E[CustomerConfiguration : IEntityTypeConfiguration&lt;Customer&gt;]
    C --> F[ProductConfiguration : IEntityTypeConfiguration&lt;Product&gt;]
    A --> G[UseSeeding / UseAsyncSeeding]
    G --> H[Données de référence : Pays, Rôles, Devises]
{{< /mermaid >}}

Trois idées à garder en tête :

1. Une classe de configuration par agrégat racine ou par entité. À ranger à côté de l'entité ou dans un dossier `Configurations/` dédié.
2. `OnModelCreating` se réduit à un seul appel `modelBuilder.ApplyConfigurationsFromAssembly(typeof(MyDbContext).Assembly)`.
3. Le seeding ne passe plus par `HasData` dès que la donnée est un peu dynamique. On utilise `UseSeeding` pour le chemin synchrone (outillage de migration, `EnsureCreated`) et `UseAsyncSeeding` pour le runtime.

## Zoom : un DbContext réaliste

Prenons un petit backend e-commerce avec des commandes, des clients et des produits.

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

C'est l'intégralité de `OnModelCreating`. Chaque détail de chaque entité part dans son propre fichier.

## Zoom : IEntityTypeConfiguration en pratique

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

Plusieurs choses ici que les data annotations ne savent pas faire : un owned type avec noms de colonnes explicites, un enum persisté en string pour la lisibilité en base, une clé primaire `ValueGeneratedNever` parce que l'ID est généré dans le domaine, un `DeleteBehavior` explicite par relation, et une propriété shadow `RowVersion` pour la concurrence optimiste.

> 💡 **Info** — `HasConversion<string>()` est la manière moderne de persister un enum en texte. Ça survit au réordonnancement de l'enum et c'est bien plus lisible dans un dump SQL que des codes entiers.

> ✅ **Bonne pratique** — Une entité, un fichier de configuration. Quand la configuration grossit, on étend le fichier, pas `OnModelCreating`. Le test du "c'est propre ?" : est-ce qu'un nouveau développeur peut trouver la définition de la table `Order` sans un full-text search.

> ⚠️ **Ça marche, mais...** — Saupoudrer `[Column(TypeName = "...")]` sur les propriétés d'entités fonctionne, mais ça couple le domaine au provider de base. On garde les annotations pour les règles vraiment universelles (`[Required]` quand c'est un invariant métier) et on pose les types spécifiques au provider dans la Fluent API.

## Zoom : value converters pour les types du domaine

Quand le domaine utilise des IDs fortement typés ou des value objects, les value converters sont ce qui garde le mapping honnête.

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

`CustomerId` reste un type de première classe dans le domaine et se mappe quand même sur une simple colonne `uuid`. EF Core 8 a introduit le support des collections primitives et amélioré les complex types, donc les value objects n'obligent plus à aplatir les propriétés à la main.

> 💡 **Info** — `Guid.CreateVersion7()` est disponible depuis .NET 9 et produit des GUID ordonnés dans le temps, bien plus sympathiques à l'indexation qu'un GUID v4 aléatoire en clé primaire ou clustered.

## Zoom : seeding avec UseSeeding et UseAsyncSeeding

EF Core 9 a introduit `UseSeeding` et `UseAsyncSeeding`, et c'est enfin un endroit où ranger la logique de seed qui tourne à la fois au design time (quand on exécute `dotnet ef database update`) et au runtime (quand l'application appelle `context.Database.MigrateAsync()`), sans les limites de `HasData`.

`HasData` stocke les lignes de seed dans les migrations sous forme d'`INSERT` littéraux. Ça marche pour des données de référence vraiment statiques, ça casse dès que la ligne dépend de quelque chose de dynamique (mot de passe haché, clé étrangère calculée, configuration), et chaque modification de ligne produit une migration. Pour tout ce qui dépasse une liste fixe de codes pays, on veut `UseSeeding`.

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
                       new Country { Code = "BE", Name = "Belgique" },
                       new Country { Code = "CH", Name = "Suisse" });
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
                       new Country { Code = "BE", Name = "Belgique" },
                       new Country { Code = "CH", Name = "Suisse" });
                   await shop.SaveChangesAsync(ct);
               }
           });
});
```

Les deux hooks tournent quand on appelle `context.Database.EnsureCreatedAsync()` ou quand l'outillage EF amorce la base. Au runtime, la variante async tourne sur le thread pool ; au design time, la variante sync tourne depuis l'outillage. EF Core ne les appelle pas tout seul à chaque démarrage d'application : on déclenche le seed explicitement, en général juste après `MigrateAsync()` dans le pipeline de démarrage.

```csharp
public static async Task MigrateAndSeedAsync(this IServiceProvider services)
{
    await using var scope = services.CreateAsyncScope();
    var db = scope.ServiceProvider.GetRequiredService<ShopDbContext>();
    await db.Database.MigrateAsync();
    await db.Database.EnsureCreatedAsync();
}
```

> ✅ **Bonne pratique** — Le seed doit être idempotent. Chaque bloc de seed vérifie "cette ligne existe-t-elle déjà ?" avant d'insérer. Redémarrer l'application, relancer les migrations en CI ou un pod qui redémarre en Kubernetes, tout ça doit être sûr.

> ❌ **Ne jamais faire** — Ne pas seeder de la donnée utilisateur (clients, commandes, fixtures de test) via `UseSeeding`. Ce hook est réservé aux données de référence : pays, devises, rôles, permissions, tables de lookup. La donnée de test appartient aux fixtures de test, pas à la configuration du `DbContext`.

> ⚠️ **Ça marche, mais...** — Si un code legacy s'appuie encore sur `HasData`, ça continue de fonctionner. Le coût de migration vers `UseSeeding` reste faible et apporte la possibilité de seeder n'importe quelle donnée calculée, y compris des lignes qui dépendent de variables d'environnement ou de clés étrangères existantes.

## Zoom : les conventions qu'on veut souvent écraser

Quelques petits changements au niveau du modèle payent sur tout le contexte :

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

`ConfigureConventions` (EF Core 6+) permet de définir des valeurs par défaut pour toute propriété d'un type CLR donné. Ça divise drastiquement le boilerplate par entité et fait disparaître le warning "j'ai oublié de fixer la précision du decimal".

## Wrap-up

Tu sais maintenant comment poser une configuration EF Core qui tient en production : Fluent API plutôt qu'annotations, un `IEntityTypeConfiguration<T>` par entité, des value converters pour les IDs typés et les value objects, et les hooks modernes `UseSeeding` / `UseAsyncSeeding` pour les données de référence. Tu peux supprimer le `OnModelCreating` de 600 lignes et le remplacer par un seul `ApplyConfigurationsFromAssembly`, et arrêter de gonfler les migrations avec des `INSERT` générés par `HasData`.

Prêt à repasser sur le `DbContext` de ton prochain projet et à déplacer la configuration là où elle mérite d'être, ou à partager la recette avec ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [Accès aux données en .NET : Repository Pattern, ou pas ?](/fr/posts/layer-focused-repository-or-not/)
- [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)

## Références

- [EF Core : Modeling](https://learn.microsoft.com/fr-fr/ef/core/modeling/)
- [EF Core : IEntityTypeConfiguration](https://learn.microsoft.com/fr-fr/dotnet/api/microsoft.entityframeworkcore.ientitytypeconfiguration-1)
- [EF Core 9 : UseSeeding et UseAsyncSeeding](https://learn.microsoft.com/fr-fr/ef/core/what-is-new/ef-core-9.0/whatsnew#database-seeding)
- [EF Core : Value Conversions](https://learn.microsoft.com/fr-fr/ef/core/modeling/value-conversions)
- [EF Core : Configuration de conventions](https://learn.microsoft.com/fr-fr/ef/core/modeling/bulk-configuration#pre-convention-configuration)
