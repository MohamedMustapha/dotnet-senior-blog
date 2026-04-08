---
title: "Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles"
date: 2026-04-08
draft: false
tags: ["testing", "dotnet", "xunit"]
categories: ["Testing"]
series: ["Tests"]
description: "Des tests unitaires qui attrapent de vrais bugs, tournent en millisecondes, et survivent aux refactorings. xUnit, le pattern AAA, le mocking et ses pièges, et ce qu'il ne faut jamais tester."
---

Hello tous le monde, aujourd'hui on va démystifier les **tests unitaires** en .NET.

Les tests unitaires, c'est les tests les moins chers à écrire, et les premiers à pourrir quand personne ne les entretient. Une suite de tests qui casse au moindre renommage de variable, qui mocke tout jusqu'à en perdre son sens, et qui met quatre secondes pour une seule assertion, apporte moins de valeur que pas de tests du tout. L'objectif de cet article, c'est de t'aider à écrire l'autre catégorie : rapides, ciblés, et sur lesquels tu peux vraiment t'appuyer pendant un refactoring.

L'histoire du testing .NET est mature. xUnit.net a été lancé par James Newkirk en 2007, après qu'il ait co-créé NUnit, comme une réécriture qui nettoyait dix ans d'habitudes accumulées. C'est devenu le framework de test par défaut dans les templates ASP.NET Core vers 2016, et .NET 10 livre xUnit v3 comme version majeure courante. Autour, FluentAssertions (pour des asserts lisibles), NSubstitute ou Moq (pour les mocks), et Bogus (pour générer des données de test) composent la boîte à outils standard.

## Le contexte : pourquoi les tests unitaires existent

Supposons que nous ayons une équipe qui livre un moteur de pricing. Les règles s'empilent : taxes régionales, remises par volume, multiplicateurs de fidélité, promotions. Au bout de six mois, personne n'ose plus toucher `PriceCalculator.Calculate()` parce qu'une ligne mal placée pourrait surfacturer des milliers de clients en silence. Chaque changement passe par trois jours de QA manuelle. La vélocité de livraison en prend un coup visible.

Ce qu'il faut vraiment à cette équipe :

1. **Un feedback rapide** : une barre verte en moins d'une seconde quand la logique est bonne.
2. **Une régression ciblée** : quand un changement casse la règle #7 précisément, le test en échec dit quelle règle et quel input.
3. **La confiance pour refactorer** : pouvoir restructurer l'intérieur du calculator sans réécrire la suite de tests.

Les tests unitaires apportent les trois, à condition d'être scopés correctement. Dès qu'un "test unitaire" démarre une base de données, un hôte web ou le système de fichiers, ce n'est plus un test unitaire, c'est un test d'intégration lent. C'est un autre outil, pour un autre job.

## Vue d'ensemble : les pièces

Avant de rentrer dans le code, voici les outils qu'une suite de tests unitaires .NET utilise vraiment en 2026 :

{{< mermaid >}}
graph TD
    A[Projet de test<br/>xUnit v3] --> B[Assertions<br/>FluentAssertions ou Shouldly]
    A --> C[Mocks / Fakes<br/>NSubstitute ou Moq]
    A --> D[Données de test<br/>Bogus, AutoFixture]
    A --> E[SUT<br/>System Under Test]
    B --> E
    C --> E
    D --> E
{{< /mermaid >}}

Le **SUT**, c'est la classe que tu testes. Tout le reste, c'est de l'échafaudage. Ton job, c'est de garder le ratio haut : un minimum d'échafaudage, un maximum de SUT.

## Zoom : le pattern AAA

Chaque bon test unitaire a trois sections : **Arrange**, **Act**, **Assert**. Séparées visuellement, elles se lisent comme du texte.

