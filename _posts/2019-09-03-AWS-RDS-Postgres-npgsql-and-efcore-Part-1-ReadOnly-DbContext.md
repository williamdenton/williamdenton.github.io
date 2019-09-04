---
layout: post
title:  "AWS RDS Postgres, npgsql and efcore - Part 1 - ReadOnly DbContext"
date:   2019-09-03T21:58:00+12:00
modified: 2019-09-05T11:45:00+12:00
categories: Blog npgsql efcore dotnet aws
socialShareImage: /assets/images/2019-09/2019-09-03-postgres-dotnet-aws.png
cover: /assets/images/2019-09/2019-09-03-postgres-dotnet-aws.png
disqus: true
description: "When using Amazon Aurora RDS you typically configure your database cluster to have two or more nodes. If something bad were to happen to one of the nodes RDS automatically fails over to one of the remaining healthy nodes.

Now you have a resilient database, but you are paying for servers that are sitting idle. Designing your application so you can leverage the read only replicas for running queries means you have a more resilient and scalable application than if you only make use of the master read write node.

EfCore doesn't make it easy to change the connection string in a DbContext..."
---

This is the first part in a series on using dotnet core with Amazon RDS for Postres. There is a [github repo with a sample project](https://github.com/williamdenton/AwsRdsPostgresDemo) that I'll be pulling code samples from. 

This series will cover:
1. EfCore ReadOnly Database Context (this post)
2. Configuring a local dev postgres instance to emulate RDS
3. Authenticating to RDS using IAM
4. RDS Failure modes with dotnet
5. DbContext gotchas with `IHostedService` and dependency injection

# Part 1 - EfCore ReadOnly Database Context
When using [Amazon Aurora RDS](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html) you typically configure your database cluster to have two or more nodes. If something bad were to happen to one of the nodes RDS automatically fails over to one of the remaining healthy nodes.

Now you have a resilient database, but you are paying for servers that are sitting idle. Designing your application so you can leverage the read only replicas for running queries means you have a more resilient and scalable application than if you only make use of the master read write node.

EfCore doesn't make it easy to change the connection string in a DbContext, and exposing the same DbContext in both Read only and read write modes will confuse your team and they will likely default to using the read write context as it does everything.

A `DbContext` typically has one or more `DbSet<Table>` representing your relations. In a read only situation, it makes no sense to use `DbSet`, an `IQueryable` expresses your read only intent in a much clearer way.

## Configuring DbContext
In this example I use an abstract base DbContext, from this there are three concrete types that have little in the way of implementation, they serve only to allow the DI Container to configure each with a different connection string.

```
Abstract DemoDbContext
  -> MigrationDemoDbContext
  -> ReadWriteDemoDbContext
  -> ReadOnlyDemoDbContext
```

You could also choose to set this up by combining the Migration and ReadWrite contexts, then inheriting that to get the ReadOnly.

```
DemoDbContext
  -> ReadOnlyDemoDbContext
```

The interesting part is defining the interfaces that each has. When using dependency injection you don't depend on the concrete type, but on an interface it implements.


### Read Write DbContext
This exposes the essential parts of your db context to your application. You need a `DbSet` for each relation you have and the usual `SaveChanges()` so you can commit your changes to the DB. This should look like every other DbContext Interface you've seen before.
```cs
interface IDemoReadWriteDbContext {
    DbSet<Customer> Customers { get; }
    int SaveChanges(bool acceptAllChangesOnSuccess);
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
    Task<int> SaveChangesAsync(bool acceptAllChangesOnSuccess, CancellationToken cancellationToken = default);
}
```

### ReadOnly DbContext

In the read only context we expose the Customer entity not as a `DbSet`, but an `IQueryable`. Now when you inject this interface into your app your intent to only allow queries is much clearer.

```cs
interface IDemoReadOnlyDbContext {
    IQueryable<Customer> Customers { get; }
}
```

You may have spotted that both read only and read write interfaces have a property named `Customers` but with differing/conflicting types. This means when you come to implement the Interface _and_ inherit the DbContext you'll have a warning.

> 'DemoReadOnlyDbContext.Customers' hides inherited member 'DemoDbContext.Customers'. 
> Use the new keyword if hiding was intended. 

To resolve this you should implement the interface [explicitly](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/interfaces/explicit-interface-implementation). 

_Alternatively, as the error suggests, you could use the [`new`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/new-modifier) keyword to hide the base implementation but this not as convenient to use when it comes to testing._

```cs
public class DemoReadOnlyDbContext : DemoDbContext, IDemoReadOnlyDbContext {
    IQueryable<Customer> IDemoReadOnlyDbContext.Customers => base.Customers.AsQueryable();
}
```

Now when you inject `IDemoReadOnlyDbContext` it will use the `IQueryable` as intended. The IQueryable is obtained from the inherited DbContexts DbSet.

```cs
var ctx = new DemoReadOnlyDbContext();
ctx.Customer... // <-- this is DbSet
```

```cs
var ctx = (IDemoReadOnlyDbContext) new DemoReadOnlyDbContext() ;
ctx.Customer...// <-- this is IQueryable
```

Explicitly implementing the interface has the desirable side effect of making it easier to test components that depend on the `IDemoReadOnlyDbContext`. We can construct a concrete `DemoReadOnlyDbContext` populate any data the test requires into the inherited DbSet in-memory (without using `SaveChanges()`) then cast it to `IDemoReadOnlyDbContext` to query that data in our test.


#### Disabling Tracking
For read only queries, there is no need for entity framework to be tracking the objects you select and holding them in memory. Tracking allows Entity Framework to be able to write changes back to the database when you `SaveChanges()`. _Thanks to Julie Lerman for point this out in the comments._
```cs
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) {
	base.OnConfiguring(optionsBuilder);
	optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}
```

You can also disable tracking on a per table basis with the `AsNoTracking()` extension method in other situations outside of the readonly context where you dont need it.

```cs
IQueryable<Customer> IDemoReadOnlyDbContext.Customers => base.Customers.AsNoTracking();
```

## Configuring DbContext for Dependency Injection

Each DbContext is configured with a different connection string, targeting either the primary read wite node, or the read only replica. However you want all other options for the DbContext to be the same.

First up we need to grab the connection strings from config
```cs
var dbMigrator = configuration.GetConnectionString("DemoDbContextMigrator")
var dbWrite = configuration.GetConnectionString("DemoDbContextReadWrite")
var dbRead = configuration.GetConnectionString("DemoDbContextReadOnly")
```

Now we are ready to configure the db contexts in exactly the same way except for the connection string
```cs
services.AddEntityFrameworkNpgsql()
    .AddDbContext<DemoMigratorDbContext>(ef => ConfigureDbContextOptions(ef, dbMigrator))
    .AddDbContext<DemoReadWriteDbContext>(ef => ConfigureDbContextOptions(ef, dbWrite))
    .AddDbContext<DemoReadOnlyDbContext>(ef => ConfigureDbContextOptions(ef, dbRead));

// local setup function for our db contexts above
static void ConfigureDbContextOptions(DbContextOptionsBuilder efOptions, string connectionString) {
    efOptions.UseNpgsql(connectionString, npgsqlOptions => {
        npgsqlOptions.UseNodaTime();
    });
}
```

Now expose the ReadOnly and ReadWrite DbContexts via their interfaces to your application.
```cs
services.AddScoped<IDemoReadWriteDbContext>(provider => provider.GetService<DemoReadWriteDbContext>());
services.AddScoped<IDemoReadOnlyDbContext>(provider => provider.GetService<DemoReadOnlyDbContext>());
```

You can see this code [in situ in the sample app](https://github.com/williamdenton/AwsRdsPostgresDemo/blob/49cda6bd0d004ab0be5b8c1a942fcd0520dcd112/src/AwsRdsPostgresDemo/Program.cs#L50-L70).

One critical detail to make this work; The abstract base must have a constructor that takes a non-generic `DbContextOptions`. Each concrete DbContext has a constructor that requires a generic `DbContextOption<T>` where `T` is the type of the DbContext. This  object contains the connection string and other configuration. It must be passed to the base to successfully configure a DbContext.

```cs
public abstract class DemoDbContext : DbContext {
    protected DemoDbContext(DbContextOptions options) : base(options) { }
}

public class DemoReadWriteDbContext : DemoDbContext, IDemoReadWriteDbContext {
    public DemoReadWriteDbContext(DbContextOptions<DemoReadWriteDbContext> options)
        : base(options)
    { }
}
```

### Wrap up
Using a read only DbContext means your application will be more scalable and utilize db instances in AWS that might otherwise sit idle. Your application can scale its read load across the RDS cluster meaning you might need less powerful instances saving you money on your aws bill. Your application will be more resilient to failure and easier to reason about by segregating your read only and read write workloads.

I covered how to architect your application to have separate interfaces for read only and read write and how to configure your DbContext using the standard dotnet core services collection.

The code samples in this post are available in full on github [https://github.com/williamdenton/AwsRdsPostgresDemo](https://github.com/williamdenton/AwsRdsPostgresDemo)

