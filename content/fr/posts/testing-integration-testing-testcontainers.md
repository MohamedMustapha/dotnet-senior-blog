---
title: "Tests d'Intégration avec TestContainers pour .NET"
date: 2026-04-08
draft: false
tags: ["testing", "integration", "testcontainers", "dotnet"]
categories: ["Testing"]
series: ["Tests"]
description: "De vraies bases de données, un vrai Redis, de vrais brokers, un vrai Keycloak, provisionnés par run de test. TestContainers pour .NET transforme les faux tests d'intégration en vrais, sans infrastructure partagée."
---

Hello tous le monde, aujourd'hui on va explorer les **tests d'intégration avec TestContainers** pour .NET.

Le test d'intégration est historiquement la discipline la plus compromise de la delivery .NET. Pas parce que les ingénieurs ne s'en souciaient pas, mais parce que les outils disponibles imposaient un choix entre une infrastructure partagée fragile et des tests qui, sans le dire, cessaient d'être des tests d'intégration. Si tu as lu l'article précédent sur [les tests unitaires en .NET](/fr/posts/testing-unit-testing/), tu sais déjà que mocker un `DbContext` ne peut pas attraper les bugs qui vivent dans le SQL généré. Ce que le métier attendait, c'était un moyen d'exécuter des tests d'intégration contre les vrais services auxquels ils prétendent s'intégrer, de façon reproductible, sans coordonner un environnement partagé.

TestContainers fournit exactement cela. La bibliothèque Java d'origine a été publiée en 2015 par Richard North, et le port .NET est arrivé en 2017 sous le nom Testcontainers for .NET. C'est aujourd'hui le standard de référence, maintenu sous l'organisation GitHub `testcontainers`, et .NET 10 le traite comme un outil de test d'intégration de première classe. Le principe est direct : ton code de test démarre un vrai Postgres, Redis, RabbitMQ, Keycloak, ou n'importe quel autre service dans un container Docker éphémère, attend qu'il devienne prêt, expose ses informations de connexion, et le détruit quand la fixture de test se libère.

## Le contexte : pourquoi ce pattern existe

Pendant la plus grande partie de l'histoire de .NET, écrire un test d'intégration honnête impliquait d'accepter cinq problèmes structurels, dont aucun n'avait de solution propre.

**1. Une infrastructure de dev ou d'intégration partagée.** La base, le fournisseur d'identité, le broker de messages, le stockage objet : tout vivait sur un environnement central vers lequel chaque développeur et chaque job de CI pointait. Lancer deux suites de tests en parallèle était un vrai risque : les fixtures d'un ingénieur entraient en collision avec celles d'un autre, un script de cleanup effaçait une ligne dont quelqu'un dépendait, et un test flaky ressemblait soudain à une vraie régression. Les équipes se défendaient avec des systèmes de verrous, des conventions de nommage, et des contrats sociaux implicites qui se cassaient à l'arrivée du premier nouvel arrivant.

**2. La CI/CD devait avoir un accès réseau à ces services partagés.** Les agents de build avaient besoin de routes vers la base de dev, de credentials renouvelés à la main, et de règles de firewall maintenues par une autre équipe. Chaque nouveau pipeline devenait un ticket. Chaque panne de l'infra partagée bloquait tous les builds. La disponibilité de la suite de tests était plafonnée par celle du service le moins fiable qu'elle appelait.

**3. Le setup se cassait avec une facilité déconcertante.** Un simple `ALTER TABLE` appliqué par un ingénieur en debug, un changement de rôle dans Keycloak, une expiration de certificat SSL sur le relais SMTP, un snapshot Redis obsolète : n'importe lequel invalidait silencieusement la suite pour tout le monde. Les matinées commençaient par la question "la CI est rouge à cause de mon changement, ou parce que quelqu'un a touché à l'environnement de test ?".

**4. Elle exigeait un cleanup et un entretien permanents de la part des développeurs eux-mêmes.** Les scripts de seed dérivaient par rapport aux migrations. Les utilisateurs de test s'accumulaient dans le fournisseur d'identité. Les lignes orphelines s'empilaient dans les tables de jointure. Quelqu'un dans l'équipe finissait par être le gardien officieux de l'environnement d'intégration, et son temps n'était jamais compté dans la planification.

