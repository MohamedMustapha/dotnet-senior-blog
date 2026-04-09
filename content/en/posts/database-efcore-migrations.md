---
title: "EF Core: Migrations in Practice"
date: 2026-04-09
draft: false
tags: ["database", "efcore", "migrations", "dotnet"]
categories: ["Database"]
series: ["Database"]
description: "Migrations are code you commit, not magic the tooling performs. How to author, review, and roll out EF Core migrations in a team without breaking production."
---

Migrations look simple in the first demo: you change the model, you run `dotnet ef migrations add`, you run `dotnet ef database update`, and the schema follows. Then you hit the second developer, the first pull request conflict on the model snapshot, the first production table with 40 million rows, and the first column rename that silently drops data. At that point the tool is the same but the mental model has to change: a migration is a piece of versioned code that is going to run against a live database, on a machine that is not yours, at a moment you do not choose. Treat it like code, review it like code, and the surprises stop.

This article walks through authoring migrations for a team, reviewing them before they hit production, and the patterns that keep zero-downtime deployments possible.

## Why migrations need their own discipline

EF Core generates migrations by diffing your current model against a snapshot of the previous model. The snapshot lives in `ModelSnapshot.cs`, checked into your repo alongside the migration files. Two developers adding migrations on two branches produce two snapshots, and when the branches merge you get a snapshot conflict that cannot be resolved by "accept both". The tooling requires you to rebase and regenerate, which is the first reason migrations need coordination.

The second reason is that the C# migration code is only half the story. EF Core translates it into SQL at runtime based on the provider. That SQL depends on the database version, the collation, the existing data, the locks it takes, and whether your deployment runs the migration offline or on a live cluster. A migration that drops a column is instant on an empty dev database and a six-minute table lock on a 40-million-row production table. Neither EF Core nor your test suite will warn you: you have to read the generated SQL.

## Overview: the migration lifecycle

{{< mermaid >}}
flowchart LR
    A[Change model] --> B[dotnet ef migrations add Name]
    B --> C[Review C# migration file]
    C --> D[Review generated SQL<br/>dotnet ef migrations script]
    D --> E{Safe?}
    E -->|No| F[Split into multiple steps]
    F --> B
    E -->|Yes| G[Commit migration<br/>and snapshot]
    G --> H[CI applies in integration env]
    H --> I[Production deployment<br/>applies migration]
{{< /mermaid >}}

Two ideas that make this flow work:

1. The migration file is reviewed as source code. `Up` and `Down` are read by a human, not just generated and committed blindly.
2. The generated SQL is scripted and read before merge. `dotnet ef migrations script` emits idempotent SQL you can attach to the pull request.

## Zoom: adding a migration

```bash
dotnet ef migrations add AddCustomerLoyaltyTier \
  --project src/Shop.Infrastructure \
  --startup-project src/Shop.Api \
  --output-dir Persistence/Migrations
```

Three flags matter in a real repo. `--project` points to the assembly that contains the `DbContext`. `--startup-project` points to the executable that configures it, because EF Core needs to build a service provider to read the connection string and the design-time configuration. `--output-dir` keeps migrations in a dedicated folder.

The generated file looks like this:

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

Read both methods. `Up` is what happens on deploy; `Down` is what happens on rollback, and a `Down` that drops a column you just filled with real data is a trap.

> 💡 **Info** — The migration file is a `partial class` because EF Core generates a second `.Designer.cs` file alongside it that captures the model snapshot at the time of the migration. Both files belong to git. Do not delete one without the other.

## Zoom: reading the generated SQL

The C# file is a description. The SQL is what actually runs. Generate it before merge:

```bash
dotnet ef migrations script LastMigration AddCustomerLoyaltyTier \
  --project src/Shop.Infrastructure \
  --startup-project src/Shop.Api \
  --idempotent \
  --output migration.sql
```

`--idempotent` makes the script check `__EFMigrationsHistory` before applying each step, so you can run the same script twice. The output is the exact SQL the production database will execute. Attach it to the pull request. Reviewing it takes a minute and catches the problems EF Core does not warn about: an `ALTER TABLE` with a default that locks the table, a `DROP COLUMN` that silently loses data, a new non-null column on a large table that fails because existing rows do not have a value.

> ✅ **Good practice** — Make "the migration script is attached to the PR" a rule in the checklist. It is the cheapest defect filter you will ever add to your database workflow.

## Zoom: the operations that need two migrations

Any change that cannot be applied as a single atomic step needs to be split into multiple migrations. The classic ones:

**Rename a column.** EF Core does not see a rename, it sees a drop-and-add, which loses data. You have to write it as two steps yourself: add the new column, copy the data in a deployment script, then in a later migration drop the old column. Alternatively, use `migrationBuilder.RenameColumn` by hand:

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.RenameColumn(
        name: "customer_email",
        table: "customers",
        newName: "email");
}
```

EF Core only generates `RenameColumn` when you rename the property with the exact same CLR type. If you rename *and* retype, you get a drop-and-add and you lose data.

**Add a non-null column to a populated table.** The naive migration fails on the first existing row. The fix is a two-phase approach: add it as nullable with a default, backfill the data, then in a later migration alter it to non-null.

```csharp
// Migration 1
migrationBuilder.AddColumn<string>("loyalty_tier", "customers",
    nullable: true, defaultValue: "standard");
