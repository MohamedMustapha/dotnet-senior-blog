---
title: "Tests API avec WebApplicationFactory en ASP.NET Core"
date: 2026-04-08
draft: false
tags: ["testing", "api", "webapplicationfactory", "dotnet"]
categories: ["Testing"]
series: ["Tests"]
description: "Teste ton vrai pipeline ASP.NET Core de bout en bout, in-process, sans Kestrel ni port HTTP. WebApplicationFactory se place entre les tests unitaires et le E2E, et pour la plupart des besoins de test d'API, c'est là que le meilleur rapport effort/valeur se trouve."
---

Hello tous le monde, aujourd'hui on va comprendre les **tests API avec WebApplicationFactory** en ASP.NET Core.

Entre [les tests unitaires](/fr/posts/testing-unit-testing/) et les [tests end-to-end](/fr/posts/testing-e2e-playwright/) pilotés par un vrai navigateur, il y a une couche intermédiaire hyper productive : des tests qui démarrent ton *vrai* pipeline ASP.NET Core (routing, model binding, middlewares, filters, DI, authentification) dans le même process que le test, et le pilotent via un `HttpClient` in-memory. Pas de Kestrel, pas de socket, pas de navigateur. Juste ton appli, qui tourne pour de vrai, en quelques millisecondes.

`WebApplicationFactory<TEntryPoint>` a été introduit dans ASP.NET Core 2.1 en 2018, dans le package `Microsoft.AspNetCore.Mvc.Testing`. Il a remplacé dix ans de solutions bricolées (`TestServer` à la main, host builders custom, bidouilles de Startup) par une primitive propre. Avec l'arrivée des minimal APIs et du `Program.cs` top-level en .NET 6, l'histoire est devenue encore plus simple. Si tu as lu l'article précédent sur [les tests d'intégration avec TestContainers](/fr/posts/testing-integration-testing-testcontainers/), tu as déjà la partie base de données. Cet article couvre la partie HTTP par-dessus.

## Le contexte : pourquoi ce pattern existe

Supposons que nous ayons une équipe dont l'API expose 40 endpoints. Ils ont de très bons tests unitaires sur les handlers et les repositories, mais les bugs qu'ils retrouvent régulièrement en prod, ce sont :

- Un filter qui a accidentellement sauté l'autorisation sur une route.
- Un model binder qui a silencieusement converti un enum null en `0`.
- Une réponse Problem Details dont la forme a changé après un upgrade de middleware.
- Une collision de route entre `/orders/{id}` et `/orders/export`.

Aucun de ces bugs ne vit dans une seule classe. Ils vivent dans le *pipeline* : l'interaction entre routing, filters, DI et sérialisation. Les tests unitaires ne peuvent pas les voir. Lancer l'appli en CI et la curler, ça marche mais c'est lent et fragile. Ce qu'il faut vraiment à cette équipe :

1. **Le vrai pipeline**, pas une simulation, pour que le routing et les filters se comportent comme en prod.
2. **Un démarrage rapide**, pour qu'un run de tests couvre 200 endpoints en moins d'une minute.
3. **Des hooks pour surcharger les services**, pour qu'un test puisse remplacer une dépendance (le payment gateway, l'horloge) sans toucher au code de production.

`WebApplicationFactory` te donne les trois.

## Vue d'ensemble : comment ça se branche

{{< mermaid >}}
graph TD
    A[Test] --> B[WebApplicationFactory&lt;Program&gt;]
    B --> C[TestServer<br/>in-memory]
    C --> D[Ton Program.cs<br/>DI, middlewares, endpoints]
    D --> E[HttpClient]
    A --> E
    D --> F[(Postgres depuis TestContainers)]
{{< /mermaid >}}

La factory boot ton `Program.cs` avec un `TestServer` au lieu de Kestrel. Le `HttpClient` qu'elle te refile parle au pipeline directement, en court-circuitant le réseau. Tout ce qui compte (routing, filters, auth, sérialisation) tourne pour de vrai.

> 💡 **Info** : `WebApplicationFactory<TEntryPoint>` prend un argument générique qui pointe vers un type de ton assembly de démarrage. La convention, c'est `WebApplicationFactory<Program>`. Si tu utilises les top-level statements, il faut ajouter `public partial class Program { }` en bas de ton `Program.cs` pour que le projet de test puisse référencer le type.

## Zoom : le test minimum

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Json;
using Xunit;

public class OrderEndpointsTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderEndpointsTests(WebApplicationFactory<Program> factory)
        => _client = factory.CreateClient();

    [Fact]
    public async Task GET_orders_renvoie_200_avec_la_liste()
    {
        var response = await _client.GetAsync("/orders");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var orders = await response.Content.ReadFromJsonAsync<List<OrderDto>>();
        orders.Should().NotBeNull();
    }
}
```

Cinq lignes de setup, le reste c'est une vraie assertion HTTP. `factory.CreateClient()` te rend un `HttpClient` déjà branché au test server. Pas de port, pas de hostname, pas de vrai socket.

> ✅ **Bonne pratique** : Asserte sur les status codes et les bodies de réponse, pas sur l'état interne. Un bon test d'API doit pouvoir être remplacé par une commande curl qui prouve le même comportement. Le couplage interne rend les tests fragiles.

## Zoom : surcharger des services

La fonctionnalité la plus précieuse de `WebApplicationFactory`, c'est `ConfigureWebHost`, où tu peux remplacer n'importe quel service enregistré en prod. Le gateway de paiement Stripe ? Tu le remplaces par un fake. L'horloge système ? Tu injectes un `FakeTimeProvider`.

```csharp
public class TestAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<IPaymentGateway>();
            services.AddSingleton<IPaymentGateway, FakePaymentGateway>();

            services.RemoveAll<TimeProvider>();
            services.AddSingleton<TimeProvider>(new FakeTimeProvider(
                DateTimeOffset.Parse("2026-04-08T12:00:00Z")));
        });
    }
}

