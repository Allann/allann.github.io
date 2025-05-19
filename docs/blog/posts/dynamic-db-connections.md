# Dynamic Connection Strings in EF Core 9 Minimal APIs

## Altering a `DbContext` Connection at Runtime

In Entity Framework Core (including EF Core 9.x), the database connection is normally configured when the `DbContext` is
constructed, and it isn’t intended to be changed afterward. Once a `DbContext` is injected (e.g. via DI in a minimal
API), its `DbContextOptions` (including the connection string) are essentially fixed. There is no built-in method to
*reconfigure* the context’s connection string after it’s been created in the DI container. In other words, you **cannot
directly “swap out” the connection string of an existing context instance** once it’s been configured.

That said, EF Core does provide an **advanced feature** to support dynamic connections *if done before first use*. EF
Core’s relational providers (SQL Server, PostgreSQL, etc.) allow you to register a context *without initially specifying
a connection string*, then supply it at runtime. For example, `UseSqlServer` (and other `Use*` calls) have an overload
that *omits* the connection string, in which case **you must set it later** before using the context. EF Core exposes
the `DatabaseFacade.SetConnectionString()` extension method for this purpose. In practice, this means:

* You would configure the `DbContext` with the provider but **no connection string** at startup. For instance:
  `builder.Services.AddDbContext<MyContext>(opt => opt.UseSqlServer());` (calling the parameterless `UseSqlServer`
  overload). This registers the context *without a concrete connection string*.
* Then, **at runtime (before any database operations)**, you can set the actual connection string on the context
  instance. For example:

  ```csharp
  // Inside your endpoint or service, before using the context:
  context.Database.SetConnectionString(actualConnectionString);
  // Now you can use context (queries, SaveChanges, etc.)
  ```

This approach will dynamically point that context instance to the given connection string. **However, caution is
required:** you must call `SetConnectionString` *before* the context is used to connect to the database (i.e., before
any query or `SaveChanges` call). If the context has already been used (or was configured with a specific connection
string initially), changing it at runtime is unsupported. In summary, EF Core technically allows dynamic assignment of
the connection *on a fresh context*, but **you cannot retroactively change** an already-initialized connection string
after the context has been used.

## Supported Patterns for Dynamic Connection Strings

Given the constraints above, the recommended solution is to **provide the correct connection string when the `DbContext`
is created**, rather than trying to alter it afterward. There are several patterns to achieve this in a .NET 9 Minimal
API:

### 1. Use a DbContext Factory or Manual Context Creation

One option is to avoid injecting the `DbContext` directly, and instead inject a factory that can create `DbContext`
instances on the fly with the desired connection string. EF Core provides `IDbContextFactory<T>` via
`AddDbContextFactory`, or you can implement your own factory. For example, a custom factory interface and implementation
might look like:

```csharp
public interface ITenantDbContextFactory<TContext> where TContext : DbContext
{
    TContext Create(string databaseName);
}

public class Dcms3DbContextFactory : ITenantDbContextFactory<Dcms3DbContext>
{
    public Dcms3DbContext Create(string databaseName)
    {
        // Build new options with the given database name in the connection string
        var optionsBuilder = new DbContextOptionsBuilder<Dcms3DbContext>();
        optionsBuilder.UseSqlServer($"Server=...;Database={databaseName};TrustServerCertificate=True;");
        return new Dcms3DbContext(optionsBuilder.Options);
    }
}
```

This factory constructs a new `Dcms3DbContext` with a connection string targeting the specified database (the rest of
the connection details can be fixed). The calling code (e.g. an endpoint handler) would request `Dcms3DbContextFactory`
from DI and use it to create a context for the current request. This pattern is essentially what one Stack Overflow
answer suggested for multi-database scenarios. For example, in a minimal API endpoint:

```csharp
app.MapGet("/data/{dbName}", async (string dbName, Dcms3DbContextFactory factory) =>
{
    await using var db = factory.Create(dbName);
    var results = await db.MyEntities.ToListAsync();
    return Results.Ok(results);
});
```

Here, the context is created at request time with the appropriate connection string. **Note:** If you use
`IDbContextFactory<T>` via `AddDbContextFactory`, it’s typically registered as a singleton by default. In a dynamic
scenario, you may still need to call `SetConnectionString` on the created context (since `AddDbContextFactory` usually
uses a fixed configuration). Alternatively, you can register the factory as *scoped* and supply the dynamic connection
inside the factory as shown above. In either case, you are responsible for disposing of the context instance (as shown
with `await using var db = ...`).

