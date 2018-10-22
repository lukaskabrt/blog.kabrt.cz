Title: Modular MVC application - building DBContext on the fly
Published: 03/07/2012
Tags: [Entity framework, ASP.NET MVC]
---

## Background
We are using Entity Framework in our solutions and for this project we'd like to take advantage of new features of the Entity Framework, especially Code First development and data migrations.

Any module in our application might want to save some data to the database - in order to simplify communication with database, the application core provides bunch of classes responsible for wiring up all the stuff like connection strings, entity configurations, etc. With that infrastructure in place, we can use the same, simple code to access data anywhere in our application ... and keep in mind that our domain model is spread across multiple modules and possibly across multiple assemblies. How have archived that?

## Implementation
If you are using Entity framework Code First, the most of the actions involve `DbContext` class. It provides methods for adding new entities, retrieving existing ones from database, saving changes and much more. Typically you create derived class with properties of `DbSet<TEntity>` type to expose collections of entities and EF will do all the work.

```csharp
public class GuestbookContext : DbContext {
    DbSet<Comment> Comments { get; set; }
}
```

Entity framework internally uses a conceptual model to map entities and tables. In order to create it EF needs types of entities to include in the model. It the previous example it's done by exposing `DbSet<Comment>`, but this solution isn't applicable in our case because we don't know types of the entities types at compile time. Fortunately there are other ways - the one we have chosen uses `DbModelBuilder` class and creating `DbContext` from the model produces by the builder. The whole process is encapsulated in the `DbContextBuilder.Build` method.

```csharp
public DbContext Build(DbContextConfig config) {
    var factory = DbProviderFactories.GetFactory(config.ConnectionString.ProviderName);
    var connection = factory.CreateConnection();
    connection.ConnectionString = config.ConnectionString.ConnectionString;

    if (config.Model == null) {
        DbModelBuilder builder = new DbModelBuilder();
        foreach (var entity in config.EntityCatalog) {
            entity.Configure(builder);
        }

        config.Model = builder.Build(connection).Compile();
    }

    return new DbContext(connection, config.Model, true);
}
```

The `DbContext` class contains data neccessary to build and configure `DbContext`.

```csharp
public class DbContextConfig {
    public ConnectionStringSettings ConnectionString { get; set; }
    public IEnumerable<IEntityConfig> EntityCatalog { get; set; }
    internal DbCompiledModel Model { get; set; }
}
```

`EntityCatalog` property contains a collection of `IEntityConfig` objects, that instructs `DbModelBuilder` which entities should be included in the model and how these entities should be mapped to the database. To avoid any manual use of reflection or manual registration of the modules, `EnityCatalog` property is populated by MEF.

Along with model classes every module includes a configuration class that implements `IEntityConfig` interface and is used to setup `DbModelBuilder`, if denoted with Export attribute, this class is automatically discovered by MEF.

```csharp
public class Comment {
    public int ID { get; set; }
    public string Author { get; set; }
    public string Text { get; set; }
}

[Export(typeof(IEntityConfig))]
public class CommentConfig : IEntityConfig {
    public void Configure(DbModelBuilder builder) {
        builder.Entity<Comment>();
    }
}
```

Using MEF to find all `IEntityConfig` classes: 

```csharp
public static IEnumerable<IEntityConfig> DiscoverEntities(Assembly assembly) {
    var configuration = new ContainerConfiguration().WithAssembly(assembly);

    using (var container = configuration.CreateContainer()) {
        return container.GetExports<IEntityConfig>();
    }
}
```

It is almost finished. The only task left is to initialize the `DbContextBuilder` application at application start-up. 

```csharp
var contextConfig = new DbContextConfig();
    contextConfig.ConnectionString = ConfigurationManager.ConnectionStrings["DomainModelDb"];
    contextConfig.EntityCatalog = DiscoverEntities(Assembly.GetExecutingAssembly());

DbContextBuilder.Instance.AddContextConfiguration(contextConfig);
```

For the sake of simplicity this example loads model class only from the current assembly. but trust me it's working with multiple assemblies as well. 

## Usage
`DbContextBuilder` class implements the singleton pattern (I know I could have done better, but it's prototype ...)  so its instance is available across application. In my controllers I can instantiate a new instance of the `DbContext` by calling `DbContextBuilder.Instance.Build()`.

```csharp
public ActionResult Index() {
    using (var ctx = DbContextBuilder.Instance.Build()) {
        var comments = ctx.Set<Comment>().ToList();
        return View(comments);
    }
}
```

The sample project with complete source code can be download from my [Bitbucket repository](https://bitbucket.org/LukasKabrt/kabrt.blog.dbcontextbuilder).