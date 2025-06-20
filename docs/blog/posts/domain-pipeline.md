---
title: Domain Pipelines
date: 2025-05-21
authors: [allann]
categories:
  - dotnet
  - minimal api
tags: [dotnet, csharp, pipeline, middleware, domain]
---

# Building Domain-Specific “Russian-Doll” Pipelines in .NET 9 – a Functional Approach with **ProblemOr**

> *Move the elegance of ASP.NET Core middleware into any complex back-end workflow – without `HttpContext`, without OOP
builders, and without losing DI, cancellation or early-exit behaviour.*

## Why a pipeline?

Business workflows such as **Get GPS Logs** often involve many steps:

1. Classify the request (Site vs Council, Commercial vs Domestic).
2. Fetch way-points from a repository.
3. Convert timestamps to local time.
4. Constrain results to a bounding-box.
5. Materialise the response.

Each step should be **independent, replaceable, testable** and able to **short-circuit** on error or empty data – the
classic “Russian-doll” pattern that ASP.NET Core middleware delivers for web traffic.

<!-- more -->

## Key design choices

| Decision                                       | Rationale                                                                          |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Immutable context record** (`GpsLogContext`) | No hidden state; easy to `with`-clone for functional purity.                       |
| **Discriminated-union result** (`ProblemOr<T>`)  | One return type covers success *or* one-or-many errors; consumers call `Match`.    |
| **Pure functions + static “compose” helper**   | No builder classes or interfaces; the pipeline itself *is* a first-class function. |
| **DI scope passed in the context**             | Middleware still resolve scoped services, but the composer stays DI-agnostic.      |

## The core primitives

```csharp
using ProblemOr;

public sealed record GpsLogContext(
    RequestModel      Request,
    ResponseModel?    Response,
    IServiceScope     Scope,
    CancellationToken Ct);

// Pipeline delegate (mirrors aspnetcore RequestDelegate)
public delegate ValueTask<ProblemOr<GpsLogContext>> GpsLogDelegate(GpsLogContext ctx);
```

## A *functional* static builder

```csharp
public static class DomainPipeline
{
    private static readonly GpsLogDelegate Terminal =
        ctx => ValueTask.FromResult<ProblemOr<GpsLogContext>>(ctx);

    public static GpsLogDelegate Compose(
        params Func<GpsLogDelegate, GpsLogDelegate>[] parts)
    {
        var app = Terminal;
        for (var i = parts.Length - 1; i >= 0; i--)
            app = parts[i](app);   // Russian-doll wrap
        return app;
    }
}
```

`Compose` returns *one* `GpsLogDelegate`; there is no stateful builder instance to maintain or mock.

## Middleware components as pure functions

```csharp
// Request classifier
static Func<GpsLogDelegate, GpsLogDelegate> RequestClassifier =>
    next => async ctx =>
    {
        if (!ctx.Request.TryDetermineFlags())
            return Error.Validation(code: "BadType", description: "Unknown request");

        return await next(ctx);
    };

// Way-point fetcher (shows early error exit)
static Func<GpsLogDelegate, GpsLogDelegate> WaypointFetcher =>
    next => async ctx =>
    {
        var repo = ctx.Scope.ServiceProvider.GetRequiredService<IWaypointRepository>();
        var points = await repo.GetAsync(ctx.Request.SiteNumber, ctx.Ct);

        if (points.Count == 0)
            return Error.NotFound(description: "No way-points");

        var updated = ctx with { Request = ctx.Request with { Waypoints = points } };
        return await next(updated);
    };

// Time-zone resolver
static Func<GpsLogDelegate, GpsLogDelegate> TimezoneResolver =>
    next => async ctx =>
    {
        var tz = ctx.Scope.ServiceProvider.GetRequiredService<ITimezoneService>();
        var local = await tz.ToLocalAsync(ctx.Request.UtcTime, ctx.Ct);

        var updated = ctx with { Request = ctx.Request with { LocalTime = local } };
        return await next(updated);
    };

// Bounding-box filter
static Func<GpsLogDelegate, GpsLogDelegate> BoundingBoxFilter =>
    next => async ctx =>
    {
        var updated = ctx with { Request = ctx.Request.FilterToBoundingBox() };
        return await next(updated);
    };

// Response builder – terminal step, always succeeds
static Func<GpsLogDelegate, GpsLogDelegate> ResponseBuilder =>
    _ => ctx =>
    {
        var response = ResponseModel.Create(ctx.Request);
        return ValueTask.FromResult<ProblemOr<GpsLogContext>>(ctx with { Response = response });
    };
```

