---
title: "Tests d'Intégration avec TestContainers pour .NET"
date: 2026-04-08
draft: false
tags: ["testing", "integration", "testcontainers", "dotnet"]
categories: ["Testing"]
series: ["Tests"]
description: "De vraies bases de données, un vrai Redis, de vrais brokers, démarrés par run de test en quelques secondes. TestContainers pour .NET enterre le 'ça marche sur ma machine' pour les tests d'intégration."
---

Hello tous le monde, aujourd'hui on va explorer les **tests d'intégration avec TestContainers** pour .NET.

Les tests d'intégration, ça a longtemps été la pire partie d'un projet .NET. Un SQL Server de dev partagé que trois équipes se disputaient. Un docker-compose que tout le monde lançait "correctement" en local, jusqu'à ce qu'il dérive. Une CI avec une chaîne de connexion codée en dur qui marchait uniquement le mardi. Résultat : personne ne faisait confiance aux tests, et l'équipe se rabattait à mocker la base et à faire semblant. Si tu as lu l'article précédent sur [les tests unitaires en .NET](/fr/posts/testing-unit-testing/), tu sais déjà pourquoi ce repli est une fausse bonne idée : les bugs EF Core vivent dans le SQL généré, et tu ne peux pas les attraper en mockant `DbContext`.

TestContainers règle ce problème. La bibliothèque Java d'origine a été publiée en 2015 par Richard North, et le port .NET est arrivé en 2017 sous le nom Testcontainers for .NET. C'est aujourd'hui le standard officiel, maintenu sous l'organisation GitHub `testcontainers`, et .NET 10 le traite comme un outil de test d'intégration de première classe. L'idée est simple : ton code de test démarre un vrai Postgres / Redis / RabbitMQ / peu importe dans un container Docker jetable, attend qu'il soit prêt, te refile une chaîne de connexion, et le détruit quand la fixture de test se libère.

## Le contexte : pourquoi ce pattern existe

Supposons que nous ayons une équipe dont les tests d'intégration tournent contre une instance SQL Server partagée sur une VM de dev. Un test laisse une ligne derrière lui. Un autre test suppose que la ligne n'existe pas. Mardi matin, tout explose. Quelqu'un patche le test avec `DELETE FROM Orders WHERE ...` et le cycle recommence. Six mois plus tard, la moitié de la suite est désactivée.

Ce qu'il faut vraiment à cette équipe :

1. **Une vraie base de données**, pour que les migrations EF Core, les indexes et les requêtes tournent contre le moteur qui sera en prod.
2. **De l'isolation par run de test**, pour que personne ne laisse d'état à personne.
3. **Une expérience développeur en une commande**, pour qu'un nouvel arrivant clone le repo, lance `dotnet test`, et que tout marche.

TestContainers livre les trois en déléguant la partie compliquée à Docker.

## Vue d'ensemble : comment ça se branche

Avant de rentrer dans le code, voici comment TestContainers s'installe dans un projet de test .NET :

{{< mermaid >}}
graph TD
    A[Fixture de test] --> B[Bibliothèque Testcontainers]
    B --> C[Daemon Docker]
    C --> D[Container Postgres]
    C --> E[Container Redis]
    A --> F[Ton SUT<br/>Repository + DbContext]
    F --> D
    F --> E
{{< /mermaid >}}

La fixture de test possède le cycle de vie des containers. Le SUT reçoit une vraie chaîne de connexion et ne sait absolument pas qu'il parle à un container qui disparaîtra dans 20 secondes.

> 💡 **Info** : TestContainers a besoin d'un Docker qui tourne sur la machine (Docker Desktop sous Windows/macOS, ou Docker rootless sous Linux). En CI, GitHub Actions et Azure DevOps fournissent tous les deux des runners Docker-in-Docker out of the box.

## Zoom : une fixture Postgres avec xUnit

Voici le minimum pour démarrer Postgres, appliquer les migrations EF Core, et le rendre disponible aux tests :

```csharp
using Testcontainers.PostgreSql;
using Microsoft.EntityFrameworkCore;
using Xunit;

public sealed class PostgresFixture : IAsyncLifetime
{
    public PostgreSqlContainer Container { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .WithDatabase("shop_test")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public ShopDbContext CreateDbContext()
    {
        var options = new DbContextOptionsBuilder<ShopDbContext>()
            .UseNpgsql(Container.GetConnectionString())
            .Options;
        return new ShopDbContext(options);
    }

    public async ValueTask InitializeAsync()
    {
        await Container.StartAsync();
        await using var db = CreateDbContext();
        await db.Database.MigrateAsync();
    }

    public ValueTask DisposeAsync() => Container.DisposeAsync();
}
```

`IAsyncLifetime`, c'est le hook xUnit pour le setup et le teardown async. `StartAsync()` télécharge l'image (cachée après le premier run) et attend que Postgres soit prêt. Ensuite, EF Core applique tes vraies migrations dessus.

> ✅ **Bonne pratique** : Fixe le tag de l'image (`postgres:17-alpine`, pas `postgres:latest`). La CI doit être reproductible. Un "latest" qui change sous toi, c'est un bug en embuscade.

## Zoom : un test qui utilise la fixture

