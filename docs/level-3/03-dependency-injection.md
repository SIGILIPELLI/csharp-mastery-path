---
description: "Dependency Injection Deep Dive — ASP.NET Core has a dependency injection (DI) container built in — IServiceProvider backed by IServiceCollection. Module…"
---

# 03 · Dependency Injection Deep Dive

ASP.NET Core has a dependency injection (DI) container built in — `IServiceProvider`
backed by `IServiceCollection`. Module 01 used it in passing; this module covers
service lifetimes, registration patterns, and the pitfalls that bite in real apps.

## The three lifetimes

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<ICounter, Counter>();   // one instance, whole app lifetime
builder.Services.AddScoped<IRequestId, RequestId>();  // one instance per HTTP request
builder.Services.AddTransient<IGreeter, Greeter>();   // new instance every time it's resolved

interface ICounter { int Next(); }
class Counter : ICounter
{
    private int _value;
    public int Next() => Interlocked.Increment(ref _value);
}

interface IRequestId { Guid Id { get; } }
class RequestId : IRequestId
{
    public Guid Id { get; } = Guid.NewGuid();
}

interface IGreeter { string Greet(); }
class Greeter : IGreeter
{
    private readonly Guid _instanceId = Guid.NewGuid();
    public string Greet() => $"Hello from instance {_instanceId}";
}
```

* **Singleton** — created once, shared by every consumer for the life of the app.
  Must be thread-safe if it holds mutable state (`Counter` uses `Interlocked`
  to stay safe under concurrent requests).
* **Scoped** — created once per scope. In ASP.NET Core, a scope is created per
  HTTP request, so every service resolved while handling one request that asks
  for `IRequestId` gets the *same* `RequestId` instance.
* **Transient** — a fresh instance every time it's requested, even within the
  same request if asked for twice.

```csharp
var app = builder.Build();

app.MapGet("/lifetimes", (ICounter counter, IRequestId reqA, IRequestId reqB, IGreeter a, IGreeter b) =>
{
    return Results.Ok(new
    {
        counter = counter.Next(),
        sameRequestId = reqA.Id == reqB.Id,   // true — same scope
        sameGreeter = ReferenceEquals(a, b),  // false — transient
    });
});

app.Run();
```

## Constructor injection is the default pattern

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IClock _clock;

    public OrderService(IOrderRepository repository, IClock clock)
    {
        _repository = repository;
        _clock = clock;
    }

    public Order Place(string customer, decimal total)
    {
        var order = new Order(Guid.NewGuid(), customer, total, _clock.UtcNow);
        _repository.Save(order);
        return order;
    }
}

public record Order(Guid Id, string Customer, decimal Total, DateTime PlacedAt);

public interface IOrderRepository { void Save(Order order); }
public interface IClock { DateTime UtcNow { get; } }
```

Classes depend on interfaces, never on concrete implementations. The container
supplies the implementation at resolution time — nothing in `OrderService`
mentions `SqlOrderRepository` or `SystemClock` directly, so swapping either
for a test double or a different backing store requires no change to
`OrderService` itself.

```csharp
builder.Services.AddScoped<IOrderRepository, InMemoryOrderRepository>();
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddScoped<OrderService>();

class InMemoryOrderRepository : IOrderRepository
{
    private readonly List<Order> _orders = new();
    public void Save(Order order) => _orders.Add(order);
}

class SystemClock : IClock { public DateTime UtcNow => DateTime.UtcNow; }
```

## Registering with a factory

Sometimes construction needs logic — reading configuration, choosing an
implementation conditionally:

```csharp
builder.Services.AddSingleton<IEmailSender>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var provider = config["Email:Provider"] ?? "console";
    return provider switch
    {
        "smtp" => new SmtpEmailSender(config["Email:Host"]!),
        _ => new ConsoleEmailSender(),
    };
});

interface IEmailSender { void Send(string to, string body); }
class ConsoleEmailSender : IEmailSender
{
    public void Send(string to, string body) => Console.WriteLine($"[email to {to}]: {body}");
}
class SmtpEmailSender : IEmailSender
{
    private readonly string _host;
    public SmtpEmailSender(string host) => _host = host;
    public void Send(string to, string body) => Console.WriteLine($"[smtp:{_host}] to {to}: {body}");
}
```

The factory lambda receives the `IServiceProvider`, so it can pull in other
registered services (`IConfiguration` here) to decide what to build.

## Options pattern for strongly-typed configuration

