---
description: "Middleware & Filters in ASP.NET Core — Module 01 introduced app.Use(...) in passing. This module covers writing proper reusable middleware classes, the…"
---

# 08 · Middleware & Filters in ASP.NET Core

Module 01 introduced `app.Use(...)` in passing. This module covers writing
proper reusable middleware classes, the built-in middleware you'll use in
almost every app, and MVC/minimal-API filters for cross-cutting logic that
needs to run closer to a specific endpoint.

## The middleware pipeline, precisely

```csharp
var app = builder.Build();

app.Use(async (context, next) =>
{
    Console.WriteLine("A: before");
    await next();
    Console.WriteLine("A: after");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("B: before");
    await next();
    Console.WriteLine("B: after");
});

app.MapGet("/", () => "hello");
app.Run();

// Request to "/" prints:
// A: before
// B: before
// A: after
// B: after
```

Each middleware wraps the next like nested function calls — registration
order determines nesting order. Calling `await next()` hands control
downstream; code after it runs on the way back up, which is why logging,
timing, and exception-wrapping middleware all follow the same
before/`next()`/after shape.

## Writing a middleware as a class

Lambdas are fine for small snippets; a class-based middleware is easier to
unit test and configure via DI:

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        await _next(context);
        sw.Stop();
        _logger.LogInformation("{Method} {Path} -> {StatusCode} in {Elapsed}ms",
            context.Request.Method, context.Request.Path, context.Response.StatusCode, sw.ElapsedMilliseconds);
    }
}

public static class RequestTimingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder app) =>
        app.UseMiddleware<RequestTimingMiddleware>();
}
```

```csharp
app.UseRequestTiming();
```

The framework constructs one instance of `RequestTimingMiddleware` per
pipeline (not per request) via DI, injecting `RequestDelegate next` (the rest
of the pipeline) and any other registered services — here `ILogger<T>`. Only
`InvokeAsync` runs per-request. A `Use...()` extension method is the
idiomatic way to expose a middleware, mirroring the built-in
`UseHttpsRedirection()`, `UseCors()`, etc.

## Short-circuiting

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    public ApiKeyMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-Api-Key", out var key) || key != "secret123")
        {
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            await context.Response.WriteAsync("Missing or invalid API key.");
            return;   // does NOT call _next — pipeline stops here
        }

        await _next(context);
    }
}
```

Not calling `_next(context)` stops the request from reaching anything
registered afterward — the standard way to implement auth gates, rate
limiting, or maintenance-mode pages.

## Built-in middleware, in a typical order

```csharp
var app = builder.Build();

app.UseExceptionHandler("/error");   // catches unhandled exceptions, first
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();              // after authentication, before endpoints

app.MapControllers();
app.Run();
```

Order matters for correctness, not just style: `UseAuthorization()` needs
`UseRouting()` to have already matched a route (so it knows which endpoint's
`[Authorize]` metadata to check) and needs `UseAuthentication()` to have
already populated `context.User`.

## Endpoint filters (minimal APIs)

Filters run per-endpoint rather than per-request, and can inspect/short-circuit
individual route handler invocations:

```csharp
app.MapGet("/products/{id:int}", (int id) => Results.Ok(new { id }))
   .AddEndpointFilter(async (context, next) =>
   {
       var id = context.GetArgument<int>(0);
       if (id <= 0)
           return Results.BadRequest("id must be positive.");

       return await next(context);
   });
```

`AddEndpointFilter` wraps just this one route's handler — good for
validation or logging scoped to a single endpoint rather than the whole app.

## MVC action filters

Controller-based APIs use `IActionFilter`/`IAsyncActionFilter` for the same
idea, attributable per-action, per-controller, or globally:

```csharp
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }
}

[ApiController]
[Route("orders")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    [ValidateModel]
    public IActionResult Create(CreateOrderRequest request) => Ok(request);
}

public record CreateOrderRequest([Required] string Customer, [Range(0.01, 100000)] decimal Total);
```