### 2. Configure DbContext per request via DI (Scoped Configuration)

Another approach is to leverage the dependency injection configuration to supply the connection string based on some
*scoped* context (such as the current HTTP request or tenant). You can use the overload of `AddDbContext` that provides
the `IServiceProvider` to build options. This allows you to retrieve information from other services (like configuration
or HTTP context) each time a `DbContext` is created. For example, in *Program.cs*:

```csharp
builder.Services.AddHttpContextAccessor(); // enable accessing HttpContext in DI

builder.Services.AddDbContext<Dcms3DbContext>((serviceProvider, options) =>
{
    // Get current HTTP context, route values, etc.
    var httpContext = serviceProvider.GetRequiredService<IHttpContextAccessor>().HttpContext;
    var dbName = httpContext?.Request.RouteValues["dbName"] as string;
    // Build the connection string dynamically (example assumes a base template)
    string baseConn = configuration.GetConnectionString("BaseTemplate"); // e.g. "Server=...;Database={0};...;"
    var connectionString = string.Format(baseConn, dbName);
    options.UseSqlServer(connectionString);
});
```

In this example, whenever `Dcms3DbContext` is requested, the DI container will execute our factory lambda: it grabs the
current request’s `dbName` (perhaps from the URL or headers) and configures the `DbContextOptions` with the correct
connection string. **This effectively gives each HTTP request a context tied to its specific database.** Felipe
Gavilán’s blog illustrates this pattern using an HTTP header to convey a tenant ID, and the Code Maze series provides a
similar example using a custom `IDataSourceProvider` service. In either case, the key is that the connection string
comes from a scoped service or request data rather than being hard-coded at startup.

This pattern requires that you have access to the necessary context (like route values, a JWT claim, or a header) by the
time the `DbContext` is being constructed. In minimal APIs, route parameters are available in
`HttpContext.Request.RouteValues`. Using an `IHttpContextAccessor` (registered as singleton) is a straightforward way to
get this information inside the `AddDbContext` lambda. Alternatively, you could set up a dedicated scoped service (e.g.
`ITenantService`) earlier in the pipeline (via middleware or an endpoint filter) that stores the chosen database name
for the request, and then have the `DbContext` configuration read from that service. This approach keeps your
`DbContext` registration clean, e.g.:

```csharp
builder.Services.AddScoped<ITenantService, TenantService>();
builder.Services.AddDbContext<Dcms3DbContext>((sp, options) =>
{
    var tenantService = sp.GetRequiredService<ITenantService>();
    string connStr = tenantService.GetCurrentTenantConnectionString();
    options.UseSqlServer(connStr);
});
```

In this case, `TenantService` would determine the current database (perhaps using the current user info or route data)
and provide the appropriate connection string. The **official EF Core documentation** for multi-tenancy demonstrates
this pattern: the context can accept an `ITenantService` (and perhaps `IConfiguration`) via its constructor, and use
that in `OnConfiguring` to choose the connection string. The context is then added via `AddDbContextFactory` or
`AddDbContext` as a *scoped* or *transient* service so that each request/tenant evaluation is fresh.

### 3. Use OnConfiguring with Injected Configuration (Alternative)

As mentioned above, you can also implement dynamic connection logic inside your `DbContext` class itself by overriding
`OnConfiguring`. If your context’s constructor has access to something like `IConfiguration` and a tenant identifier,
you can call the appropriate `UseSqlServer` (or other provider) within `OnConfiguring`. For example:

```csharp
public class Dcms3DbContext : DbContext
{
    private readonly string _connection;
    public Dcms3DbContext(DbContextOptions<Dcms3DbContext> opts, IConfiguration config, ITenantService tenantSvc)
        : base(opts)
    {
        // Build the connection string using the current tenant info
        var tenantId = tenantSvc.TenantId;
        _connection = config.GetConnectionString(tenantId); 
    }
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!string.IsNullOrEmpty(_connection))
            optionsBuilder.UseSqlServer(_connection);
    }
}
```

