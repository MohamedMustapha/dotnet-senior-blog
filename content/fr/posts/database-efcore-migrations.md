---
title: "EF Core : les Migrations en pratique"
date: 2026-04-09
draft: false
tags: ["database", "efcore", "migrations", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Une migration est du code qu'on commite, pas de la magie exécutée par l'outillage. Comment écrire, relire et déployer des migrations EF Core en équipe sans casser la production."
---

Hello tous le monde, aujourd'hui on va démystifier **les migrations EF Core telles qu'on les pratique en équipe**.

Les migrations ont l'air simples dans la première démo : on change le modèle, on lance `dotnet ef migrations add`, puis `dotnet ef database update`, et le schéma suit. Et puis arrivent le deuxième développeur, le premier conflit de pull request sur le snapshot du modèle, la première table de production à 40 millions de lignes, et le premier renommage de colonne qui perd silencieusement la donnée. À ce moment-là, l'outil est le même mais le modèle mental doit changer : une migration est un bout de code versionné qui va tourner sur une base vivante, sur une machine qui n'est pas la tienne, à un moment que tu ne choisis pas. Traite ça comme du code, relis-le comme du code, et les surprises s'arrêtent.

Cet article déroule l'écriture d'une migration dans une équipe, sa relecture avant production, et les patterns qui gardent le déploiement sans interruption possible.

## Le contexte : pourquoi les migrations demandent leur propre discipline

EF Core génère les migrations en diffant le modèle courant contre un snapshot du modèle précédent. Le snapshot vit dans `ModelSnapshot.cs`, commité dans le repo à côté des fichiers de migration. Deux développeurs qui ajoutent une migration sur deux branches produisent deux snapshots, et quand les branches fusionnent on obtient un conflit de snapshot qui ne se résout pas par "accept both". L'outillage impose un rebase et une régénération, ce qui est la première raison pour laquelle les migrations demandent de la coordination.

La deuxième raison, c'est que le code C# de la migration n'est que la moitié de l'histoire. EF Core le traduit en SQL au runtime selon le provider. Ce SQL dépend de la version de la base, de la collation, des données existantes, des verrous qu'il prend, et de si le déploiement applique la migration hors-ligne ou sur un cluster en production. Une migration qui drop une colonne est instantanée sur une base dev vide et un verrou de table de six minutes sur une table de 40 millions de lignes en prod. Ni EF Core ni la suite de tests ne préviennent : il faut lire le SQL généré.

## Vue d'ensemble : le cycle de vie d'une migration

{{< mermaid >}}
flowchart LR
    A[Changer le modèle] --> B[dotnet ef migrations add Name]
    B --> C[Relire le fichier C# de la migration]
    C --> D[Relire le SQL généré<br/>dotnet ef migrations script]
    D --> E{Safe ?}
    E -->|Non| F[Découper en plusieurs étapes]
    F --> B
    E -->|Oui| G[Commit de la migration<br/>et du snapshot]
    G --> H[CI applique sur l'env d'intégration]
    H --> I[Déploiement production<br/>applique la migration]
{{< /mermaid >}}

Deux idées qui font tenir ce flow :

1. Le fichier de migration est relu comme du code source. `Up` et `Down` sont lus par un humain, pas juste générés et commités les yeux fermés.
2. Le SQL généré est scripté et relu avant le merge. `dotnet ef migrations script` émet un SQL idempotent qu'on attache à la pull request.

## Zoom : ajouter une migration

```bash
dotnet ef migrations add AddCustomerLoyaltyTier \
  --project src/Shop.Infrastructure \
  --startup-project src/Shop.Api \
  --output-dir Persistence/Migrations
```

Trois flags comptent dans un vrai repo. `--project` pointe vers l'assembly qui contient le `DbContext`. `--startup-project` pointe vers l'exécutable qui le configure, parce qu'EF Core doit construire un service provider pour lire la chaîne de connexion et la configuration design-time. `--output-dir` range les migrations dans un dossier dédié.

Le fichier généré ressemble à ça :

```csharp
public partial class AddCustomerLoyaltyTier : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "loyalty_tier",
            table: "customers",
            type: "varchar(20)",
            nullable: false,
            defaultValue: "standard");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "loyalty_tier",
            table: "customers");
    }
}
```

On lit les deux méthodes. `Up`, c'est ce qui se passe au déploiement ; `Down`, c'est ce qui se passe au rollback, et un `Down` qui supprime une colonne qu'on vient de remplir avec de la vraie donnée, c'est un piège.

> 💡 **Info** — Le fichier de migration est une `partial class` parce qu'EF Core génère un second fichier `.Designer.cs` à côté, qui capture le snapshot du modèle au moment de la migration. Les deux fichiers appartiennent à git. Ne jamais supprimer l'un sans l'autre.

## Zoom : relire le SQL généré

Le fichier C# est une description. Le SQL est ce qui tourne vraiment. On le génère avant le merge :

```bash
dotnet ef migrations script LastMigration AddCustomerLoyaltyTier \
  --project src/Shop.Infrastructure \
  --startup-project src/Shop.Api \
  --idempotent \
  --output migration.sql
```

`--idempotent` fait en sorte que le script vérifie `__EFMigrationsHistory` avant chaque étape, donc on peut le rejouer deux fois. La sortie, c'est le SQL exact que la base de production va exécuter. On l'attache à la pull request. Le relire prend une minute et attrape ce qu'EF Core ne signale pas : un `ALTER TABLE` avec un default qui verrouille la table, un `DROP COLUMN` qui perd de la donnée, une nouvelle colonne non-null sur une grosse table qui échoue parce que les lignes existantes n'ont pas de valeur.

> ✅ **Bonne pratique** — Imposer "le script SQL de la migration est attaché à la PR" dans la checklist. C'est le filtre à défauts le moins cher qu'on ajoutera jamais à un workflow base de données.

## Zoom : les opérations qui demandent deux migrations

Tout changement qui ne peut pas s'appliquer en une étape atomique se découpe en plusieurs migrations. Les classiques :

**Renommer une colonne.** EF Core ne voit pas un renommage, il voit un drop suivi d'un add, ce qui perd la donnée. Il faut l'écrire comme deux étapes : ajouter la nouvelle colonne, copier la donnée dans un script de déploiement, puis dans une migration ultérieure supprimer l'ancienne. Ou alors, utiliser `migrationBuilder.RenameColumn` à la main :

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(
        name: "customer_email",
        table: "customers",
        newName: "email");
}
```

EF Core ne génère `RenameColumn` que quand on renomme la propriété avec exactement le même type CLR. Si on renomme *et* qu'on change le type, on obtient un drop-and-add et on perd la donnée.

**Ajouter une colonne non-null à une table peuplée.** La migration naïve échoue sur la première ligne existante. La correction, c'est une approche en deux temps : l'ajouter en nullable avec une valeur par défaut, backfill la donnée, puis dans une migration ultérieure la passer en non-null.

```csharp
// Migration 1
migrationBuilder.AddColumn<string>("loyalty_tier", "customers",
    nullable: true, defaultValue: "standard");
migrationBuilder.Sql("UPDATE customers SET loyalty_tier = 'standard' WHERE loyalty_tier IS NULL;");

// Migration 2 (release ultérieure)
migrationBuilder.AlterColumn<string>("loyalty_tier", "customers",
    nullable: false, type: "varchar(20)");
```

**Séparer une table.** Jamais en une seule migration. Ajouter la nouvelle table, faire du dual-write depuis l'application pendant une release, backfill la donnée historique, puis dans une release ultérieure supprimer les colonnes de l'ancienne table.

> ⚠️ **Ça marche, mais...** — Une migration unique qui fait "ajout de la nouvelle structure, backfill, suppression de l'ancienne" marche aussi sur un environnement de dev, et le jour où on la déploie en production elle tient un verrou pendant toute la durée du backfill. Sur une grosse table, c'est l'incident.

> ❌ **Ne jamais faire** — Ne pas éditer une migration qui a déjà été appliquée dans un environnement partagé. L'historique des migrations est le contrat. Si on doit corriger quelque chose, on ajoute une nouvelle migration. Éditer l'historique fait échouer le `dotnet ef database update` du développeur suivant avec un checksum mismatch incompréhensible.

## Zoom : appliquer les migrations au runtime ou au déploiement

Deux endroits peuvent appliquer les migrations : l'application elle-même au démarrage via `context.Database.MigrateAsync()`, ou une étape dédiée du pipeline de déploiement via `dotnet ef database update` ou un bundle SQL généré.

Les migrations au runtime sont les plus simples pour les petites apps. Une instance boot, migre et commence à servir. Le mode d'échec, c'est le déploiement multi-instances : dix pods appellent `MigrateAsync()` en même temps, le verrou de migration d'EF Core (`__EFMigrationsHistory`) les sérialise, mais ceux qui perdent la course prennent le temps de démarrage supplémentaire pendant que le gagnant applique une longue migration. Pire, une migration qui échoue à mi-chemin laisse la base dans un état où aucun des pods ne peut démarrer.

Les migrations au déploiement découplent le changement de schéma du démarrage de l'application. Le pipeline applique la migration une fois, puis déroule l'application. Le revers, c'est que schéma et code doivent être compatibles pour la fenêtre de recouvrement où le nouveau schéma est en place mais certains pods tournent encore sur l'ancien code. C'est toute la raison d'être du pattern des deux migrations : chaque déploiement est rétrocompatible avec la version précédente.

```bash
# Option A : runtime (simple, une seule instance)
app.Services.GetRequiredService<ShopDbContext>().Database.Migrate();

# Option B : pipeline de déploiement (recommandé en multi-instance)
dotnet ef migrations bundle \
  --project src/Shop.Infrastructure \
  --startup-project src/Shop.Api \
  --self-contained \
  --output efbundle
./efbundle --connection "Host=db;Database=shop;Username=deploy;Password=***"
```

`migrations bundle` produit un exécutable autonome qui applique les migrations sans nécessiter le SDK .NET sur la machine cible. C'est le remplaçant moderne de l'envoi d'un script SQL, parce qu'il gère le suivi `__EFMigrationsHistory` tout en tournant en dehors du processus de l'application.

> 💡 **Info** — Depuis EF Core 7, les migration bundles sont la manière recommandée d'appliquer les migrations en CI/CD. Ils fonctionnent avec n'importe quel provider et ne demandent pas le SDK .NET sur la cible.

## Zoom : le conflit de snapshot

Deux développeurs ajoutent chacun une migration sur leur branche. Les deux commits modifient `ModelSnapshot.cs`. Quand le second merge, git signale un conflit dans le fichier snapshot, et il n'y a pas de manière saine de "accept both".

La correction, c'est un rebase, pas une édition manuelle du merge :

```bash
git fetch origin
git rebase origin/main
# conflit dans ModelSnapshot.cs et dans une des migrations
dotnet ef migrations remove --project src/Shop.Infrastructure --startup-project src/Shop.Api
# les changements de modèle sont toujours là, on reconstruit la migration
dotnet ef migrations add YourMigrationName --project src/Shop.Infrastructure --startup-project src/Shop.Api
git add .
git rebase --continue
```

`migrations remove` supprime les fichiers de migration et remet le snapshot dans l'état précédent. On rajoute ensuite la migration par-dessus la branche fraîchement rebasée, et le snapshot redevient cohérent.

> ✅ **Bonne pratique** — Merger les migrations sur main une pull request à la fois. On sérialise l'ordre. Des merges de migrations en parallèle produisent un conflit de snapshot à 100%, et la correction est toujours un rebase pour l'un des deux auteurs.

## Wrap-up

Tu sais maintenant que les migrations sont du code source, pas de la magie. Tu peux relire le fichier C#, générer et lire le SQL avant le merge, découper les opérations risquées en deux déploiements, et choisir consciemment entre l'application au runtime ou depuis le pipeline. Une fois que l'équipe s'accorde sur ce flow, le bug de migration sort du rapport d'incident pour de bon.

Prêt à imposer la relecture du script SQL sur ta prochaine PR, ou à partager cette discipline avec ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [EF Core : Configuration et Seed](/fr/posts/database-efcore-config-seed/)
- [Accès aux données en .NET : Repository Pattern, ou pas ?](/fr/posts/layer-focused-repository-or-not/)

## Références

- [EF Core : Gérer les migrations](https://learn.microsoft.com/fr-fr/ef/core/managing-schemas/migrations/)
- [EF Core : Appliquer les migrations](https://learn.microsoft.com/fr-fr/ef/core/managing-schemas/migrations/applying)
- [EF Core : Migration Bundles](https://learn.microsoft.com/fr-fr/ef/core/managing-schemas/migrations/applying#bundles)
- [EF Core : Opérations de migration personnalisées](https://learn.microsoft.com/fr-fr/ef/core/managing-schemas/migrations/operations)
