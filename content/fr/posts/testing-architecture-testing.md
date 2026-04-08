---
title: "Tests d'Architecture en .NET : les règles que le compilateur ne peut pas imposer"
date: 2026-04-08
draft: false
tags: ["testing", "architecture", "dotnet"]
categories: ["Testing"]
series: ["Tests"]
description: "Transforme tes décisions d'architecture en règles exécutables. NetArchTest et ArchUnitNET font échouer le build quand quelqu'un fait fuiter Infrastructure dans Domain, oublie un suffixe async ou saute une couche."
---

Hello tous le monde, aujourd'hui on va démystifier les **tests d'architecture** en .NET.

Chaque codebase .NET a des règles qui vivent uniquement dans les README, les docs d'onboarding, ou la mémoire tribale du dev senior. "Domain ne référence jamais Infrastructure." "Les handlers se terminent par `Handler`." "Pas de `using System.Data;` dans la couche Application." Le compilateur n'en vérifie aucune. Six mois plus tard, quelqu'un ajoute un `using Shop.Infrastructure;` dans une classe de domaine parce qu'IntelliSense l'a proposé, le build passe, et la règle meurt en silence.

Les tests d'architecture transforment ces règles en assertions exécutables. Ce sont des tests unitaires sur le graphe de tes assemblies : "assert qu'aucun type dans `Shop.Domain` ne dépend de `Shop.Infrastructure`", "assert que chaque handler se termine par `Handler`", "assert que chaque type dans `Application.Orders.Commands` implémente `IRequest`". La première bibliothèque largement utilisée a été ArchUnit sur la JVM (2017). Le port .NET, NetArchTest de Ben Morris, est sorti en 2018, suivi par ArchUnitNET en 2020 avec une API fluent plus riche. Les deux sont matures, open source, et marchent avec n'importe quel runner de test.

Si tu as suivi cette série depuis [les tests unitaires](/fr/posts/testing-unit-testing/) jusqu'aux [tests E2E avec Playwright](/fr/posts/testing-e2e-playwright/), tu as déjà couvert l'axe de la correction. Les tests d'architecture couvrent un autre axe : **la dérive structurelle dans le temps**.

## Le contexte : pourquoi ce pattern existe

Supposons que nous ayons une équipe qui a adopté la [Clean Architecture](/fr/posts/code-structure-clean-architecture/) il y a dix-huit mois. Au jour 1, les règles étaient écrites au tableau et tout le monde les connaissait. Au mois 3, un nouvel arrivant a ajouté `[Table("orders")]` sur une entité de domaine parce que "c'était plus rapide". Au mois 6, un raccourci pendant un incident a ajouté un `using Microsoft.EntityFrameworkCore;` dans `Domain/Orders/Order.cs`. Au mois 12, le projet "Clean Architecture" a l'air propre sur le diagramme et ressemble à du layered spaghetti dans le code.

Aucun de ces changements n'a causé de bug le jour où il a été mergé. Ils ont causé une dégradation lente. Ce qu'il faut vraiment à cette équipe :

1. **Des invariants exécutables** : un test qui fait échouer le build quand une règle est cassée, pas une ligne dans une checklist de review.
2. **Une seule source de vérité** : la règle vit dans le code, à côté des tests, lisible par chaque ingénieur.
3. **Peu de cérémonial** : écrire une nouvelle règle prend cinq minutes, pas un sprint de yak-shaving.

NetArchTest et ArchUnitNET livrent les deux.

## Vue d'ensemble : ce que tu peux imposer

Les tests d'architecture couvrent trois grandes catégories :