In this setup, the context is still created per request (e.g. via a context factory or regular DI), and whenever it’s
constructed, it grabs the current tenant’s identifier and finds the matching connection string from configuration. The
`OnConfiguring` ensures the context uses that connection. This pattern achieves the same result – each `DbContext`
instance is configured for the appropriate database – but keeps the logic inside the context class. The downside is that
you must ensure the extra services (`IConfiguration`, `ITenantService`) are *available* to the context (which often
means using `AddDbContextFactory` with a scoped lifetime or using constructor injection with `AddDbContext` and
explicitly passing those services in). The EF Core team’s guidance notes that if a user can *switch* tenants within the
same session or scope, you might need to register the context as transient to avoid caching the old connection string.
In typical request-based scoping, this isn’t an issue (each request gets a new context anyway).

### 4. Multiple `DbContext` registrations (not typical for this scenario)

For completeness, if the set of possible databases is known and small, some applications simply register multiple
contexts (one per connection string) and choose between them. However, in your scenario (a single `DbContext` type,
database name only known at request time, no multi-schema), that’s not an ideal solution. It’s better to use one of the
dynamic approaches above rather than duplicating context types or manually choosing between many DI registrations.

## Best Practices for Dynamic Connection Scenarios in EF Core 9

When implementing dynamic connection strings, keep these best practices in mind:

* **Provide the connection string as early as possible:** Ideally, configure the `DbContext` with the correct connection
  string at creation time (per request). This avoids any need to change it afterward. Use factories or DI patterns to
  supply the string based on the current context (tenant, user, etc.) before the context is used.

* **Use scoped or transient lifetimes appropriately:** For web apps, a scoped lifetime (per HTTP request) is usually
  appropriate for `DbContext`. If there’s a possibility a user will **change** the target database mid-session and you
  want the same user session to fetch a new database, consider using a transient context or creating new context
  instances as needed. Do not reuse the same context instance for different databases.

* **Avoid global mutable state:** Don’t store the “current connection string” in a static or singleton that is modified
  per request without proper scoping. For example, the Code Maze example uses a singleton `DataSourceProvider` with a
  `CurrentDataSource` property, but notes this must be carefully managed (and per-user in multi-user scenarios). A safer
  approach is to keep tenant-specific info in scoped services or the `HttpContext`. This ensures threads or concurrent
  requests don’t interfere with each other’s settings.

* **Dispose of contexts properly:** When you manually create contexts (via a factory or `new DbContext(options)`), be
  sure to dispose of them after use (e.g. use `using`/`await using` blocks or let the DI scope handle disposal if the
  context is resolved from the container). Each context/connection should be cleaned up after the request/unit-of-work
  ends.

* **Be mindful of connection pooling and performance:** If you use **DbContext pooling** (`AddDbContextPool` or
  `AddPooledDbContextFactory`), be aware that pooled contexts might retain state (including an open connection or a set
  connection string). Pooling is generally not recommended for dynamic connection scenarios because a context from the
  pool might still be tied to a previous connection. Stick to regular scoped contexts or your own factories so that each
  context starts fresh with the correct connection. If performance becomes a concern, measure it – EF Core is designed
  to create contexts quickly, and per-request creation is usually fine.

* **Secure the tenant selection:** Since the database is chosen at runtime, ensure that the mechanism for selecting the
  database is secure and validated. For example, if you pass a database name via an API route or header, validate that
  the caller is authorized for that database and that the name is valid. Avoid directly concatenating untrusted input
  into connection strings without checks (to prevent connection string injection or accidental exposure of other
  databases).

* **Follow EF Core updates:** EF Core (including v9) continues to improve support for multi-tenant scenarios. Keep an
  eye on official docs and release notes. The official multi-tenancy guidance (for EF Core 7/8+) provides patterns that
  are still applicable in EF Core 9. While EF Core doesn’t have a built-in multi-tenant manager, the combination of the
  techniques above (scoped config, context factories, `SetConnectionString`, etc.) is the supported way to go.

By using these patterns, you can handle a scenario where the database name is determined at request processing time. The
**preferred** approach is to create a new `DbContext` (or configure one via DI) with the proper connection string **per
request**, rather than trying to mutate an existing injected context. This aligns with EF Core’s unit-of-work pattern
and ensures each context instance talks to the correct database.

**Sources:**

* Microsoft Docs – *EF Core Multi-Tenancy (Multiple Databases)*
* Microsoft Docs – *DbContext Configuration & Lifetime* (for using `AddDbContextFactory` and DI)
* Code Maze – *Dynamically Switching DbContext at Runtime* (dynamic connection via DI)
* Stack Overflow – *EF Core: dynamic connection string per tenant* (custom factory example)
