---
title: What's New in EF Core 5.0
description: Overview of new features in EF Core 5.0
author: SamMonoRT
ms.date: 09/10/2020
uid: core/what-is-new/ef-core-5.0/whatsnew
---

# What's New in EF Core 5.0

The following list includes the major new features in EF Core 5.0. For the full list of issues in the release, see our [issue tracker](https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A5.0.0).

As a major release, EF Core 5.0 also contains several [breaking changes](xref:core/what-is-new/ef-core-5.0/breaking-changes), which are API improvements or behavioral changes that may have negative impact on existing applications.

## Many-to-many

EF Core 5.0 supports many-to-many relationships without explicitly mapping the join table.

For example, consider these entity types:

```csharp
public class Post
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Tag> Tags { get; set; }
}

public class Tag
{
    public int Id { get; set; }
    public string Text { get; set; }
    public ICollection<Post> Posts { get; set; }
}
```

EF Core 5.0 recognizes this as a many-to-many relationship by convention, and automatically creates a `PostTag` join table in the database. Data can be queried and updated without explicitly referencing the join table, considerably simplifying code. The join table can still be customized and queried explicitly if needed.

For further information, [see the full documentation on many-to-many](xref:core/modeling/relationships/many-to-many).

## Split queries

Starting with EF Core 3.0, EF Core always generates a single SQL query for each LINQ query. This ensures consistency of the data returned within the constraints of the transaction mode in use. However, this can become very slow when the query uses `Include` or a projection to bring back multiple related collections.

EF Core 5.0 now allows a single LINQ query including related collections to be split into multiple SQL queries. This can significantly improve performance, but can result in inconsistency in the results returned if the data changes between the two queries. Serializable or snapshot transactions can be used to mitigate this and achieve consistency with split queries, but that may bring other performance costs and behavioral difference.

For example, consider a query that pulls in two levels of related collections using `Include`:

```csharp
var artists = await context.Artists
    .Include(e => e.Albums)
    .ToListAsync();
```

By default, EF Core will generate the following SQL when using the SQLite provider:

```sql
SELECT a."Id", a."Name", a0."Id", a0."ArtistId", a0."Title"
FROM "Artists" AS a
LEFT JOIN "Album" AS a0 ON a."Id" = a0."ArtistId"
ORDER BY a."Id", a0."Id"
```

With split queries, the following SQL is generated instead:

```sql
SELECT a."Id", a."Name"
FROM "Artists" AS a
ORDER BY a."Id"

SELECT a0."Id", a0."ArtistId", a0."Title", a."Id"
FROM "Artists" AS a
INNER JOIN "Album" AS a0 ON a."Id" = a0."ArtistId"
ORDER BY a."Id"
```

Split queries can be enabled by placing the new `AsSplitQuery` operator anywhere in your LINQ query, or globally in your model's `OnConfiguring`. For further information, [see the full documentation on split queries](xref:core/querying/single-split-queries).

## Simple logging and improved diagnostics

EF Core 5.0 introduces a simple way to set up logging via the new `LogTo` method. The following will cause logging messages to be written to the console, including all SQL generated by EF Core:

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder.LogTo(Console.WriteLine);
```

In addition, it is now possible to call `ToQueryString` on any LINQ query, retrieving the SQL that the query would execute:

```csharp
Console.WriteLine(
    ctx.Artists
    .Where(a => a.Name == "Pink Floyd")
    .ToQueryString());
```

Finally, various EF Core types have been fitted with an enhanced `DebugView` property which provides a detailed view into the internals. For example, <xref:Microsoft.EntityFrameworkCore.ChangeTracking.ChangeTracker.DebugView*?displayProperty=nameWithType> can be consulted to see exactly which entities are being tracked in a given moment.

For further information, [see the documentation on logging and interception](xref:core/logging-events-diagnostics/index).

## Filtered include

The `Include` method now supports filtering of the entities included:

```csharp
var blogs = await context.Blogs
    .Include(e => e.Posts.Where(p => p.Title.Contains("Cheese")))
    .ToListAsync();
