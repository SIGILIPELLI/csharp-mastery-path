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

## Exercise

Build a console-hosted DI container (`Host.CreateApplicationBuilder(args)`)
registering an `INotifier` interface with two implementations
(`ConsoleNotifier`, `LoggingNotifier` that wraps another `INotifier` and
writes to `ILogger` before/after delegating). Register a service that depends
on `INotifier` as scoped, resolve it twice within one manually-created scope
(`IServiceScopeFactory.CreateScope()`), and confirm you get the same
instance both times using `ReferenceEquals`.
