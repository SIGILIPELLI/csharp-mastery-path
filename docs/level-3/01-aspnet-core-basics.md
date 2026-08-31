# 01 · ASP.NET Core Basics

ASP.NET Core is Microsoft's cross-platform web framework. This module covers
the minimal API style — the fastest way to stand up an HTTP service — before
Level 4 goes deeper into MVC controllers, auth, and gRPC.

## The smallest possible app

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello, ASP.NET Core!");

app.Run();
```

`dotnet new web -o HelloApi && cd HelloApi && dotnet run` starts a Kestrel
server (by default on `http://localhost:5000`-ish, printed to the console).
`WebApplicationBuilder` configures services and middleware; `Build()`
produces the `WebApplication`; `Run()` blocks, serving requests until the
process is stopped.

## Routing and route parameters

```csharp
app.MapGet("/hello/{name}", (string name) => $"Hello, {name}!");

app.MapGet("/add/{a:int}/{b:int}", (int a, int b) => new { a, b, sum = a + b });
```

`{name}` binds a string from the URL segment; `{a:int}` adds a *route
constraint* so `/add/abc/2` returns 404 instead of a bad model-binding error.
Route parameter names are matched to method parameter names automatically.

## Returning JSON, status codes, and typed results

```csharp
app.MapGet("/products/{id:int}", (int id) =>
{
    var product = ProductCatalog.Find(id);
    return product is not null
        ? Results.Ok(product)
        : Results.NotFound(new { message = $"Product {id} not found." });
});

app.MapPost("/products", (Product incoming) =>
{
    var created = ProductCatalog.Add(incoming);
    return Results.Created($"/products/{created.Id}", created);
});
```

`Results.Ok(...)`, `Results.NotFound(...)`, and `Results.Created(...)` all
implement `IResult` and set the appropriate HTTP status code alongside a JSON
body — minimal APIs serialize return values with `System.Text.Json`
automatically.

## Model binding from the request body

```csharp
record Product(int Id, string Name, decimal Price);

app.MapPost("/echo", (Product p) => Results.Ok(p));
```

A POST with body `{"id":1,"name":"Mug","price":9.99}` and header
`Content-Type: application/json` binds directly into a `Product` — no manual
`JsonSerializer.Deserialize` call needed for the common case.

## Query strings and `[FromQuery]`

```csharp
app.MapGet("/search", (string? term, int page = 1, int pageSize = 10) =>
{
    return Results.Ok(new { term, page, pageSize });
});
// GET /search?term=laptop&page=2  ->  { "term": "laptop", "page": 2, "pageSize": 10 }
```

Primitive-typed parameters that don't appear in the route are bound from the
query string by default; giving them a default value makes them optional.

## Grouping related endpoints

```csharp
var products = app.MapGroup("/products");

products.MapGet("/", () => ProductCatalog.All());
products.MapGet("/{id:int}", (int id) => ProductCatalog.Find(id) is { } p ? Results.Ok(p) : Results.NotFound());
products.MapPost("/", (Product p) => Results.Created($"/products/{p.Id}", ProductCatalog.Add(p)));
```

`MapGroup` prefixes every route registered on it with `/products` and lets
you apply shared configuration (auth, filters, tags) to the whole group at
once — Level 4 covers this alongside authentication.

## Middleware and the request pipeline

Every request flows through an ordered pipeline of middleware, each able to
inspect/modify the request, short-circuit, or call the next one:

```csharp
app.Use(async (context, next) =>
{
    var start = DateTime.UtcNow;
    await next();   // hands off to the rest of the pipeline
    var elapsed = DateTime.UtcNow - start;
    Console.WriteLine($"{context.Request.Method} {context.Request.Path} -> {context.Response.StatusCode} ({elapsed.TotalMilliseconds:F0}ms)");
});

app.MapGet("/slow", async () =>
{
    await Task.Delay(200);
    return "done";
});
```

Order matters: middleware registered earlier wraps everything registered
after it, like nested function calls. Module 08 (Middleware & Filters) goes
much deeper into writing your own.

## Configuration and environments

```csharp
var builder = WebApplication.CreateBuilder(args);

string connectionString = builder.Configuration.GetConnectionString("Default")
    ?? "Data Source=app.db";

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapGet("/debug/config", () => new { env = app.Environment.EnvironmentName });
}
```

`builder.Configuration` merges `appsettings.json`, `appsettings.{Environment}.json`,
environment variables, and command-line args, in that precedence order.
`app.Environment.IsDevelopment()` reads the `ASPNETCORE_ENVIRONMENT`
variable, letting you gate diagnostics or verbose errors to non-production
runs.

## Dependency injection, briefly

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();

app.MapGet("/time", (IClock clock) => new { now = clock.UtcNow });

interface IClock { DateTime UtcNow { get; } }
class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }
```

Minimal API handlers can request any registered service as a parameter — the
framework resolves it from the DI container per request. Module 03
(Dependency Injection Deep Dive) covers lifetimes and patterns in depth.

## Exercise

Build a minimal API with an in-memory `List<Product>` "database" (a static
list is fine for this exercise) exposing `GET /products`, `GET
/products/{id}`, `POST /products`, and `DELETE /products/{id}`. Add a logging
middleware like the one above, and a `GET /health` endpoint returning
`Results.Ok("healthy")`. Run it with `dotnet run` and exercise every endpoint
with `curl`.