```csharp
using FluentAssertions;
using Xunit;

public class PriceCalculatorTests
{
    [Fact]
    public void Calculate_applique_une_remise_de_volume_au_dessus_de_10_articles()
    {
        // Arrange
        var calculator = new PriceCalculator();
        var order = new Order(
            customerId: CustomerId.New(),
            lines: [new OrderLine("SKU-42", quantity: 12, unitPrice: 10m)]);

        // Act
        var total = calculator.Calculate(order);

        // Assert
        total.Amount.Should().Be(108m); // 10% de remise au dessus de 10 articles
    }
}
```

Trois choses font que ce test est bon : le nom décrit le *comportement*, pas la méthode ; l'arrange est minimal ; l'assert vérifie un seul résultat. Si quelqu'un change les entrailles de `PriceCalculator` demain, ce test continue de passer tant que la règle tient.

> 💡 **Info** : L'attribut `[Fact]` marque un test sans paramètre. Pour plusieurs inputs, utilise `[Theory]` avec `[InlineData]` ou `[MemberData]`. Ce n'est pas du sucre syntaxique, c'est précisément ce pour quoi les tests paramétrés existent.

> ✅ **Bonne pratique** : Nomme tes tests en `Methode_etat_resultat` ou en phrases claires comme `applique_une_remise_de_volume_au_dessus_de_10_articles`. La sortie du runner de test, c'est de la documentation pour toi demain matin.

## Zoom : Theory pour les tables d'inputs

Quand une méthode a plusieurs branches d'entrée, une theory bat dix facts copiés-collés :

```csharp
[Theory]
[InlineData(1,  10.00,   0,  10.00)]   // aucune remise
[InlineData(10, 10.00,   0, 100.00)]   // pile au seuil
[InlineData(11, 10.00,  10,  99.00)]   // 10% de remise
[InlineData(50, 10.00,  15, 425.00)]   // palier 15%
public void Calculate_applique_une_remise_par_paliers(
    int quantity, decimal unitPrice, int expectedDiscountPct, decimal expectedTotal)
{
    var calculator = new PriceCalculator();
    var order = new Order(
        CustomerId.New(),
        [new OrderLine("SKU-1", quantity, unitPrice)]);

    var total = calculator.Calculate(order);

    total.Amount.Should().Be(expectedTotal);
}
```

Une seule méthode de test, quatre cas, quatre lignes dans le runner. Ajouter un nouveau palier, c'est une ligne.

## Zoom : mocker, mais prudemment

Le mocking, c'est la technique la plus souvent mal employée dans les tests. La règle est simple : mocke les **frontières**, pas le **comportement**. Une frontière, c'est une interface vers laquelle ton SUT appelle (repository, client HTTP, provider de temps). Tout le reste doit être réel.

```csharp
[Fact]
public async Task Submit_debite_le_client_et_marque_la_commande_soumise()
{
    // Arrange
    var payments = Substitute.For<IPaymentGateway>();
    payments.ChargeAsync(Arg.Any<CustomerId>(), Arg.Any<Money>(), Arg.Any<CancellationToken>())
        .Returns(new ChargeResult(Success: true));

    var repo = Substitute.For<IOrderRepository>();
    var order = Order.Create(CustomerId.New());
    order.AddLine(new ProductId(1), 2, new Money(50m));
    repo.GetByIdAsync(order.Id, Arg.Any<CancellationToken>()).Returns(order);

    var handler = new SubmitOrderHandler(repo, payments, new FakeUnitOfWork());

    // Act
    var result = await handler.Handle(new SubmitOrderCommand(order.Id.Value), default);

    // Assert
    result.IsSuccess.Should().BeTrue();
    order.Status.Should().Be(OrderStatus.Submitted);
    await payments.Received(1).ChargeAsync(order.CustomerId, order.Total, Arg.Any<CancellationToken>());
}
```

L'entité de domaine `Order` est **vraie**, pas mockée. Seuls `IPaymentGateway` et `IOrderRepository` sont substitués, parce qu'ils parlent au monde extérieur.

