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

Les tests unitaires sont les tests les moins chers à écrire, et les premiers à se dégrader quand personne ne les entretient. Une suite de tests qui casse au moindre renommage de variable, qui mocke tout jusqu'à en perdre son sens, et qui met quatre secondes pour une seule assertion, apporte moins de valeur que pas de tests du tout. L'objectif de cet article est d'aider à écrire l'autre catégorie : des tests rapides, ciblés, et sur lesquels on peut réellement s'appuyer pendant un refactoring.

L'histoire du testing .NET est mature. xUnit.net a été lancé par James Newkirk en 2007, après qu'il ait co-créé NUnit, comme une réécriture qui nettoyait dix ans d'habitudes accumulées. C'est devenu le framework de test par défaut dans les templates ASP.NET Core vers 2016, et .NET 10 livre xUnit v3 comme version majeure courante. Autour, FluentAssertions (pour des asserts lisibles), NSubstitute ou Moq (pour les mocks), et Bogus (pour générer des données de test) composent la boîte à outils standard.

## Le contexte : pourquoi les tests unitaires existent

Les tests unitaires répondent à quatre problèmes concrets qu'aucune autre couche de la pyramide de tests ne traite aussi efficacement.

**1. Ils protègent contre la régression dans le temps.** C'est leur raison d'être principale. Une équipe livre un moteur de pricing au mois 1. Au mois 2, un palier de remise par volume est ajouté. Au mois 4, un multiplicateur de fidélité interagit avec. Au mois 9, un nouvel arrivant refactore un helper et casse, sans s'en rendre compte, l'interaction entre les paliers et la fidélité. Sans tests unitaires, la régression atteint la prod, et l'équipe la découvre par une réclamation client. Avec des tests unitaires, le changement ne se merge jamais : le test qui fixait le comportement du mois 2 échoue en moins d'une seconde, sur le poste de la personne qui a fait le changement.

**2. Ils détectent tôt les god classes et les god methods.** Une méthode difficile à tester en unitaire n'a presque jamais un problème de test, elle a un problème de conception. Quand un seul test demande quinze mocks, quatre pages d'arrange et une dizaine d'assertions pour couvrir un seul appel, le test nous dit que la méthode sous test fait trop de choses à la fois. La bonne réponse n'est pas d'écrire le test géant. C'est de découper la méthode. Les tests unitaires agissent comme un système d'alerte précoce sur les god classes et les god methods, bien avant qu'un rapport de qualité de code ne les signale.

**3. Ils testent la logique des méthodes, et rien d'autre.** Les tests unitaires sont le bon outil quand la question est "est-ce que ce morceau de logique calcule le bon résultat pour un input donné". Les requêtes base de données, les pipelines HTTP, les middlewares, la sérialisation, l'authentification : ce n'est pas de la logique de méthode, ce sont des comportements d'infrastructure. Ils relèvent des tests d'intégration, des tests d'API ou du E2E, pas d'ici. Garder le périmètre à la logique pure est ce qui rend les tests unitaires rapides, stables et dignes de confiance.

**4. La cerise sur le gâteau pour les conceptions orientées domaine.** Quand la logique métier vit dans un agrégat bien conçu et non-anémique (c'est-à-dire une entité qui fait respecter ses propres invariants au lieu d'exposer des setters publics manipulés par un service), les tests unitaires deviennent exceptionnellement propres. L'agrégat contient les règles, les tests contiennent les scénarios, et il n'y a rien d'autre à câbler. C'est l'argument le plus fort pour garder la logique à l'intérieur du domaine plutôt que de la disperser entre services, mappers et validators. Un article dédié au DDD et aux agrégats reviendra sur ce point en profondeur.

## Que faut-il tester

Un défaut raisonnable pour toute méthode qui contient de la vraie logique : **le chemin passant, les cas limites, et les cas d'échec**. Ces trois catégories couvrent la quasi-totalité des bugs qui valent la peine d'être attrapés.

- **Le chemin passant** : l'exécution normale et réussie avec des inputs valides. Un test par méthode, minimum.
- **Les cas limites** : les frontières où le comportement bascule. Une quantité de 0, de 1, exactement le seuil de remise, une collection vide, un champ optionnel null, la valeur maximale autorisée, le premier jour d'un mois, une année bissextile.
- **Les cas d'échec** : ce qui se passe quand un invariant est violé. Une quantité négative, une commande déjà soumise qu'on tente de soumettre à nouveau, un remboursement qui dépasse le montant initial. Le test vérifie que la bonne exception (ou le bon `Result.Failure`) revient, pas un état à moitié corrompu.

Au-delà de ce socle, deux règles supplémentaires gagnent leur place dans toute équipe sérieuse.

**Chaque bug en prod devrait ajouter un test.** Quand un bug est trouvé en prod, la correction est incomplète tant qu'un test capable de l'attraper n'existe pas. C'est le seul moyen durable pour que la protection contre la régression s'accumule. Une suite de tests qui grandit d'un test par incident devient, au fil des années, une cartographie de tout ce qui a déjà mal tourné, et l'équipe hérite de cette connaissance gratuitement.

**Les garde-fous et l'autorisation méritent leurs propres tests.** La programmation défensive n'est pas complète tant qu'elle n'est pas vérifiée. Pour chaque opération sensible au rôle, il faut écrire la paire : "en tant qu'admin, l'action est autorisée" et "en tant qu'utilisateur simple, l'action est refusée". Même chose pour l'isolation de tenant, les vérifications de propriétaire, et les limites de débit. Ce sont les règles qui sont cassées en silence lors d'un refactoring et découvertes lors d'un audit.