migrationBuilder.Sql("UPDATE customers SET loyalty_tier = 'standard' WHERE loyalty_tier IS NULL;");

// Migration 2 (later release)
migrationBuilder.AlterColumn<string>("loyalty_tier", "customers",
    nullable: false, type: "varchar(20)");
```

**Split a table.** Never do it in one migration. Add the new table, dual-write from the application for one release, backfill historical data, then in a later release drop the columns from the old table.

> ⚠️ **Works, but...** — A single migration that does "add new structure, backfill, remove old structure" also works in a dev environment and on the day you deploy it to production it holds a lock for the duration of the backfill. On a large table, that is your outage.

> ❌ **Never do** — Do not edit a migration that has already been applied in any shared environment. The migration history is the contract. If you need to fix something, add a new migration. Editing history makes the next developer's `dotnet ef database update` fail with a confusing checksum mismatch.

## Zoom: applying migrations at runtime or at deploy time

Two places can run migrations: the application itself at startup via `context.Database.MigrateAsync()`, or a dedicated step in the deployment pipeline via `dotnet ef database update` or a generated SQL bundle.

Runtime migrations are the easiest for small apps. One instance boots, migrates, and starts serving. The failure mode is multi-instance deployments: ten pods all call `MigrateAsync()` at the same time, and EF Core's migration lock (`__EFMigrationsHistory`) serializes them, but the ones that lose the race now take the extra startup time while the winner is applying a long migration. Worse, a migration that fails halfway leaves the database in a state where none of the pods can start.

Deploy-time migrations decouple the schema change from the application start. The pipeline applies the migration once, then rolls out the application. The downside is that the schema and the code must be deploy-compatible for the overlap window where the new schema is live but some pods still run the old code. That is the whole point of the two-migration pattern: each deployment is backward-compatible with the previous version.

```bash
# Option A: runtime (simple, single instance)
app.Services.GetRequiredService<ShopDbContext>().Database.Migrate();

# Option B: deploy pipeline (recommended for multi-instance)
dotnet ef migrations bundle \
  --project src/Shop.Infrastructure \
  --startup-project src/Shop.Api \
  --self-contained \
  --output efbundle
./efbundle --connection "Host=db;Database=shop;Username=deploy;Password=***"
```

`migrations bundle` produces a self-contained executable that applies migrations without requiring the .NET SDK on the target machine. It is the modern replacement for shipping a SQL script, because it handles the `__EFMigrationsHistory` tracking for you while still running outside the application process.

> 💡 **Info** — Since EF Core 7, migration bundles are the recommended way to apply migrations in CI/CD. They work with any provider and do not need the .NET SDK on the deployment target.

## Zoom: the snapshot conflict

Two developers each add a migration on their own branch. Both commits modify `ModelSnapshot.cs`. When the second one merges, git reports a conflict in the snapshot file, and there is no sane way to "accept both".

The fix is a rebase, not a merge edit:

```bash
git fetch origin
git rebase origin/main
# conflict in ModelSnapshot.cs and in one of the migrations
dotnet ef migrations remove --project src/Shop.Infrastructure --startup-project src/Shop.Api
# your model changes are still there, rebuild the migration
dotnet ef migrations add YourMigrationName --project src/Shop.Infrastructure --startup-project src/Shop.Api
git add .
git rebase --continue
```

`migrations remove` deletes the migration files and reverts the snapshot to the previous state. You then re-add the migration on top of the freshly rebased branch, and the snapshot becomes consistent again.

> ✅ **Good practice** — Merge migrations to main one pull request at a time. Serialize the order. Parallel migration merges produce snapshot conflicts 100% of the time, and the fix is always a rebase for one of the two authors.

## Wrap-up

Migrations are source code, not magic. You review the C# file, you generate and read the SQL before merge, you split risky operations into two deployments, and you decide consciously whether to apply them at runtime or from the pipeline. Once the team agrees on this flow, the migration bug disappears from the incident report for good.

The next step is making reads fast. Even on a perfectly migrated schema, a badly written LINQ query can bring the database down.

## Related articles

- [EF Core: Configuration and Seeding](/en/posts/database-efcore-config-seed/)
- [Data Access in .NET: Repository Pattern, or Not?](/en/posts/layer-focused-repository-or-not/)

## References

- [EF Core: Managing Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [EF Core: Applying Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying)
- [EF Core: Migration Bundles](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/applying#bundles)
- [EF Core: Custom Migration Operations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/operations)