> ⚠️ **Ça marche, mais...** : Si tu te retrouves à mocker tes propres classes de domaine (`Order`, `Invoice`, `Customer`), prends du recul. Soit la classe est une frontière déguisée (extrais une interface), soit le test teste le mock, pas le SUT.

> ❌ **Ne jamais faire** : N'écris pas de tests qui assertent `mock.Received(1).HelperInterne()`. Tu fixes l'implémentation, pas le comportement. Un refactoring qui garde le même contrat public cassera tes tests pour rien.

## Zoom : ce qu'il ne faut pas tester en unitaire

Les tests unitaires sont le mauvais outil pour :

- **Les requêtes base de données** : une expression EF Core `Where` n'est pas testable de façon utile en unitaire. Le bug se cache dans le SQL généré. Teste ça avec une vraie base via les tests d'intégration.
- **Le pipeline HTTP, les middlewares, les filters** : démarre le vrai pipeline avec `WebApplicationFactory`.
- **Les aller-retours de sérialisation** : assert sur le JSON réel en end-to-end, pas sur un mock.
- **Le rendu UI** : tester un composant Blazor pour son layout, c'est le mauvais niveau. Le E2E Playwright attrape les vrais bugs.

Si un test prend plus de 50ms à tourner, ce n'est probablement pas un test unitaire. C'est pas grave, il a juste sa place dans un autre projet de test avec un autre cycle de vie.

> ✅ **Bonne pratique** : Découpe ta solution en `MyApp.UnitTests`, `MyApp.IntegrationTests`, `MyApp.E2ETests`. La CI peut lancer les tests unitaires à chaque commit et les suites plus lentes moins souvent, ou en étages parallèles.

## Tourner vite et en parallèle

xUnit v3 exécute les classes de test en parallèle par défaut. C'est génial, sauf si tes tests partagent de l'état statique (singletons cachés, `DateTime.Now`, variables d'environnement). Deux règles :

1. **Aucun état mutable partagé** entre les tests. Chaque test arrange son propre monde.
2. **Injecte une horloge** au lieu d'appeler `DateTime.UtcNow` directement. Depuis .NET 8, `TimeProvider` est l'abstraction canonique.

```csharp
public sealed class PromotionService(TimeProvider clock)
{
    public bool IsActive(Promotion p) => clock.GetUtcNow() < p.EndsAt;
}

// Dans les tests
var fakeClock = new FakeTimeProvider(DateTimeOffset.Parse("2026-04-08T12:00:00Z"));
var service = new PromotionService(fakeClock);
service.IsActive(new Promotion { EndsAt = DateTimeOffset.Parse("2026-04-09T00:00:00Z") })
    .Should().BeTrue();
```

`FakeTimeProvider` vit dans le package NuGet `Microsoft.Extensions.TimeProvider.Testing`. Fini le `DateTime.UtcNow` dans le code de production.

> 💡 **Info** : `TimeProvider` a été introduit dans .NET 8. Avant, les équipes roulaient leur propre interface `IClock`. Si tu es encore sur .NET 6/7, garde ta propre abstraction, le pattern de test est identique.

## Wrap-up

Tu sais maintenant écrire des tests unitaires qui gagnent vraiment leur place : scopés sur un seul comportement, avec le layout AAA, en mockant uniquement les frontières, qui tournent en millisecondes, et qui survivent aux refactorings sans tout réécrire. Tu peux choisir xUnit v3 + FluentAssertions + NSubstitute comme défaut safe, utiliser `[Theory]` pour les tables d'inputs, injecter `TimeProvider` au lieu de taper l'horloge système, et reconnaître les cas où un test unitaire n'est pas le bon outil.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Références

- [Documentation xUnit.net](https://xunit.net/)
- [Tests unitaires en .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/testing/)
- [TimeProvider en .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/standard/datetime/timeprovider-overview)
- [Documentation NSubstitute](https://nsubstitute.github.io/)
- [Documentation FluentAssertions](https://fluentassertions.com/)
