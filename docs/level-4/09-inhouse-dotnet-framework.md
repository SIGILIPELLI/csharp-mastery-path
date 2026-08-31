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

## Exercise

Extract a `Result<T>` type and the `ProblemResults` helpers above into a
separate class library project, reference it from the Level 3 REST API
project, and refactor the `POST /books` and `PUT /books/{id}` endpoints to
return `Result<Book>` from the repository layer, translated to
`IResult`/`ProblemDetails` at the endpoint boundary via `Match`. Pack the
library with `dotnet pack` and confirm the `.nupkg` is produced with the
expected version number.
