---
title: "Dapper : quand et comment l'utiliser"
date: 2026-04-09
draft: false
tags: ["database", "dapper", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Dapper n'est pas un remplaçant d'EF Core, c'est un outil différent avec un contrat différent. Quand le sortir, comment l'utiliser correctement, et comment le faire cohabiter avec EF Core dans le même code."
---

Hello tous le monde, aujourd'hui on va comprendre **quand sortir Dapper et comment le faire cohabiter avec EF Core**.

Dapper existe depuis 2011, écrit par l'équipe de Stack Overflow pour gérer la charge SQL qu'Entity Framework de l'époque n'arrivait pas à tenir. Il est toujours maintenu, toujours rapide, toujours la bonne réponse pour un ensemble précis de problèmes. L'erreur que font la plupart des équipes, c'est de traiter ça comme un choix entre Dapper et EF Core, comme s'il fallait en retenir un pour tout le projet. En pratique, les deux cohabitent proprement : EF Core pour les écritures et pour les 90% de lectures que LINQ sait bien exprimer, Dapper pour les requêtes de reporting, les dashboards, les exports massifs et les hot paths où le SQL n'est fondamentalement pas une requête LINQ.

Cet article décrit ce qu'est vraiment Dapper, où il gagne, comment l'utiliser sans reconstruire un mini-ORM autour de lui, et comment le faire tourner dans le même code qu'EF Core.

## Le contexte : pourquoi Dapper existe

Dapper est un ensemble de méthodes d'extension sur `IDbConnection`. C'est toute la bibliothèque. On ouvre une connexion, on appelle `connection.QueryAsync<T>("SELECT ...", parameters)`, et Dapper mappe le result set vers `T` en faisant correspondre les noms de colonnes aux noms de propriétés. Pas de modèle, pas de change tracker, pas de migrations, pas de provider LINQ, pas de mapping de relations. On écrit le SQL, Dapper l'exécute et hydrate les objets.

Ce minimalisme, c'est le principe. EF Core résout le problème "j'ai un modèle de domaine et je veux que la base suive". Dapper résout le problème "j'ai une requête SQL et je veux qu'elle me renvoie des objets typés". Ce sont deux problèmes différents, et une équipe qui comprend la distinction arrête de se disputer sur lequel choisir.

L'écart de performance entre Dapper et EF Core s'est nettement réduit depuis EF Core 6. Sur une requête d'entité simple, EF Core avec `AsNoTracking()` et une projection est typiquement à 10-15% de Dapper. L'écart se creuse sur les requêtes aux formes SQL inhabituelles : window functions, CTE récursives, `UNION ALL` sur des tables sans rapport, ordre de `JOIN` tuné à la main. Ce sont les requêtes où Dapper gagne son salaire, non pas parce qu'il est plus rapide sur les cas triviaux mais parce que les écrire en LINQ est soit impossible, soit produit un SQL traduit qu'on ne livrerait pas.

## Vue d'ensemble : où Dapper trouve sa place

{{< mermaid >}}
graph TD
    A[Besoin d'accès aux données] --> B{Forme}
    B -->|Écriture domaine : charger l'agrégat, modifier, sauver| C[EF Core]
    B -->|Lecture typée liste/détail d'une entité| C
    B -->|Requête de reporting, CTE, window function| D[Dapper]
    B -->|Export massif, endpoint read-heavy| D
    B -->|Stitching multi-tables avec SQL tuné à la main| D
    C --> E[ShopDbContext]
    D --> F[IDbConnection + fichiers SQL]
{{< /mermaid >}}

Le partage se fait par forme de requête, pas par domaine fonctionnel. Un même module peut avoir EF Core pour ses command handlers et Dapper pour le read model qui alimente le dashboard.

## Zoom : une installation Dapper propre

```csharp
public sealed class OrderReports
{
    private readonly string _connectionString;

    public OrderReports(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("Shop")
            ?? throw new InvalidOperationException("Connection string manquante");
    }

    public async Task<IReadOnlyList<MonthlyRevenueRow>> GetMonthlyRevenueAsync(int year, CancellationToken ct)
    {
        const string sql = """
            SELECT
                date_trunc('month', o.created_at) AS Month,
                SUM(o.total_amount)                AS Revenue,
                COUNT(*)                           AS OrderCount
            FROM orders o
            WHERE EXTRACT(YEAR FROM o.created_at) = @Year
              AND o.status = 'completed'
            GROUP BY date_trunc('month', o.created_at)
            ORDER BY Month;
            """;

        await using var conn = new NpgsqlConnection(_connectionString);
        var rows = await conn.QueryAsync<MonthlyRevenueRow>(
            new CommandDefinition(sql, new { Year = year }, cancellationToken: ct));
        return rows.ToList();
    }
}

public sealed record MonthlyRevenueRow(DateTime Month, decimal Revenue, int OrderCount);
```

Trois choses à noter. D'abord, `CommandDefinition` est la surcharge moderne qui accepte un `CancellationToken`. À utiliser sur chaque appel : l'annulation n'est pas gratuite sur un rapport long. Ensuite, les paramètres passent via un objet anonyme, que Dapper transforme en SQL paramétré. On ne concatène jamais une entrée utilisateur dans la chaîne SQL. Enfin, le SQL est dans un raw string literal (`"""`), ce qui le garde lisible et indenté naturellement.

> 💡 **Info** — Les raw string literals sont arrivés en C# 11. Pour des frameworks cibles plus anciens, on déplace le SQL dans un fichier `.sql` embarqué et on le lit avec `typeof(OrderReports).Assembly.GetManifestResourceStream(...)`.

## Zoom : réutiliser la connexion EF Core

Quand EF Core et Dapper cohabitent dans la même requête, l'erreur commune est d'ouvrir deux connexions séparées. EF Core en a une ouverte dans `DbContext`, et Dapper en ouvre une deuxième, ce qui double l'usage du pool de connexions et casse toute garantie transactionnelle qu'on aurait pu vouloir entre les deux. La correction, c'est de demander sa connexion au `DbContext` :

```csharp
public sealed class OrderReports
{
    private readonly ShopDbContext _db;

    public OrderReports(ShopDbContext db) => _db = db;

    public async Task<IReadOnlyList<MonthlyRevenueRow>> GetMonthlyRevenueAsync(int year, CancellationToken ct)
    {
        var conn = _db.Database.GetDbConnection();
        if (conn.State != ConnectionState.Open)
            await conn.OpenAsync(ct);

        const string sql = /* même SQL que plus haut */;
        var rows = await conn.QueryAsync<MonthlyRevenueRow>(
            new CommandDefinition(sql, new { Year = year }, cancellationToken: ct));
        return rows.ToList();
    }
}
```

`_db.Database.GetDbConnection()` renvoie la `DbConnection` sous-jacente qu'EF Core gère. Dapper peut exécuter n'importe quelle requête dessus, et on reste dans la même transaction s'il y en a une d'active. Quand le `DbContext` est disposé à la fin du scope, la connexion retourne au pool.

> ✅ **Bonne pratique** — Pour tout ce qui tourne dans une requête qui touche aussi EF Core, on partage la connexion. Pour les jobs d'arrière-plan ou les endpoints de reporting autonomes sans EF Core, un `NpgsqlConnection` dédié reste parfaitement correct.

## Zoom : résultats multi-lignes et splitOn

Quand la requête joint plusieurs tables et qu'on veut un parent avec ses enfants mappés dans des types CLR différents, on utilise `QueryAsync` avec un splitOn :

```csharp
const string sql = """
    SELECT o.id, o.reference, o.total_amount,
           c.id, c.email, c.loyalty_tier
    FROM orders o
    INNER JOIN customers c ON c.id = o.customer_id
    WHERE o.created_at > @Since;
    """;

var rows = await conn.QueryAsync<OrderRow, CustomerRow, OrderWithCustomer>(
    sql,
    (order, customer) => new OrderWithCustomer(order, customer),
    new { Since = since },
    splitOn: "id");
```

`splitOn: "id"` dit à Dapper où couper la ligne de résultat entre les deux types mappés. Tout ce qui va de la première colonne jusqu'à la première colonne `id` suivante (exclue) devient le premier type, le reste devient le second. C'est le seul détail Dapper qui piège tout le monde la première fois.

> ⚠️ **Ça marche, mais...** — Pour les requêtes qui joignent quatre tables ou plus, la syntaxe splitOn devient difficile à lire. À ce stade, on renvoie des DTO plats et on fait le stitching en C#, ou on déplace la requête vers une vue SQL et on mappe la vue vers un seul DTO.

## Zoom : DynamicParameters et procédures stockées

```csharp
var parameters = new DynamicParameters();
parameters.Add("CustomerId", customerId, DbType.Guid);
parameters.Add("Since", since, DbType.DateTime2);
parameters.Add("TotalOut", dbType: DbType.Decimal, direction: ParameterDirection.Output);

await conn.ExecuteAsync(
    "sp_compute_customer_total",
    parameters,
    commandType: CommandType.StoredProcedure);

var total = parameters.Get<decimal>("TotalOut");
```

`DynamicParameters` est l'échappatoire pour tout ce qui dépasse les paramètres d'entrée simples : paramètres de sortie, valeurs de retour, procédures stockées, contrôle typé du `DbType` pour les cas où le mapping par défaut de Dapper est faux.

> ❌ **Ne jamais faire** — Ne pas construire le SQL par concaténation de chaînes sous prétexte que "c'est juste pour un dashboard admin". Le dashboard admin est le premier endroit où une injection SQL se remarque, parce que c'est là que quelqu'un finit par coller une valeur de filtre avec une apostrophe dedans. Des paramètres, toujours.

## Zoom : quand ne pas utiliser Dapper

Dapper est le mauvais choix pour trois choses :

1. **Écrire les agrégats de domaine.** Pas de change tracker, donc soit on écrit tous les INSERT et UPDATE à la main, soit on finit par reconstruire un EF Core mal ficelé. On garde les écritures sur EF Core.
2. **Filtrage multi-tenant.** Les global query filters d'EF Core sont déclaratifs et difficiles à oublier. En Dapper, le filtre de tenant est un `WHERE tenant_id = @TenantId` qu'il faut se rappeler sur chaque requête. Un oubli, et on lit les données d'un autre tenant.
3. **Projets avec des développeurs juniors et pas de culture de revue SQL.** Dapper fait confiance à l'auteur. Il ne protège pas d'un mauvais ordre de `JOIN`, d'un index manquant, ou d'une requête qui a l'air bien en dev et qui met la prod à genoux. Sur une équipe qui ne sait pas relire le SQL, les garde-fous d'EF Core valent leur léger surcoût.

## Wrap-up

Tu sais maintenant que Dapper et EF Core sont complémentaires, pas concurrents. EF Core pour le côté écriture et les lectures typées que LINQ exprime proprement ; Dapper pour le reporting, les requêtes à base de CTE, les dashboards, et tout ce où le SQL est le langage de première classe. Partager la connexion via `_db.Database.GetDbConnection()` permet aux deux de cohabiter dans la même requête et la même transaction. Une fois le partage clair, le débat "EF Core ou Dapper" s'arrête d'être un débat.

Prêt à sortir Dapper sur la prochaine requête de reporting qui résiste à LINQ, ou à poser ce partage dans ton équipe ?

À la prochaine, a++ 👋

## Pour aller plus loin

- [EF Core : Optimisation des Lectures](/fr/posts/database-efcore-read-optimization/)
- [Accès aux données en .NET : Repository Pattern, ou pas ?](/fr/posts/layer-focused-repository-or-not/)
- [Couche Application : CQS et CQRS](/fr/posts/layer-focused-cqs-cqrs/)

## Références

- [Dapper sur GitHub](https://github.com/DapperLib/Dapper)
- [Dapper sur NuGet](https://www.nuget.org/packages/Dapper)
- [EF Core : Accès à la connexion ADO.NET sous-jacente](https://learn.microsoft.com/fr-fr/ef/core/miscellaneous/connections)
- [Documentation Npgsql](https://www.npgsql.org/doc/index.html)