{{< mermaid >}}
graph TD
    A[Tests d'architecture] --> B[Règles de dépendances<br/>qui peut référencer quoi]
    A --> C[Règles de nommage<br/>suffixes, préfixes, namespaces]
    A --> D[Règles structurelles<br/>sealed, public, abstract, interfaces]
{{< /mermaid >}}

Les règles de dépendances sont les plus précieuses : elles protègent la forme de l'applicationcation. Les règles de nommage et structurelles sont moins chères mais s'additionnent pour donner une vraie cohérence à un grand codebase.

> 💡 **Info** : Les tests d'architecture tournent comme des tests xUnit / NUnit classiques. Pas d'outillage supplémentaire, pas de plugin SonarQube, pas d'analyseur Roslyn custom à moins que tu le veuilles. Les assertions s'exécutent en millisecondes contre tes assemblies compilés.

## Zoom : règles de dépendances avec NetArchTest

NetArchTest a une API fluent qui se lit comme du texte. La règle classique "Domain ne référence rien" :

```csharp
using NetArchTest.Rules;
using Xunit;

public class DomainDependencyTests
{
    [Fact]
    public void Domain_ne_doit_pas_dependre_de_Application_Infrastructure_ou_Api()
    {
        var result = Types.InAssembly(typeof(Shop.Domain.Orders.Order).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(
                "Shop.Application",
                "Shop.Infrastructure",
                "Shop.Api")
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            "Domain doit être indépendant. Types fautifs : "
            + string.Join(", ", result.FailingTypeNames ?? Array.Empty<string>()));
    }

    [Fact]
    public void Domain_ne_doit_pas_dependre_de_EntityFramework()
    {
        var result = Types.InAssembly(typeof(Shop.Domain.Orders.Order).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }
}
```

Le message d'assertion inclut les types fautifs quand il se déclenche, donc le dev qui a cassé la règle sait exactement quel fichier ouvrir.

> ✅ **Bonne pratique** : Ajoute un test de dépendance par couche, pas un seul test géant qui vérifie tout. Quand un échec apparaît, le nom du test te dit quelle frontière a été traversée.

## Zoom : règles de nommage et structurelles

La cohérence de nommage, c'est surtout un boulot de code review, mais les tests d'architecture attrapent la dérive :

```csharp
[Fact]
public void Chaque_IRequestHandler_doit_se_terminer_par_Handler()
{
    var result = Types.InAssembly(typeof(SubmitOrderHandler).Assembly)
        .That()
        .ImplementInterface(typeof(IRequestHandler<,>))
        .Should()
        .HaveNameEndingWith("Handler")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}

[Fact]
public void Chaque_commande_doit_etre_un_sealed_record()
{
    var result = Types.InAssembly(typeof(SubmitOrderCommand).Assembly)
        .That()
        .ImplementInterface(typeof(IRequest<>))
        .And()
        .ResideInNamespace("Shop.Application")
        .And()
        .HaveNameEndingWith("Command")
        .Should()
        .BeSealed()
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```

Ces règles sont peu chères, elles attrapent de la vraie dérive, et elles éduquent les nouveaux arrivants plus vite que n'importe quel style guide.

> ⚠️ **Ça marche, mais...** : Ne sur-contraint pas. Si tu écris 200 tests de nommage, chaque refactoring devient une négociation. Choisis les dix règles qui comptent vraiment pour ton équipe et laisse tomber le reste. Le goût fait partie du métier.

## Zoom : ArchUnitNET pour des assertions plus riches

Pour des règles plus complexes, ArchUnitNET a une API plus expressive :

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Fluent;
using ArchUnitNET.Loader;
using ArchUnitNET.xUnit;
using static ArchUnitNET.Fluent.ArchRuleDefinition;

public class ApplicationArchitectureTests
{
    private static readonly Architecture Architecture =
        new ArchLoader()
            .LoadAssemblies(typeof(SubmitOrderHandler).Assembly,
                            typeof(Order).Assembly)
            .Build();

    [Fact]
    public void Handlers_ne_doivent_etre_utilises_que_par_le_mediator()
    {
        IObjectProvider<Class> handlers = Classes()
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .As("Handlers");

        IObjectProvider<IType> mediator = Types()
            .That().ResideInNamespace("MediatR", true)
            .As("MediatR");

        Classes()
            .That().Are(handlers)
            .Should()
            .OnlyBeAccessedBy(mediator)
            .Check(Architecture);
    }
}
```

`OnlyBeAccessedBy` est exactement le genre de règle laborieuse à exprimer à la main et triviale avec ArchUnitNET. La règle dit "les handlers sont un détail d'implémentation privé, seul MediatR devrait y toucher", ce qui est exactement le comportement attendu d'un codebase CQRS.

> 💡 **Info** : ArchUnitNET charge l'assembly une fois dans un modèle in-memory, donc chaque règle tourne contre le même graphe. Pour un codebase avec 50 tests d'architecture, c'est nettement plus rapide que NetArchTest, qui reparcourt les types à chaque assertion.

## Zoom : par où commencer

Commence par les trois règles qui protègent le plus de valeur :

**1. La direction des dépendances** (Domain → rien, Application → Domain, Infrastructure → Application + Domain). C'est la seule règle qui empêche un projet Clean Architecture de s'effondrer en layered spaghetti.

**2. Pas de fuite de framework dans le domaine**. Domain ne doit pas référencer `Microsoft.EntityFrameworkCore`, `Microsoft.AspNetCore.*`, `System.Data.*`, `Stripe.*`. Un seul test avec une liste de namespaces interdits.

**3. Le nommage du vocabulaire partagé**. Si ton équipe dit "les commandes se terminent par `Command`", "les queries par `Query`", "les handlers par `Handler`", impose-le. Quand les mots dans le code correspondent aux mots dans les réunions, l'onboarding accélère visiblement.

Le reste, c'est du bonus. Ajoute des règles quand un vrai incident t'a appris la leçon, pas par anticipation.

> ✅ **Bonne pratique** : Mets les tests d'architecture dans un projet dédié, `Shop.ArchitectureTests`, qui référence tous les autres assemblies. Ils tournent en CI à côté des tests unitaires et font échouer le build sur la moindre violation.

> ❌ **Ne jamais faire** : Ne mets pas `Skip` sur un test d'architecture quand quelqu'un casse la règle "pour débloquer une release". Un test d'architecture skippé, c'est une règle qui n'existe plus. Soit tu corriges le code, soit tu supprimes le test et tu reconnais ouvertement que la règle est partie.

## Quand les tests d'architecture sont le mauvais outil

Les tests d'architecture marchent bien pour des règles sur le graphe d'assembly, les noms de types, et la structure de types. Ce n'est **pas** le bon outil pour :

- **La correction logique** : "la remise est de 15% au-dessus de 50 articles". C'est un test unitaire.
- **Le comportement runtime** : "le handler commit la transaction". C'est un test d'intégration.
- **La performance** : "cette requête tourne en moins de 100ms". C'est un benchmark ou un test de charge.
- **La sécurité** : "cet endpoint exige le rôle `admin`". C'est un test WebApplicationFactory.

Si tu te surprends à écrire `Types.InAssembly(...).Should().HaveMethodBody("...")`, arrête et écris un vrai test à la place.

## Wrap-up

Tu sais maintenant comment transformer des décisions d'architecture en invariants exécutables : choisir NetArchTest pour des règles fluent simples ou ArchUnitNET pour des assertions de graphe plus riches, commencer par les tests de direction de dépendances et de fuite de framework, ajouter des règles de nommage qui reflètent le vocabulaire partagé de ton équipe, mettre tout ça dans un projet de test d'architecture dédié, et refuser de les skipper sous pression. Tu peux rendre le codebase résistant à la dérive structurelle lente qui tue tous les projets de longue durée.

Prêt à booster ton prochain projet ou à le partager avec ton équipe ? À la prochaine, a++ 👋

## Pour aller plus loin

- [Clean Architecture en .NET : des dépendances qui pointent dans le bon sens](/fr/posts/code-structure-clean-architecture/)
- [Les Tests Unitaires en .NET : rapides, ciblés, et vraiment utiles](/fr/posts/testing-unit-testing/)
- [Tests d'Intégration avec TestContainers pour .NET](/fr/posts/testing-integration-testing-testcontainers/)
- [Tests API avec WebApplicationFactory en ASP.NET Core](/fr/posts/testing-webapplicationfactory/)
- [Tests End-to-End avec Playwright pour .NET](/fr/posts/testing-e2e-playwright/)

## Références

- [NetArchTest sur GitHub](https://github.com/BenMorris/NetArchTest)
- [ArchUnitNET sur GitHub](https://github.com/TNG/ArchUnitNET)
- [ArchUnit (original Java)](https://www.archunit.org/)
- [Clean Architecture, Microsoft Learn](https://learn.microsoft.com/fr-fr/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture)
