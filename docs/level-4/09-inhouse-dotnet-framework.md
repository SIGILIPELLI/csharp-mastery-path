# 09 · Building an In-House .NET Framework

Larger organizations often build a thin internal framework on top of
ASP.NET Core — shared conventions for error handling, result types, and
cross-cutting concerns so every team's service looks and behaves
consistently. This module builds a small one, in miniature, to show the
underlying techniques: source-generator-free extension methods, a shared
`Result<T>` type, and a reusable NuGet package.

## A shared `Result<T>` type instead of exceptions for expected failures

```csharp
public readonly struct Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool isSuccess, T? value, string? error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}
```

```csharp
public Result<Order> PlaceOrder(string customer, decimal total)
{
    if (string.IsNullOrWhiteSpace(customer))
        return Result<Order>.Failure("Customer name is required.");
    if (total <= 0)
        return Result<Order>.Failure("Total must be positive.");

    return Result<Order>.Success(new Order(Guid.NewGuid(), customer, total));
}

// Calling code:
var result = PlaceOrder("Alice", 49.99m);
IResult response = result.Match(
    onSuccess: order => Results.Created($"/orders/{order.Id}", order),
    onFailure: error => Results.BadRequest(new { error }));
```

Using `Result<T>` for *expected* failures (bad input, business rule
violations) keeps exceptions reserved for genuinely exceptional conditions
(a database connection dropping) — exceptions are relatively expensive and
easy to accidentally swallow with an overly broad `catch`, whereas a
`Result<T>` forces the caller to handle both branches via `Match`.

## A shared problem-details error format

```csharp
public static class ProblemResults
{
    public static IResult ValidationProblem(string detail, string? instance = null) =>
        Results.Problem(
            title: "Validation failed",
            detail: detail,
            statusCode: StatusCodes.Status400BadRequest,
            instance: instance);

    public static IResult NotFoundProblem(string resource, object id) =>
        Results.Problem(
            title: $"{resource} not found",
            detail: $"No {resource} exists with id '{id}'.",
            statusCode: StatusCodes.Status404NotFound);
}
```

Every team's API returning errors in the *same* shape (RFC 7807 "problem
details") means every consuming client can write one error-handling code
path instead of one per service — this is exactly the kind of small,
boring consistency an in-house framework exists to enforce.

## A reusable middleware extension package

```csharp
// MyCompany.AspNetCore.Shared/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCompanyDefaults(this IServiceCollection services, IConfiguration config)
    {
        services.AddProblemDetails();
        services.AddHealthChecks().AddCheck("self", () => HealthCheckResult.Healthy());
        services.AddCors(options => options.AddDefaultPolicy(policy =>
            policy.WithOrigins(config.GetSection("AllowedOrigins").Get<string[]>() ?? Array.Empty<string>())
                  .AllowAnyMethod().AllowAnyHeader()));
        return services;
    }

    public static WebApplication UseCompanyDefaults(this WebApplication app)
    {
        app.UseExceptionHandler();
        app.UseCors();
        app.MapHealthChecks("/health/live");
        return app;
    }
}
```

```csharp
// In each team's Program.cs — one line pulls in every shared convention:
builder.Services.AddCompanyDefaults(builder.Configuration);
var app = builder.Build();
app.UseCompanyDefaults();
```

This is the actual shape most "in-house frameworks" take in practice: not a
from-scratch reimplementation of ASP.NET Core, but a thin, versioned NuGet
package of extension methods wrapping already-battle-tested framework
pieces with the org's chosen defaults — new services opt in with one or two
lines instead of every team reinventing CORS policy and health checks
slightly differently.

## Packaging it as a NuGet package

```xml
<!-- MyCompany.AspNetCore.Shared.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <PackageId>MyCompany.AspNetCore.Shared</PackageId>
    <Version>1.2.0</Version>
    <Authors>Platform Team</Authors>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>
</Project>
```