```csharp
public class SmtpOptions
{
    public string Host { get; set; } = "";
    public int Port { get; set; } = 25;
}

builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection("Smtp"));

app.MapGet("/smtp-config", (IOptions<SmtpOptions> options) =>
    Results.Ok(new { options.Value.Host, options.Value.Port }));
```

`IOptions<T>` binds a configuration section to a POCO once and shares it as a
singleton. Use `IOptionsSnapshot<T>` instead for scoped config that can be
reloaded per request (rarely needed outside long-running config-reload
scenarios), and `IOptionsMonitor<T>` for live-reload notifications.

## The captive dependency trap

```csharp
builder.Services.AddSingleton<ReportCache>();       // singleton
builder.Services.AddScoped<IOrderRepository, InMemoryOrderRepository>();  // scoped

class ReportCache
{
    // BUG: capturing a scoped service inside a singleton's constructor
    // "captures" it beyond its intended lifetime — the same repository
    // instance from whichever request first built ReportCache gets reused
    // forever, which is almost never what you want.
    public ReportCache(IOrderRepository repository) { }
}
```

The container throws `InvalidOperationException: Cannot consume scoped
service ... from singleton` at startup if validation is enabled (it is, by
default, in the Development environment) — that's the framework catching the
bug for you. The fix is to depend on `IServiceScopeFactory` and create a
scope on demand instead:

```csharp
class ReportCache
{
    private readonly IServiceScopeFactory _scopeFactory;
    public ReportCache(IServiceScopeFactory scopeFactory) => _scopeFactory = scopeFactory;

    public int CountOrders()
    {
        using var scope = _scopeFactory.CreateScope();
        var repository = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        return repository is InMemoryOrderRepository r ? r.Count : 0;
    }
}
```

## How It Actually Works

- **The container resolves a dependency graph by walking constructor
  parameters via reflection, once per resolution (with caching for the
  metadata, not the instances).** `GetRequiredService<OrderService>()`
  reflects over `OrderService`'s constructor, sees it needs `IOrderRepository`
  and `IClock`, recursively resolves each of *those* by consulting the
  registration table (a dictionary from service type to a descriptor holding
  the implementation type/factory and lifetime), and only then calls `new
  OrderService(repo, clock)` via a compiled expression tree the container
  builds and caches after the first resolution — subsequent resolutions of
  the same service type reuse that compiled factory rather than
  re-reflecting every time, the same "build once, reuse" pattern as
  `System.Text.Json`'s serialization plan.
- **Lifetimes are implemented as three different storage/lookup strategies
  inside the container, not a label.** Singleton instances are stored once
  in the root `IServiceProvider`'s own instance cache and handed out for
  every future request, from any scope. Scoped instances live in a
  dictionary owned by the *current* `IServiceScope` — ASP.NET Core creates
  exactly one such scope per incoming HTTP request (wired in during request
  processing, as Module 01 covered) and disposes it — running `Dispose()` on
  every `IDisposable` scoped instance it created — when the request
  finishes. Transient services are never cached anywhere; every resolution
  runs the constructor-injection factory fresh, which is exactly why `a` and
  `b` above are different objects even resolved in the same request.
- **The captive-dependency check is a real graph traversal run at
  `app.Build()` (in Development) or the first resolution, not a documentation
  warning.** ASP.NET Core's validation walks every registered singleton's
  constructor dependencies transitively and flags any path that reaches a
  scoped or transient-holding-scoped registration — this is a genuine static
  analysis over the DI registration graph, catching the bug before a single
  request runs, rather than relying on you noticing stale data in production.
  `IServiceScopeFactory.CreateScope()` sidesteps the rule legitimately by
  creating an independent, short-lived scope on demand — its own separate
  dictionary of scoped instances, disposed when the `using` block ends,
  exactly like the request-scoped one but created manually instead of by
  the framework.
- **`IOptions<T>` binding uses reflection once at startup to map
  configuration keys to POCO properties** (case-insensitively, walking
  nested sections for nested objects) and caches the bound instance as a
  singleton — `IOptionsSnapshot<T>`/`IOptionsMonitor<T>` differ only in
  *when* that binding re-runs (per-scope, or on a file-change/reload
  notification), not in the underlying reflection-based binding mechanism
  itself.

## Exercise

Build a console-hosted DI container (`Host.CreateApplicationBuilder(args)`)
registering an `INotifier` interface with two implementations
(`ConsoleNotifier`, `LoggingNotifier` that wraps another `INotifier` and
writes to `ILogger` before/after delegating). Register a service that depends
on `INotifier` as scoped, resolve it twice within one manually-created scope
(`IServiceScopeFactory.CreateScope()`), and confirm you get the same
instance both times using `ReferenceEquals`.