`OnActionExecuting` runs before the action method; setting `context.Result`
short-circuits the action itself, similar to not calling `next()` in
middleware. `[ApiController]` actually applies automatic model-state
validation like this already — the example illustrates the mechanism, which
is the same one you'd use for a custom cross-cutting concern (auditing,
caching action results, enforcing a custom header).

## Exception-handling middleware, hand-written

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception processing {Path}", context.Request.Path);
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(new { error = "An unexpected error occurred." });
        }
    }
}
```

Registering this first in the pipeline (`app.UseMiddleware<ExceptionHandlingMiddleware>()`
before everything else) means it wraps every downstream middleware and
endpoint, catching anything they throw — this is essentially what
`UseExceptionHandler` provides out of the box, worth building once by hand
to understand it.

## How It Actually Works

- **`UseMiddleware<T>()` constructs exactly one instance of your middleware
  class per application startup — not per request — using a compiled
  factory built via `ActivatorUtilities`, which is why per-request state
  must never live in instance fields.** At startup, the framework reflects
  over `RequestTimingMiddleware`'s constructor, resolves `RequestDelegate
  next` (the already-built rest-of-pipeline delegate at that registration
  point) and any DI services like `ILogger<T>` exactly once, and caches the
  constructed instance for the app's entire lifetime. Only `InvokeAsync` —
  never the constructor — runs per incoming request; this singleton-like
  construction is why the class-based middleware pattern explicitly warns
  against storing request-specific data in fields (it would leak across
  concurrent requests) and instead threads everything through the
  per-request `HttpContext` parameter.
- **The nested-delegate chain built at startup is precisely the same
  mechanism Module 01 described for `app.Use(...)` lambdas — class-based
  middleware just wraps `InvokeAsync` in a delegate that closes over the
  constructed instance.** `app.UseRequestTiming()` calls `UseMiddleware<T>`,
  which builds a `RequestDelegate` (a `Func<HttpContext, Task>`) that calls
  `instance.InvokeAsync(context)`, and stitches it into the nested-delegate
  pipeline exactly like a raw lambda would — there's no architecturally
  distinct "class middleware system," just a convenience wrapper generating
  the same shape.
- **Returning without calling `_next(context)` in `ApiKeyMiddleware` doesn't
  throw or signal cancellation — it simply lets that middleware's
  `InvokeAsync`/lambda complete, which unwinds back up through every
  middleware registered before it (running their post-`next()` code) without
  ever entering anything registered after it.** This is exactly the "nested
  function call" model made concrete: skipping `next()` is equivalent to a
  nested call simply never happening, so nothing inside that unexecuted call
  (routing, endpoint execution, later middleware) ever runs, while
  everything wrapping the current middleware still unwinds normally.
- **Endpoint filters (`AddEndpointFilter`) attach to the specific
  `RequestDelegate` the routing middleware ultimately invokes for that one
  route, not the global pipeline** — they're resolved and composed once per
  endpoint at startup (another nested-delegate chain, scoped narrowly), which
  is why a filter added to one `MapGet` call never runs for a different
  route, in contrast to `app.Use(...)` which always wraps every request
  regardless of which endpoint eventually handles it.
- **MVC action filters (`OnActionExecuting`) run inside the MVC action
  invocation pipeline, which itself executes as one step deep inside the
  outer middleware pipeline** — setting `context.Result` short-circuits only
  that inner action-invocation pipeline (skipping the actual controller
  method and remaining action filters) while the outer middleware chain
  still unwinds normally afterward, the same nested-scope relationship
  `AddEndpointFilter` has to the top-level pipeline in minimal APIs.

## Exercise

Write a `RateLimitingMiddleware` that tracks request counts per client IP
(a `ConcurrentDictionary<string, (int count, DateTime windowStart)>` is
enough) and returns `429 Too Many Requests` once a client exceeds 5 requests
in a rolling 10-second window, otherwise calls `next()`. Register it early
in the pipeline, and write a minimal integration test (module 06) that fires
6 rapid requests and asserts the 6th gets a 429.
