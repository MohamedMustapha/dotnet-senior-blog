---
title: "Tests End-to-End avec Playwright pour .NET"
date: 2026-04-08
draft: false
tags: ["testing", "e2e", "playwright", "dotnet"]
categories: ["Testing"]
series: ["Tests"]
description: "Des tests end-to-end pilotés par un vrai navigateur en .NET avec Playwright : des sélecteurs fiables, de l'exécution parallèle, de l'auto-wait, et la discipline qui garde la suite verte."
---

Hello tous le monde, aujourd'hui on va découvrir les **tests end-to-end avec Playwright** pour .NET.

Les tests end-to-end ont une mauvaise réputation bien méritée. Dix ans de suites Selenium flakys, des implicit waits qui n'attendent jamais tout à fait assez, des sélecteurs XPath qui cassent à chaque refresh d'UI, et des runs de CI qui plantent "parfois" ont convaincu beaucoup d'équipes que le E2E ne valait pas le coup. Ils avaient raison à propos de Selenium. Ils avaient tort à propos du E2E.

Playwright a changé la donne. Microsoft l'a publié en 2020 comme successeur moderne de Puppeteer, et les bindings .NET ont suivi début 2021. Il embarque Chromium, Firefox et WebKit, fait de l'auto-wait sur les éléments avant d'agir, isole chaque test dans un contexte de navigateur frais, et livre un générateur de code qui enregistre tes actions en fichier de test. Si tu as lu les articles précédents sur [les tests unitaires](/fr/posts/testing-unit-testing/), [les tests d'intégration avec TestContainers](/fr/posts/testing-integration-testing-testcontainers/), et [les tests API avec WebApplicationFactory](/fr/posts/testing-webapplicationfactory/), tu as déjà les couches rapides, peu chères, in-process. Playwright, c'est le sommet de la pyramide : plus lent, mais la seule chose qui prouve que ton appli marche vraiment comme un utilisateur va l'utiliser.

## Le contexte : pourquoi ce pattern existe

Supposons que nous ayons une équipe qui livre un tunnel de checkout. Les tests unitaires sont verts. Les tests d'API sont verts. Une QA manuelle découvre que le bouton "Payer" est désactivé quand le formulaire a des erreurs de validation, donc les utilisateurs ne voient jamais le message d'erreur que l'API renvoie. Une seconde passe découvre une race condition où le total du panier se met à jour après que le handler de soumission se soit déclenché, donc les utilisateurs paient l'ancien prix. Aucun des deux bugs n'est rattrapable à une couche inférieure au navigateur. Les deux partent en prod.

Ce qu'il faut vraiment à cette équipe :

1. **Un vrai navigateur** qui rend la vraie page, pour que les événements DOM, le CSS et le JavaScript se comportent exactement comme un utilisateur les voit.
2. **Des sélecteurs fiables** qui survivent à des changements mineurs d'UI, pour qu'un reorder de div ne casse pas 40 tests.
3. **Une exécution parallèle rapide** et isolée, pour que la suite tourne en minutes, pas en heures, et qu'un test flaky n'empoisonne pas les autres.

Playwright livre les trois.

## Vue d'ensemble : les pièces

{{< mermaid >}}
graph TD
    A[Test Playwright] --> B[Microsoft.Playwright.NUnit<br/>ou wrapper MSTest / xUnit]
    B --> C[Navigateur<br/>Chromium / Firefox / WebKit]
    C --> D[Ton appli qui tourne<br/>Kestrel sur localhost:5000]
    D --> E[(Postgres depuis TestContainers)]
    A --> F[Page Object<br/>CheckoutPage]
    F --> C
{{< /mermaid >}}

Le test pilote un vrai navigateur. Le navigateur parle à ton appli qui tourne, qui elle-même parle à une vraie base. Le pattern Page Object garde les sélecteurs à un seul endroit, pour que les refactorings d'UI touchent un fichier et pas toute la suite.

> 💡 **Info** : Playwright pour .NET livre son propre runner de tests via les packages `Microsoft.Playwright.NUnit` / `Microsoft.Playwright.MSTest`. Ils te donnent l'exécution parallèle, un contexte navigateur frais par test, et l'enregistrement de traces out of the box. Tu peux aussi utiliser `PlaywrightSharp` brut dans xUnit, mais l'adapteur NUnit est plus mature.

## Zoom : installation et premier test

Installe le package et les navigateurs en une étape :

```bash
dotnet add package Microsoft.Playwright.NUnit
dotnet build
pwsh bin/Debug/net10.0/playwright.ps1 install
```