```bash
dotnet pack -c Release
dotnet nuget push bin/Release/MyCompany.AspNetCore.Shared.1.2.0.nupkg --source https://nuget.mycompany.internal/v3/index.json
```

Publishing to an internal NuGet feed (Azure Artifacts, GitHub Packages, or a
self-hosted feed) lets every service reference
`MyCompany.AspNetCore.Shared` by version the same way they reference any
public package — semantic versioning (module 09, Level 2) becomes the
contract for "is this a safe upgrade" across every consuming team.

## Guarding against overreach

The failure mode of in-house frameworks is scope creep: a "shared defaults"
package that grows into a mandatory, leaky abstraction over ASP.NET Core
itself, coupling every team to the platform team's release cadence for
things that didn't need sharing. A good rule: only centralize what's
genuinely identical across teams (health check conventions, error shape,
required security headers) — leave anything domain-specific
(business logic, entity models, team-specific validation) entirely out of
the shared package.

## How It Actually Works

- **`Result<T>` being a `readonly struct` rather than a `class` is a
  deliberate cost decision, not stylistic — it avoids a heap allocation on
  every method call that would otherwise return one.** Per Module 5 of
  Level 1's stack-vs-heap split, a struct returned by value is copied
  inline into the caller's stack frame or, for an `async Task<Result<T>>`,
  inline into the compiler-generated state machine's fields (Module 04 of
  Level 2) — no GC-tracked object is created at all for the success/failure
  wrapper itself, only for whatever `T` genuinely needs to live on the heap.
  Contrast this with throwing an `Exception` for the same "expected
  failure" case: Module 07 of Level 1 covered the real cost of exception
  throwing (stack-trace capture, heap allocation, two-pass unwinding) — a
  `Result<T>` struct sidesteps all of that for outcomes that aren't
  actually exceptional.
- **`Match`'s two `Func<...>` parameters each cost a delegate allocation per
  call site unless the compiler can prove they're non-capturing** — per
  Module 03 of Level 2, `onSuccess: order => Results.Created(...)` closes
  over nothing external here, so the compiler can (and typically does)
  cache a single static delegate instance across calls; a lambda that
  captured a local variable instead would allocate a closure object on
  every `Match` invocation, a subtle cost worth knowing about if `Match` is
  called in a genuinely hot path.
- **`AddCompanyDefaults`/`UseCompanyDefaults` are ordinary extension
  methods — static methods with no different calling mechanism than any
  other C# method — resolved entirely at compile time by Roslyn's extension-
  method lookup, not a runtime plugin or reflection-based discovery
  mechanism.** `services.AddCompanyDefaults(config)` compiles to a plain
  static method call `ServiceCollectionExtensions.AddCompanyDefaults(services,
  config)`; this is precisely why "one line pulls in every shared
  convention" works with zero runtime indirection — it's the same
  extension-method mechanism behind `.Where()`/`.Select()` from Module 08
  of Level 1, just registering DI services and configuring middleware
  instead of filtering a sequence.
- **`dotnet pack` builds a `.nupkg` — a renamed `.zip` archive — containing
  the compiled assembly, a `.nuspec` manifest describing the package
  metadata/dependencies, and (if configured) source/symbol files**, exactly
  the same artifact format any public NuGet package uses; publishing to an
  internal feed versus nuget.org is purely a difference in the target URL
  `dotnet nuget push` sends the package to — the restore/resolution
  mechanism consuming teams rely on (Module 09 of Level 2's
  `project.assets.json`/`.deps.json` machinery) is identical either way.

## Exercise

Extract a `Result<T>` type and the `ProblemResults` helpers above into a
separate class library project, reference it from the Level 3 REST API
project, and refactor the `POST /books` and `PUT /books/{id}` endpoints to
return `Result<Book>` from the repository layer, translated to
`IResult`/`ProblemDetails` at the endpoint boundary via `Match`. Pack the
library with `dotnet pack` and confirm the `.nupkg` is produced with the
expected version number.