public sealed class FakePaymentGateway : IPaymentGateway
{
    public Task<ChargeResult> ChargeAsync(CustomerId c, Money m, CancellationToken ct)
        => Task.FromResult(new ChargeResult(Success: true));
}
```

Puis dans les tests :

```csharp
public class SubmitOrderTests : IClassFixture<TestAppFactory>
{
    private readonly TestAppFactory _factory;
    public SubmitOrderTests(TestAppFactory factory) => _factory = factory;

    [Fact]
    public async Task POST_submit_debite_et_renvoie_204()
    {
        var client = _factory.CreateClient();
        var response = await client.PostAsync($"/orders/{Guid.NewGuid()}/submit", null);

        response.StatusCode.Should().Be(HttpStatusCode.NoContent);
    }
}
```

> 💡 **Info** : `services.RemoveAll<T>()` vient de `Microsoft.Extensions.DependencyInjection.Extensions`. C'est la façon idiomatique de surcharger un enregistrement, plutôt que d'en ajouter un second.

> ❌ **Ne jamais faire** : N'utilise pas `Mock.Setup(...)` pour simuler du comportement dans `ConfigureServices`. Les mocks, c'est pour les tests unitaires. Pour les tests d'intégration, une petite classe `Fake*` écrite à la main se lit mieux et survit mieux aux refactorings.

## Zoom : combiner avec TestContainers

La vraie puissance, c'est de combiner `WebApplicationFactory` avec un container Postgres. Tes tests pilotent le vrai pipeline contre la vraie base. C'est là que se cachent 80% des bugs.

```csharp
public class ApiWithDbFixture : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine").Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<DbContextOptions<ShopDbContext>>();
            services.AddDbContext<ShopDbContext>(o =>
                o.UseNpgsql(_db.GetConnectionString()));
        });
    }

    public async ValueTask InitializeAsync()
    {
        await _db.StartAsync();
        using var scope = Services.CreateScope();
        var ctx = scope.ServiceProvider.GetRequiredService<ShopDbContext>();
        await ctx.Database.MigrateAsync();
    }

    public new async ValueTask DisposeAsync()
    {
        await _db.DisposeAsync();
        await base.DisposeAsync();
    }
}
```

Une fixture. Le vrai pipeline. La vraie base. Des tests qui POST une commande, commit une ligne, et vérifient que l'endpoint GET la renvoie. Si un mapping EF Core est faux, qu'un filter oublie de tourner, ou qu'un middleware bousille le body de réponse, ce genre de test l'attrape.

> ✅ **Bonne pratique** : Seed tes données de test via l'API quand c'est possible, pas en insérant des lignes directement en base. Un test qui fait `POST /orders` puis `GET /orders/{id}` prouve que tout le flux marche de bout en bout. Un test qui court-circuite l'API ne prouve que les morceaux que tu as pensé à exercer.

## Zoom : l'authentification dans les tests

Les vraies APIs sont protégées. Tu as deux options propres :

**1. Test authentication handler** : enregistre un schéma fake qui authentifie chaque requête comme un utilisateur de test.

```csharp
services.AddAuthentication("Test")
    .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });

services.PostConfigure<AuthenticationOptions>(o =>
{
    o.DefaultAuthenticateScheme = "Test";
    o.DefaultChallengeScheme = "Test";
});
```

Le `TestAuthHandler` se contente de construire un `ClaimsPrincipal` à partir d'un utilisateur de test configuré. Simple, rapide, déterministe.

**2. Vrai flux JWT** : le test appelle ton vrai endpoint `/token` avec un compte de test, récupère le token, l'attache au `HttpClient`. Plus lent, mais ça prouve que le flux d'auth marche aussi.

> ⚠️ **Ça marche, mais...** : L'option 1 est ok pour la plupart des tests, mais garde au moins un test par route protégée qui passe par l'option 2. Sinon, le jour où ta vraie config JWT casse, aucun test ne le verra.

## Quand ne pas l'utiliser

`WebApplicationFactory`, c'est top, mais ça reste in-process. Si ton système de prod dépend de comportements qui n'apparaissent qu'avec du vrai réseau, plusieurs process, ou un vrai reverse proxy (sticky sessions, SignalR sur WebSockets via Nginx, auth par certificat client), il te faudra un vrai setup E2E par-dessus. C'est le sujet de l'article suivant.

## Wrap-up

Tu sais maintenant piloter ton vrai pipeline ASP.NET Core depuis des tests : créer une `WebApplicationFactory<Program>`, utiliser `ConfigureWebHost` pour remplacer les dépendances de prod par des fakes, la combiner avec un container Postgres pour une couverture d'intégration complète, et ajouter un handler d'authentification de test pour que tes routes protégées soient joignables. Tu peux écrire des tests qui postent une commande et vérifient l'état via une seconde requête, prouvant que tout le pipeline du routing à la persistence marche de bout en bout.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles](/fr/posts/testing-unit-testing/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)

## Références

- [Tests d'intégration dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/test/integration-tests)
- [WebApplicationFactory&lt;TEntryPoint&gt;, documentation API](https://learn.microsoft.com/fr-fr/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1)
- [Test d'authentification dans ASP.NET Core, Microsoft Learn](https://learn.microsoft.com/fr-fr/aspnet/core/test/integration-tests#mock-authentication)