Le script `install` télécharge Chromium, Firefox et WebKit (environ 500 Mo). Commit l'invocation dans ton script CI pour que les agents aient les navigateurs au premier run.

```csharp
using Microsoft.Playwright;
using Microsoft.Playwright.NUnit;
using NUnit.Framework;

[Parallelizable(ParallelScope.Self)]
public class CheckoutTests : PageTest
{
    [Test]
    public async Task User_peut_soumettre_une_commande_depuis_le_panier()
    {
        await Page.GotoAsync("http://localhost:5000");
        await Page.GetByRole(AriaRole.Link, new() { Name = "Catalogue" }).ClickAsync();
        await Page.GetByRole(AriaRole.Button, new() { Name = "Ajouter au panier" }).First.ClickAsync();
        await Page.GetByRole(AriaRole.Link, new() { Name = "Panier" }).ClickAsync();
        await Page.GetByRole(AriaRole.Button, new() { Name = "Valider" }).ClickAsync();

        await Expect(Page.GetByText("Commande confirmée")).ToBeVisibleAsync();
    }
}
```

`PageTest` te donne une `Page` fraîche par test et libère tout à la fin. Aucun boilerplate.

> ✅ **Bonne pratique** : Préfère `GetByRole`, `GetByLabel`, `GetByPlaceholder` et `GetByText` aux sélecteurs CSS ou XPath. Ils matchent la façon dont les utilisateurs et les technos d'assistance perçoivent la page, et ils survivent aux renommages de classes CSS.

## Zoom : locators et auto-wait

La killer feature de Playwright, c'est **l'auto-wait**. Chaque action (`ClickAsync`, `FillAsync`) attend que l'élément soit visible, activé et stable avant d'agir. Chaque assertion (`ToBeVisibleAsync`, `ToHaveTextAsync`) retry jusqu'à ce que la condition soit vraie ou qu'un timeout expire. Tu n'écris quasiment jamais d'attente explicite.

```csharp
// Playwright attend que le bouton existe, soit activé et visible.
await Page.GetByRole(AriaRole.Button, new() { Name = "Valider" }).ClickAsync();

// Playwright retry l'assertion jusqu'à 5 secondes par défaut.
await Expect(Page.GetByTestId("order-total")).ToHaveTextAsync("199,98 €");
```

Compare ça au monde Selenium où tu écrivais `Thread.Sleep(2000)` parce que l'élément chargeait depuis une API. Ces sleeps ont disparu.

> ❌ **Ne jamais faire** : N'ajoute pas de `Task.Delay` dans un test Playwright. Si un test échoue par intermittence, la réponse est presque toujours un meilleur sélecteur (utilise `getByTestId` sur un attribut stable) ou une meilleure assertion (laisse Playwright retry), pas un sleep plus long.

## Zoom : le pattern Page Object

Garde les sélecteurs en dehors des tests. Une classe par page ou composant, réutilisée dans plusieurs tests :

```csharp
public sealed class CheckoutPage
{
    private readonly IPage _page;
    public CheckoutPage(IPage page) => _page = page;

    public ILocator CheckoutButton =>
        _page.GetByRole(AriaRole.Button, new() { Name = "Valider" });

    public ILocator OrderTotal => _page.GetByTestId("order-total");
    public ILocator Confirmation => _page.GetByText("Commande confirmée");

    public Task GotoAsync() => _page.GotoAsync("/panier");
    public Task SubmitAsync() => CheckoutButton.ClickAsync();
}

[Test]
public async Task Checkout_affiche_la_confirmation()
{
    var checkout = new CheckoutPage(Page);
    await checkout.GotoAsync();
    await checkout.SubmitAsync();

    await Expect(checkout.Confirmation).ToBeVisibleAsync();
}
```

Quand le bouton "Valider" devient "Passer commande" au trimestre suivant, tu changes une ligne dans `CheckoutPage.cs` et 40 tests continuent de passer.

> 💡 **Info** : Ajoute des attributs `data-testid` dans tes composants Razor ou React pour les éléments sans nom accessible naturel. `GetByTestId("cart-line-1-qty")` est béton et survit à tout type de refactoring d'UI.

## Zoom : héberger l'appli sous test

Tu as deux options pour où "l'appli" tourne pendant le test :