```

This query will return blogs together with each associated post, but only when the post title contains "Cheese".

For further information, [see the full documentation on filtered include](xref:core/querying/related-data/eager#filtered-include).

## Table-per-type (TPT) mapping

By default, EF Core maps an inheritance hierarchy of .NET types to a single database table. This is known as table-per-hierarchy (TPH) mapping. EF Core 5.0 also allows mapping each .NET type in an inheritance hierarchy to a different database table; known as table-per-type (TPT) mapping.

For example, consider this model with a mapped hierarchy:

```csharp
public class Animal
{
    public int Id { get; set; }
    public string Name { get; set; }
}

public class Cat : Animal
{
    public string EducationLevel { get; set; }
}

public class Dog : Animal
{
    public string FavoriteToy { get; set; }
}
```

With TPT, a database table is created for each type in the hierarchy:

```sql
CREATE TABLE [Animals] (
    [Id] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NULL,
    CONSTRAINT [PK_Animals] PRIMARY KEY ([Id])
);

CREATE TABLE [Cats] (
    [Id] int NOT NULL,
    [EducationLevel] nvarchar(max) NULL,
    CONSTRAINT [PK_Cats] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Cats_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id]) ON DELETE NO ACTION,
);

CREATE TABLE [Dogs] (
    [Id] int NOT NULL,
    [FavoriteToy] nvarchar(max) NULL,
    CONSTRAINT [PK_Dogs] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Dogs_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id]) ON DELETE NO ACTION,
);
```

For further information, [see the full documentation on TPT](xref:core/modeling/inheritance).

## Flexible entity mapping

Entity types are commonly mapped to tables or views such that EF Core will pull back the contents of the table or view when querying for that type. EF Core 5.0 adds additional mapping options, where an entity can be mapped to a SQL query (called a "defining query"), or to a table-valued function (TVF):

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Post>().ToSqlQuery(
        @"SELECT Id, Name, Category, BlogId FROM posts
          UNION ALL
          SELECT Id, Name, ""Legacy"", BlogId from legacy_posts");

    modelBuilder.Entity<Blog>().ToFunction("BlogsReturningFunction");
}
```

Table-valued functions can also be mapped to a .NET method rather than to a DbSet, allowing parameters to be passed; the mapping can be set up with <xref:Microsoft.EntityFrameworkCore.RelationalModelBuilderExtensions.HasDbFunction*>.

Finally, it is now possible to map an entity to a view when querying (or to a function or defining query), but to a table when updating:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder
        .Entity<Blog>()
        .ToTable("Blogs")
        .ToView("BlogsView");
}
```

## Shared-type entity types and property bags

EF Core 5.0 allows the same CLR type to be mapped to multiple different entity types; such types are known as shared-type entity types. While any CLR type can be used with this feature, .NET `Dictionary` offers a particularly compelling use-case which we call "property bags":

```csharp
public class ProductsContext : DbContext
{
    public DbSet<Dictionary<string, object>> Products => Set<Dictionary<string, object>>("Product");

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.SharedTypeEntity<Dictionary<string, object>>("Product", b =>
        {
            b.IndexerProperty<int>("Id");
            b.IndexerProperty<string>("Name").IsRequired();
            b.IndexerProperty<decimal>("Price");
        });
    }
}
```

These entities can then be queried and updated just like normal entity types with their own, dedicated CLR type. More information can be found in the documentation on [property bags](xref:core/modeling/shadow-properties#property-bag-entity-types).

## Required 1:1 dependents

In EF Core 3.1, the dependent end of a one-to-one relationship was always considered optional. This was most apparent when using owned entities, as all the owned entity's column were created as nullable in the database, even if they were configured as required in the model.

In EF Core 5.0, a navigation to an owned entity can be configured as a required dependent. For example:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>(b =>
    {
        b.OwnsOne(e => e.HomeAddress,
            b =>
            {
                b.Property(e => e.City).IsRequired();
                b.Property(e => e.Postcode).IsRequired();
            });
        b.Navigation(e => e.HomeAddress).IsRequired();
    });
}
```