Add or remove steps at will; order is controlled exclusively by the `Compose` call.

## Wiring the pipeline in *Program.cs* (minimal-API style)

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using ProblemOr;

using static DomainPipeline;

var builder = WebApplication.CreateBuilder(args);

builder.Services
       .AddScoped<IWaypointRepository, WaypointRepository>()
       .AddScoped<ITimezoneService, TimezoneService>();

// Build the single function at start-up
GpsLogDelegate gpsPipeline = Compose(
    RequestClassifier,
    WaypointFetcher,
    TimezoneResolver,
    BoundingBoxFilter,
    ResponseBuilder);

var app = builder.Build();

app.MapPost("/gpslogs", async (RequestModel req, IServiceProvider sp, CancellationToken ct) =>
{
    await using var scope = sp.CreateAsyncScope();
    var seed = new GpsLogContext(req, null, scope, ct);

    var result = await gpsPipeline(seed);

    return result.Match(
        ok     => Results.Ok(ok.Response),
        errors => Results.Problem(title: "GPS log error",
                                  statusCode: 400,
                                  detail: string.Join(" | ", errors.Select(e => e.Description))));
});

await app.RunAsync();
```

* **DI scope** is created per request, keeping repository and service lifetimes correct.
* All middleware run **in-memory**; no `HttpContext`, no Kestrel overhead.
* Early failures propagate as a single 400 response with an aggregated error message.

## Unit-testing a component (xUnit + NSubstitute)

```csharp
[Fact]
public async Task WaypointFetcher_returns_NotFound_when_empty()
{
    // Arrange
    var repo = Substitute.For<IWaypointRepository>();
    repo.GetAsync("05", Arg.Any<CancellationToken>()).Returns([]);

    var services = new ServiceCollection().AddSingleton(repo).BuildServiceProvider();
    await using var scope = services.CreateAsyncScope();

    var ctx  = new GpsLogContext(new("05"), null, scope, default);

    // Act
    var result = await WaypointFetcher(_ => throw new Exception("Next should not run"))(ctx);

    // Assert
    Assert.True(result.IsError);
    Assert.Equal("No way-points", result.FirstError.Description);
}
```

Because each middleware is a pure function, testing entails *no* test-server fixtures.

## Performance and scaling notes

* **No reflection** – the pipeline is a pre-composed delegate chain.
* **Zero allocations per step** – except when `with` creates a new record instance (which is unavoidable for
  immutability).
* **Parallel runs** – the delegate is thread-safe; every invocation receives its own `GpsLogContext`.

If you need to reuse *singleton* configuration inside a component, capture it when you declare the lambda:

```csharp
var staticOptions = builder.Configuration.GetSection("Gps").Get<GpsOptions>();
Func<GpsLogDelegate, GpsLogDelegate> ConfigInjector =
    next => ctx => next(ctx with { Request = ctx.Request with { Options = staticOptions } });
```

## Take-aways

* **Functional composition** is enough – no builder objects, no interfaces.
* **ProblemOr<T>** gives strongly-typed early exits; no boolean flags or exceptions are required for control-flow.
* **DI remains first-class** because each middleware receives the current scope from the context.
* The pattern *mirrors ASP.NET Core middleware* closely, so new team-members recognise the mental model instantly.

### Ready-to-paste code

All code above fits into three small files:

| File                 | Contents                                                  |
| -------------------- |-----------------------------------------------------------|
| **Pipeline.cs**      | Static `Pipeline.Compose` and the `GpsLogDelegate` alias. |
| **GpsLogContext.cs** | Immutable record + supporting models.                     |
| **Middleware.cs**    | The five lambda components shown.                         |

Drop them into any .NET 9 project and enjoy web-grade pipeline composability inside your domain logic — no custom
frameworks, no accidental complexity.