**5. Et surtout : toute dépendance qui n'avait pas de package NuGet in-memory était mockée.** C'est la conséquence la plus grave, et celle que personne n'aime reconnaître. Si ton service parlait à SQL Server, tu avais `Microsoft.EntityFrameworkCore.InMemory` et tu faisais comme si ça comptait, alors qu'il ignore silencieusement les foreign keys, la sensibilité à la casse, et tout ce qui est spécifique au SQL. S'il parlait à Keycloak, tu mockais `IAuthenticationService`. S'il parlait à MinIO, tu mockais `IAmazonS3`. S'il parlait à RabbitMQ, tu mockais `IBus`. Les suites étaient étiquetées "tests d'intégration" et étaient, en pratique, **de faux tests d'intégration** : elles exerçaient ton code contre une fiction que tu avais écrite toi-même. Le jour où la vraie dépendance se comportait différemment, les tests restaient silencieux.

TestContainers démonte les cinq points d'un coup. Il remplace l'infrastructure partagée par des containers provisionnés par run, supprime la dépendance de la CI à des services externes (l'agent n'a besoin que de Docker), rend le setup reproductible depuis le code au lieu d'une page de wiki, déplace le cleanup de "la discipline des développeurs" vers "la disposition des containers", et, surtout, supprime la dernière excuse pour mocker une dépendance qui a une image Docker : Postgres avec `pg_trgm`, Keycloak avec un realm complet, MinIO pour S3, RabbitMQ, Kafka, Mongo, Elasticsearch. Si l'outil a une image, tu testes contre la vraie chose.

Le reste de cet article explique comment faire cela proprement.

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

> ✅ **Bonne pratique** : Fixe le tag de l'image (`postgres:17-alpine`, pas `postgres:latest`). La reproductibilité est l'objectif même. Un `latest` non fixé qui se déplace sous toi invalide silencieusement tous les runs qui l'ont précédé.

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

> ❌ **Ne jamais faire** : Ne pointe pas tes tests d'intégration vers une base de dev partagée. Deux devs qui lancent la suite en parallèle corrompent l'état l'un de l'autre, et l'échec ressemble à un test flaky plutôt qu'à une contention sur une ressource commune. TestContainers supprime directement la cause.

## Zoom : les scénarios que tu ne pouvais pas tester avant

C'est là que TestContainers gagne sa place. Trois exemples concrets de choses qui étaient quasiment impossibles (ou qui te coûtaient une semaine de YAML) avant, et qui tiennent maintenant dans une fixture.

### Comportement spécifique à Postgres : recherche floue avec pg_trgm

Tu as un endpoint de recherche qui trouve des clients par nom approximatif avec l'extension `pg_trgm`. Aucun mock sur terre ne reproduit le ranking de `similarity()`. La seule façon de le tester, c'est contre un vrai Postgres.

```csharp
public sealed class SearchFixture : IAsyncLifetime
{
    public PostgreSqlContainer Postgres { get; } = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .WithDatabase("search_test")
        .Build();

    public async ValueTask InitializeAsync()
    {
        await Postgres.StartAsync();
        await using var db = CreateDbContext();
        await db.Database.MigrateAsync();
        // Active l'extension, crée l'index GIN, seed les données de test.
        await db.Database.ExecuteSqlRawAsync("CREATE EXTENSION IF NOT EXISTS pg_trgm");
        await db.Database.ExecuteSqlRawAsync(
            "CREATE INDEX idx_customers_name_trgm ON customers USING gin (name gin_trgm_ops)");
        db.Customers.AddRange(
            new Customer("Jean Dupont"),
            new Customer("Jeanne Dupond"),
            new Customer("John Doe"));
        await db.SaveChangesAsync();
    }

    public ShopDbContext CreateDbContext() => new(new DbContextOptionsBuilder<ShopDbContext>()
        .UseNpgsql(Postgres.GetConnectionString()).Options);

    public ValueTask DisposeAsync() => Postgres.DisposeAsync();
}

[Fact]
public async Task Search_renvoie_les_matches_flous_classes_par_similarite()
{
    await using var db = _fixture.CreateDbContext();
    var repo = new CustomerRepository(db);

    var hits = await repo.SearchAsync("Jen Dupon", limit: 5);

    hits.Should().HaveCountGreaterThan(0);
    hits[0].Name.Should().BeOneOf("Jean Dupont", "Jeanne Dupond");
}
```

Le test prouve que l'extension est installée, que l'index est utilisé, et que le SQL que tu as écrit classe les résultats comme un vrai utilisateur l'attend. Un mock du repository ne validerait rien de tout cela, parce que le comportement sous test vit à l'intérieur de Postgres, pas dans ton code C#.