**Pour les applications principalement CRUD**, la même catégorisation s'applique, mais l'accent change :

- **Les opérations d'écriture** avec logique métier : règles de validation, invariants croisés, transitions d'état. À tester au niveau unitaire, avec l'agrégat ou le service comme SUT.
- **Les opérations de lecture** avec transformation : projection d'entité vers DTO, agrégation, champs calculés, formatage. Tester la transformation elle-même.
- **Le CRUD pur passe-plat** (controller vers repository vers base, sans logique au milieu) n'a pas besoin de test unitaire. Il a besoin d'un test d'intégration qui prouve que l'aller-retour fonctionne.

Les tests unitaires couvrent tout ce qui précède, à condition de rester correctement scopés. Dès qu'un "test unitaire" démarre une base de données, un hôte web ou le système de fichiers, ce n'est plus un test unitaire, c'est un test d'intégration lent. C'est un autre outil, pour un autre rôle.

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

Le **SUT** est la classe sous test. Tout le reste est de l'échafaudage. L'objectif est de garder le ratio élevé : un minimum d'échafaudage, un maximum de SUT.

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

> 💡 **Info** : L'attribut `[Fact]` marque un test sans paramètre. Pour plusieurs inputs, on utilise `[Theory]` avec `[InlineData]` ou `[MemberData]`. Ce n'est pas du sucre syntaxique, c'est précisément ce pour quoi les tests paramétrés existent.

> ✅ **Bonne pratique** : Nommer les tests en `Methode_etat_resultat` ou en phrases claires comme `applique_une_remise_de_volume_au_dessus_de_10_articles`. La sortie du runner de test fait office de documentation pour le futur lecteur.

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

Le mocking, c'est la technique la plus souvent mal employée dans les tests. La règle est simple : mocke les **frontières**, pas le **comportement**. Une frontière, c'est une interface vers laquelle le SUT appelle (repository, client HTTP, provider de temps). Tout le reste doit être réel.

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

> ⚠️ **Ça marche, mais...** : Lorsque nous nous retrouvons à mocker nos propres classes de domaine (`Order`, `Invoice`, `Customer`), il faut prendre du recul. Soit la classe est une frontière déguisée (extraire une interface), soit le test vérifie le mock plutôt que le SUT.

> ❌ **Ne jamais faire** : Ne pas écrire de tests qui assertent `mock.Received(1).HelperInterne()`. Ce type de test fige l'implémentation, pas le comportement. Un refactoring qui garde le même contrat public casserait ces tests sans raison.

## Zoom : ce qu'il ne faut pas tester en unitaire

Les tests unitaires sont le mauvais outil pour :

- **Les requêtes base de données** : une expression EF Core `Where` n'est pas testable de façon utile en unitaire. Le bug se cache dans le SQL généré, et se vérifie avec une vraie base via les tests d'intégration.
- **Le pipeline HTTP, les middlewares, les filters** : se testent en démarrant le vrai pipeline avec `WebApplicationFactory`.
- **Les aller-retours de sérialisation** : à vérifier sur le JSON réel en end-to-end, pas sur un mock.
- **Le rendu UI** : tester un composant Blazor pour son layout est le mauvais niveau. Le E2E Playwright attrape les vrais bugs.

Si un test prend plus de 50 ms à tourner, ce n'est probablement pas un test unitaire. Ce n'est pas un problème en soi : il a simplement sa place dans un autre projet de test, avec un autre cycle de vie.

> ✅ **Bonne pratique** : Découper la solution en `MyApp.UnitTests`, `MyApp.IntegrationTests`, `MyApp.E2ETests`. La CI peut alors lancer les tests unitaires à chaque commit et les suites plus lentes moins souvent, ou en étages parallèles.

## Tourner vite et en parallèle

xUnit v3 exécute les classes de test en parallèle par défaut. C'est précieux, à condition que les tests ne partagent pas d'état statique (singletons cachés, `DateTime.Now`, variables d'environnement). Deux règles :

1. **Aucun état mutable partagé** entre les tests. Chaque test arrange son propre état.
2. **Injecter une horloge** au lieu d'appeler `DateTime.UtcNow` directement. Depuis .NET 8, `TimeProvider` est l'abstraction canonique.

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

`FakeTimeProvider` vit dans le package NuGet `Microsoft.Extensions.TimeProvider.Testing`. Fini le `DateTime.UtcNow` directement dans le code de production.

> 💡 **Info** : `TimeProvider` a été introduit dans .NET 8. Avant, les équipes maintenaient leur propre interface `IClock`. Sur .NET 6/7, cette abstraction maison reste valable, le pattern de test est identique.

## Wrap-up

Tu sais maintenant écrire des tests unitaires qui sont réellement utiles : scopés sur un seul comportement, structurés en AAA, en mockant uniquement les frontières, qui tournent en millisecondes, et qui survivent aux refactorings sans tout réécrire. Tu peux choisir xUnit v3 + FluentAssertions + NSubstitute comme défaut fiable, utiliser `[Theory]` pour les tables d'inputs, injecter `TimeProvider` au lieu de taper l'horloge système, et reconnaître les cas où un test unitaire n'est pas le bon outil.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Références

- [Documentation xUnit.net](https://xunit.net/)
- [Tests unitaires en .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/core/testing/)
- [TimeProvider en .NET, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/standard/datetime/timeprovider-overview)
- [Documentation NSubstitute](https://nsubstitute.github.io/)
- [Documentation FluentAssertions](https://fluentassertions.com/)