```csharp
[Collection("postgres")]
public class OrderRepositoryTests
{
    private readonly PostgresFixture _fixture;

    public OrderRepositoryTests(PostgresFixture fixture) => _fixture = fixture;

    [Fact]
    public async Task AddAsync_persiste_la_commande_avec_ses_lignes()
    {
        // Arrange
        await using var db = _fixture.CreateDbContext();
        var repo = new OrderRepository(db);
        var order = Order.Create(CustomerId.New());
        order.AddLine(new ProductId(1), 2, new Money(49.99m));

        // Act
        await repo.AddAsync(order, default);
        await db.SaveChangesAsync();

        // Assert
        await using var verify = _fixture.CreateDbContext();
        var loaded = await verify.Orders.Include(o => o.Lines)
            .FirstOrDefaultAsync(o => o.Id == order.Id);

        loaded.Should().NotBeNull();
        loaded!.Lines.Should().HaveCount(1);
        loaded.Lines.First().Subtotal.Amount.Should().Be(99.98m);
    }
}

[CollectionDefinition("postgres")]
public class PostgresCollection : ICollectionFixture<PostgresFixture> { }
```

`[Collection("postgres")]` dit à xUnit de partager la même fixture entre tous les tests de la collection. Un container, beaucoup de tests, rapide.

> 💡 **Info** : xUnit v3 utilise toujours les collection fixtures pour les ressources partagées coûteuses. La collection garantit que les tests à l'intérieur ne tournent pas en parallèle, ce qui est exactement ce qu'on veut quand ils partagent une base.

## Zoom : nettoyer entre les tests

Partager un container entre les tests, ça veut dire que les tests voient les données des autres. Deux stratégies classiques :

**1. Respawn (le plus rapide)** : la bibliothèque [Respawn](https://github.com/jbogard/Respawn) (encore de Jimmy Bogard) supprime toutes les lignes entre les tests, en gardant le schéma :

```csharp
public async Task ResetDatabaseAsync()
{
    await using var conn = new NpgsqlConnection(Container.GetConnectionString());
    await conn.OpenAsync();
    var respawner = await Respawner.CreateAsync(conn,
        new RespawnerOptions { DbAdapter = DbAdapter.Postgres });
    await respawner.ResetAsync(conn);
}
```

Appelle `ResetDatabaseAsync` dans le constructeur du test ou dans un `IAsyncLifetime` sur la classe de test.

**2. Rollback de transaction** : commence une transaction au début de chaque test, laisse le test tourner, rollback à la fin. Plus rapide que Respawn mais incompatible avec du code qui commit sa propre transaction.

> ⚠️ **Ça marche, mais...** : Un provider in-memory comme `Microsoft.EntityFrameworkCore.InMemory` est tentant parce qu'il est rapide, mais il ignore silencieusement les foreign keys, les contraintes, et tout le comportement SQL-spécifique. C'est ok pour tester des services avec une logique EF triviale et dangereux pour tout ce qui touche à une vraie requête. Préfère un vrai container Postgres.

> ❌ **Ne jamais faire** : Ne pointe pas tes tests d'intégration vers une base de dev partagée. Tu découvriras, dans la douleur, que deux devs qui lancent la suite en même temps corrompent l'état l'un de l'autre. TestContainers supprime l'excuse.

## Zoom : composer plusieurs services

Les vraies applis ont besoin de plus qu'une base. Redis pour le cache, RabbitMQ pour les messages, MinIO pour du stockage compatible S3. TestContainers compose tout ça dans la même fixture :

```csharp
public sealed class AppServicesFixture : IAsyncLifetime
{
    public PostgreSqlContainer Postgres { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine").Build();

    public RedisContainer Redis { get; } = new RedisBuilder()
        .WithImage("redis:7-alpine").Build();

    public async ValueTask InitializeAsync()
    {
        await Task.WhenAll(Postgres.StartAsync(), Redis.StartAsync());
    }

    public async ValueTask DisposeAsync()
    {
        await Postgres.DisposeAsync();
        await Redis.DisposeAsync();
    }
}
```

`Task.WhenAll` les démarre en parallèle, économisant quelques secondes par run. Le premier run télécharge les images, les suivants réutilisent le cache Docker et démarrent en moins de deux secondes chacun.

> ✅ **Bonne pratique** : Mets la fixture dans un projet de test partagé et référence-la depuis `IntegrationTests`, `ApiTests`, et `E2ETests`. Une seule source de vérité pour ce dont ton appli dépend.

## Quand c'est surdimensionné

Tous les projets n'ont pas besoin de TestContainers. Un service sans base de données qui ne parle qu'à des APIs HTTP stateless peut tout tester en unitaire + `WebApplicationFactory`. Un prototype qui sera réécrit dans deux mois n'a probablement pas besoin de cette mise en place.

Sors TestContainers quand :

- Tu as de vraies requêtes EF Core dont le SQL généré compte.
- Tes tests doivent prouver qu'une migration s'applique proprement.
- Tu dépends de Redis, d'un broker de messages ou d'un stockage compatible S3 dont le vrai comportement compte.
- Tu as plus d'un dev et tu veux que "clone et test" marche vraiment dès le premier jour.

## Wrap-up

Tu sais maintenant comment faire démarrer de vraies bases de données et dépendances pour tes tests d'intégration avec TestContainers : choisir un container builder, câbler une fixture xUnit avec `IAsyncLifetime`, appliquer les migrations EF Core dessus, la partager sur une collection de tests, et nettoyer l'état entre les tests avec Respawn ou une transaction. Tu peux composer Postgres, Redis et d'autres services dans la même fixture et offrir à ton équipe une expérience "clone et `dotnet test`" qui marche vraiment.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles](/fr/posts/testing-unit-testing/)

## Références

- [Testcontainers for .NET, documentation officielle](https://dotnet.testcontainers.org/)
- [Tests d'intégration dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/test/integration-tests)
- [Tester EF Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/ef/core/testing/)
- [Respawn sur GitHub](https://github.com/jbogard/Respawn)