### Keycloak avec un vrai realm, utilisateurs, rôles et clients

L'autorisation par rôle, c'est notoirement pénible à tester. "Est-ce que `/admin/users` refuse un non-admin ?" demandait autrefois un Keycloak partagé, un realm curé à la main, et une convention que personne ne documentait. Avec TestContainers, tu importes un JSON de realm au démarrage du container, et tu obtiens tout : utilisateurs, mots de passe, rôles, clients, client scopes, mappers.

```csharp
public sealed class KeycloakFixture : IAsyncLifetime
{
    public IContainer Keycloak { get; } = new ContainerBuilder()
        .WithImage("quay.io/keycloak/keycloak:26.0")
        .WithPortBinding(8080, true)
        .WithEnvironment("KC_BOOTSTRAP_ADMIN_USERNAME", "admin")
        .WithEnvironment("KC_BOOTSTRAP_ADMIN_PASSWORD", "admin")
        .WithResourceMapping(
            new FileInfo("test-realm.json"),
            "/opt/keycloak/data/import/test-realm.json")
        .WithCommand("start-dev", "--import-realm")
        .WithWaitStrategy(Wait.ForUnixContainer()
            .UntilHttpRequestIsSucceeded(r => r.ForPath("/realms/test").ForPort(8080)))
        .Build();

    public string BaseUrl =>
        $"http://{Keycloak.Hostname}:{Keycloak.GetMappedPublicPort(8080)}";

    public ValueTask InitializeAsync() => new(Keycloak.StartAsync());
    public ValueTask DisposeAsync() => Keycloak.DisposeAsync();
}
```

`test-realm.json` vit à côté de la fixture. Il contient `alice` (rôle `admin`), `bob` (rôle `user`), un client confidentiel, des scopes, tout ce que ton realm de prod a, figé comme donnée de test. Chaque run obtient un Keycloak propre avec exactement le même état.

```csharp
[Fact]
public async Task Endpoint_admin_refuse_un_utilisateur_non_admin()
{
    var token = await GetTokenAsync("bob", "bob-password"); // utilisateur simple
    _client.DefaultRequestHeaders.Authorization = new("Bearer", token);

    var response = await _client.GetAsync("/admin/users");

    response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}
```

Le test passe par un vrai Keycloak, un vrai JWT, le vrai pipeline d'autorisation ASP.NET Core, contre la vraie policy. Rien n'est mocké. Quand ton mapping de rôles change en prod, ce test te le dit avant le deploy.

### MinIO pour du stockage compatible S3

Ton code utilise `AmazonS3Client` pour uploader des factures, générer des URLs présignées, et poser des policies de bucket. Tu veux vérifier que l'URL présignée télécharge vraiment le fichier, et qu'elle expire quand elle doit.

```csharp
public sealed class MinioFixture : IAsyncLifetime
{
    public IContainer Minio { get; } = new ContainerBuilder()
        .WithImage("minio/minio:latest")
        .WithPortBinding(9000, true)
        .WithEnvironment("MINIO_ROOT_USER", "minioadmin")
        .WithEnvironment("MINIO_ROOT_PASSWORD", "minioadmin")
        .WithCommand("server", "/data")
        .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(9000))
        .Build();

    public AmazonS3Client CreateClient() => new(
        new BasicAWSCredentials("minioadmin", "minioadmin"),
        new AmazonS3Config
        {
            ServiceURL = $"http://{Minio.Hostname}:{Minio.GetMappedPublicPort(9000)}",
            ForcePathStyle = true,
        });

    public ValueTask InitializeAsync() => new(Minio.StartAsync());
    public ValueTask DisposeAsync() => Minio.DisposeAsync();
}
```

À partir de là, tu testes de vrais uploads multipart, de vraies URLs présignées, un vrai comportement d'expiration. Le même code client tourne en prod contre AWS S3, et en test contre MinIO, parce que les deux parlent le protocole S3.

> 💡 **Info** : Le pattern se généralise. Si un outil a une image Docker officielle, tu peux le piloter depuis une fixture : RabbitMQ, Kafka, Mongo, Elasticsearch, Vault, Mailhog. Les packages NuGet `Testcontainers.*` fournissent des builders pré-construits pour les plus courants, et `ContainerBuilder` s'occupe du reste.

## Zoom : composer plusieurs services

Les vraies applis ont besoin de plus d'une dépendance. Postgres + Keycloak + MinIO + Redis, c'est une forme classique. Compose-les dans une seule fixture et démarre-les en parallèle :

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