**1. La démarrer in-process** : utilise `WebApplicationFactory` (couvert dans [l'article précédent](/fr/posts/testing-webapplicationfactory/)) pour lancer l'appli sur un vrai port Kestrel à l'intérieur du process de test.

```csharp
public sealed class AppFixture : IDisposable
{
    private readonly WebApplication _app;
    public string BaseUrl { get; }

    public AppFixture()
    {
        var builder = WebApplication.CreateBuilder();
        // ... mêmes services que Program.cs ...
        _app = builder.Build();
        _app.Urls.Add("http://127.0.0.1:0"); // port aléatoire libre
        _app.StartAsync().GetAwaiter().GetResult();
        BaseUrl = _app.Urls.First();
    }

    public void Dispose() => _app.StopAsync().GetAwaiter().GetResult();
}
```

Avantages : pas de dépendance externe, le test possède le cycle de vie. Inconvénient : il te faut du vrai Kestrel, pas le `TestServer` in-memory, parce que Playwright pilote un vrai navigateur qui a besoin d'un vrai socket.

**2. La lancer comme process séparé** : un job CI démarre `dotnet run` en arrière-plan, attend le health endpoint, puis lance la suite Playwright. Plus réaliste, plus proche de la prod, mais plus de pièces mobiles.

L'option 1 est le sweet spot pour la plupart des équipes.

> ⚠️ **Ça marche, mais...** : Associer Playwright à TestContainers pour la base fonctionne, et c'est le E2E le plus honnête que tu peux obtenir sans un vrai environnement de staging. Le compromis, c'est le temps de démarrage : un run à froid (téléchargement de Postgres et des binaires du navigateur) peut prendre 30 secondes. Un run à chaud est rapide.

## Zoom : traces, vidéos, debug

Quand un test échoue en CI, la trace viewer de Playwright vaut son pesant d'or. Active-la uniquement sur les tests qui plantent :

```csharp
[SetUp]
public async Task BeforeEach()
{
    await Context.Tracing.StartAsync(new() { Screenshots = true, Snapshots = true });
}

[TearDown]
public async Task AfterEach()
{
    var failed = TestContext.CurrentContext.Result.Outcome != ResultState.Success;
    await Context.Tracing.StopAsync(new()
    {
        Path = failed ? $"traces/{TestContext.CurrentContext.Test.Name}.zip" : null
    });
}
```

Upload le dossier `traces/` comme artefact CI. Ouvre le zip avec `pwsh bin/Debug/net10.0/playwright.ps1 show-trace trace.zip` et tu vois le snapshot DOM exact à chaque action, avec les appels réseau et les logs console.

> ✅ **Bonne pratique** : Garde la suite E2E petite et à haute valeur. Dix tests qui couvrent les flux critiques (inscription, ajout au panier, checkout, historique de commandes, annulation) valent mieux que cent tests qui cliquent partout sur des pages à faible valeur. La flakiness grandit avec la taille de la suite.

## Quand ne pas utiliser Playwright

Le E2E, c'est la couche la plus lente et la plus chère de ta pyramide de tests. Utilise-la pour :

- **Les parcours critiques visibles par l'utilisateur** qui ne doivent pas casser.
- **Les flux qui touchent plusieurs systèmes** (auth + API + base + UI).
- **Les régressions après un gros refactoring** du front ou de la topologie de déploiement.

Ne l'utilise *pas* pour :

- **Les règles métier** : elles vont dans les tests unitaires.
- **Les contrats d'API** : ils vont dans les tests `WebApplicationFactory`.
- **Le comportement base de données** : il va dans les tests d'intégration TestContainers.

Suis la pyramide : beaucoup de tests unitaires, moins de tests d'intégration, très peu de tests E2E. Le ratio garde la suite rapide et fiable.

## Wrap-up

Tu sais maintenant écrire des tests end-to-end qui survivent vraiment : installer Playwright et ses navigateurs, utiliser des locators basés sur les roles et les labels pour que tes tests matchent comment les utilisateurs et les outils d'accessibilité perçoivent la page, t'appuyer sur l'auto-wait au lieu de sleeps manuels, organiser les sélecteurs via le pattern Page Object, héberger l'appli sous test avec `WebApplicationFactory` sur un vrai port Kestrel, et capturer des traces pour les runs qui échouent en CI. Tu peux garder la suite tendue, concentrée sur les parcours critiques, et la faire tourner en minutes plutôt qu'en heures.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles](/fr/posts/testing-unit-testing/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)
- [Tests API avec WebApplicationFactory en ASP.NET Core](/fr/posts/testing-webapplicationfactory/)

## Références

- [Playwright pour .NET, documentation officielle](https://playwright.dev/dotnet/)
- [Guide des locators Playwright](https://playwright.dev/dotnet/docs/locators)
- [Auto-waiting Playwright](https://playwright.dev/dotnet/docs/actionability)
- [Trace viewer Playwright](https://playwright.dev/dotnet/docs/trace-viewer)