## DbContextFactory

EF Core 5.0 introduces `AddDbContextFactory` and `AddPooledDbContextFactory` to register a factory for creating DbContext instances in the application's dependency injection (D.I.) container; this can be useful when application code needs to create and dispose context instances manually.

```csharp
services.AddDbContextFactory<SomeDbContext>(b =>
    b.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=Test"));
```

At this point, application services such as ASP.NET Core controllers can then be injected with `IDbContextFactory<TContext>`, and use it to instantiate context instances:

```csharp
public class MyController : Controller
{
    private readonly IDbContextFactory<SomeDbContext> _contextFactory;

    public MyController(IDbContextFactory<SomeDbContext> contextFactory)
        => _contextFactory = contextFactory;

    public void DoSomeThing()
    {
        using (var context = _contextFactory.CreateDbContext())
        {
            // ...
        }
    }
}
```

For further information, [see the full documentation on DbContextFactory](xref:core/dbcontext-configuration/index#using-a-dbcontext-factory-eg-for-blazor).

## SQLite table rebuilds

Compared to other databases, SQLite is relatively limited in its schema manipulation capabilities; for example, dropping a column from an existing table is not supported. EF Core 5.0 works around these limitations by automatically creating a new table, copying the data from the old table, dropping the old table and renaming the new one. This "rebuilds" the table, and allows previously unsupported migration operations to be safely applied.

For details on which migration operations are now supported via table rebuilds, [see this documentation page](xref:core/providers/sqlite/limitations#migrations-limitations).

## Database collations

EF Core 5.0 introduces support for specifying text collations at the database, column or query level. This allows case sensitivity and other textual aspects to be configured in a way that is both flexible and does not compromise query performance.

For example, the following will configure the `Name` column to be case-sensitive on SQL Server, and any indexes created on the column will function accordingly:

```csharp
modelBuilder
    .Entity<User>()
    .Property(e => e.Name)
    .UseCollation("SQL_Latin1_General_CP1_CS_AS");
```

For further information, [see the full documentation on collations and case sensitivity](xref:core/miscellaneous/collations-and-case-sensitivity).

## Event counters

EF Core 5.0 exposes [event counters](https://devblogs.microsoft.com/dotnet/introducing-diagnostics-improvements-in-net-core-3-0) which can be used to track your application's performance and spot various anomalies. Simply attach to a process running EF with the [dotnet-counters](/dotnet/core/diagnostics/dotnet-counters) tool:

```console
> dotnet counters monitor Microsoft.EntityFrameworkCore -p 49496

[Microsoft.EntityFrameworkCore]
    Active DbContexts                                               1
    Execution Strategy Operation Failures (Count / 1 sec)           0
    Execution Strategy Operation Failures (Total)                   0
    Optimistic Concurrency Failures (Count / 1 sec)                 0
    Optimistic Concurrency Failures (Total)                         0
    Queries (Count / 1 sec)                                     1,755
    Queries (Total)                                            98,402
    Query Cache Hit Rate (%)                                      100
    SaveChanges (Count / 1 sec)                                     0
    SaveChanges (Total)                                             1
```

For further information, [see the full documentation on event counters](xref:core/logging-events-diagnostics/metrics#event-counters-legacy).

## Other features

### Model building

* Model building APIs have been introduced for easier configuration of [value comparers](xref:core/modeling/value-comparers).
* Computed columns can now be configured as [*stored* or *virtual*](xref:core/modeling/generated-properties#computed-columns).
* Precision and scale can now be configured [via the Fluent API](xref:core/modeling/entity-properties#precision-and-scale).
* New model building APIs have been introduced for [navigation properties](xref:core/modeling/relationships/navigations).
* New model building APIs have been introduced for fields, similar to properties.
* The .NET [PhysicalAddress](/dotnet/api/system.net.networkinformation.physicaladdress) and [IPAddress](/dotnet/api/system.net.ipaddress) types can now be mapped to database string columns.
* A backing field can now be configured via [the new `[BackingField]` attribute](xref:core/modeling/backing-field).
* Nullable backing fields are now allowed, providing better support for store-generated defaults where the CLR default isn't a good sentinel value (notable `bool`).
* A new `[Index]` attribute can be used on an entity type to specify an index, instead of using the Fluent API.
* A new `[Keyless]` attribute can be used to configure an entity type [as having no key](xref:core/modeling/keyless-entity-types).
* By default, [EF Core now regards discriminators as *complete*](xref:core/modeling/inheritance#table-per-hierarchy-and-discriminator-configuration), meaning that it expects to never see discriminator values not configured by the application in the model. This allows for some performance improvements, and can be disabled if your discriminator column might hold unknown values.

### Query

* Query translation failure exceptions now contain more explicit reasons about the reasons for the failure, to help pinpoint the issue.
* No-tracking queries can now perform [identity resolution](xref:core/querying/tracking#identity-resolution), avoiding multiple entity instances being returned for the same database object.
* Added support for GroupBy with conditional aggregates (e.g. `GroupBy(o => o.OrderDate).Select(g => g.Count(i => i.OrderDate != null))`).
* Added support for translating the Distinct operator over group elements before aggregate.
* Translation of [`Reverse`](/dotnet/api/system.linq.queryable.reverse).
* Improved translation around `DateTime` for SQL Server (e.g. [`DateDiffWeek`](xref:Microsoft.EntityFrameworkCore.SqlServerDbFunctionsExtensions.DateDiffWeek*), [`DateFromParts`](xref:Microsoft.EntityFrameworkCore.SqlServerDbFunctionsExtensions.DateFromParts*)).
* Translation of new methods on byte arrays (e.g. [`Contains`](/dotnet/api/system.linq.enumerable.contains), [`Length`](/dotnet/api/system.array.length), [`SequenceEqual`](/dotnet/api/system.linq.enumerable.sequenceequal)).
* Translation of some additional bitwise operators, such as two's complement.
* Translation of `FirstOrDefault` over strings.
* Improved query translation around null semantics, resulting in tighter, more efficient queries.
* User-mapped functions can now be annotated to control null propagation, again resulting in tighter, more efficient queries.
* SQL containing CASE blocks is now considerably more concise.
* The SQL Server [`DATALENGTH`](/sql/t-sql/functions/datalength-transact-sql) function can now be called in queries via the new [`EF.Functions.DataLength`](xref:Microsoft.EntityFrameworkCore.SqlServerDbFunctionsExtensions.DataLength*) method.
* `EnableDetailedErrors` adds [additional details to exceptions](xref:core/logging-events-diagnostics/simple-logging#detailed-query-exceptions).

### Saving

* SaveChanges [interception](xref:core/logging-events-diagnostics/interceptors#savechanges-interception) and [events](xref:core/logging-events-diagnostics/events).
* APIs have been introduced for controlling [transaction savepoints](xref:core/saving/transactions#savepoints). In addition, EF Core will automatically create a savepoint when `SaveChanges` is called and a transaction is already in progress, and roll back to it in case of failure.
* A transaction ID can be explicitly set by the application, allowing for easier correlation of transaction events in logging and elsewhere.
* The default maximum batch size for SQL Server has been changed to 42 based on an analysis of batching performance.

### Migrations and scaffolding

* Tables can now be [excluded from migrations](xref:core/modeling/entity-types#excluding-from-migrations).
* A new [`dotnet ef migrations list`](xref:core/cli/dotnet#dotnet-ef-migrations-list) command now shows which migrations have not yet been applied to the database ([`Get-Migration`](xref:core/cli/powershell#get-migration) does the same in the Package Management Console).
* Migrations scripts now contain transaction statements where appropriate to improve handling cases where migration application fails.
* The columns for unmapped base classes are now ordered after other columns for mapped entity types. Note this only impacts newly created tables; the column order for existing tables remains unchanged.
* Migration generation can now be aware if the migration being generated is idempotent, and whether the output will be executed immediately or generated as a script.
* New command-line parameters have been added for specifying namespaces in [Migrations](xref:core/managing-schemas/migrations/index#namespaces) and [scaffolding](xref:core/managing-schemas/scaffolding#directories-and-namespaces).
* The [dotnet ef database update](xref:core/cli/dotnet#dotnet-ef-database-update) command now accepts a new `--connection` parameter for specifying the connection string.
* Scaffolding existing databases now singularizes table names, so tables named `People` and `Addresses` will be scaffolded to entity types called `Person` and `Address`. [Original database names can still be preserved](xref:core/managing-schemas/scaffolding#preserving-names).
* The new [`--no-onconfiguring`](xref:core/cli/dotnet#dotnet-ef-dbcontext-scaffold) option can instruct EF Core to exclude `OnConfiguring` when scaffolding a model.

### Azure Cosmos DB

* [Azure Cosmos DB connection settings](xref:core/providers/cosmos/index#cosmos-options) have been expanded.
* Optimistic concurrency is now [supported on Azure Cosmos DB via the use of ETags](xref:core/providers/cosmos/index#optimistic-concurrency-with-etags).
* The new `WithPartitionKey` method allows the Azure Cosmos DB [partition key](xref:core/providers/cosmos/index#partition-keys) to be included both in the model and in queries.
* The string methods [`Contains`](/dotnet/api/system.string.contains), [`StartsWith`](/dotnet/api/system.string.startswith) and [`EndsWith`](/dotnet/api/system.string.endswith) are now translated for Azure Cosmos DB.
* The C# `is` operator is now translated on Azure Cosmos DB.

### Sqlite

* Computed columns are now supported.
* Retrieving binary and string data with GetBytes, GetChars, and GetTextReader is now more efficient by making use of SqliteBlob and streams.
* Initialization of SqliteConnection is now lazy.

### Other

* Change-tracking proxies can be generated that automatically implement [INotifyPropertyChanging](/dotnet/api/system.componentmodel.inotifypropertychanging) and [INotifyPropertyChanged](/dotnet/api/system.componentmodel.inotifypropertychanged). This provides an alternative approach to change-tracking that doesn't scan for changes when `SaveChanges` is called.
* A <xref:System.Data.Common.DbConnection> or connection string can now be changed on an already-initialized DbContext.
* The new <xref:Microsoft.EntityFrameworkCore.ChangeTracking.ChangeTracker.Clear*?displayProperty=nameWithType> method clears the DbContext of all tracked entities. This should usually not be needed when using the best practice of creating a new, short-lived context instance for each unit-of-work. However, if there is a need to reset the state of a DbContext instance, then using the new `Clear()` method is more efficient and robust than mass-detaching all entities.
* The EF Core command line tools now automatically configure the `ASPNETCORE_ENVIRONMENT` *and* `DOTNET_ENVIRONMENT` environment variables to "Development". This brings the experience when using the generic host in line with the experience for ASP.NET Core during development.
* Custom command-line arguments can be flowed into <xref:Microsoft.EntityFrameworkCore.Design.IDesignTimeDbContextFactory`1>, allowing applications to control how the context is created and initialized.
* The index fill factor can now be [configured on SQL Server](xref:core/providers/sql-server/indexes#fill-factor).
* The new <xref:Microsoft.EntityFrameworkCore.RelationalDatabaseFacadeExtensions.IsRelational*> property can be used to distinguish when using a relational provider and a non-relation provider (such as in-memory).
